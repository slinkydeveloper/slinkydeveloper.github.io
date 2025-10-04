---
title: Vert.x container images with jlink
description: In this blog post I'm gonna show you how I managed to reduce the container image size of an Eclipse Vert.x application, creating a smaller JDK with jdeps and jlink.
date: 2020-10-08
tags: [development, container, cloud, vertx, java, jlink, jdeps]
---

In this blog post I'm gonna show you how I managed to reduce the container image size of an Eclipse Vert.x application, creating a smaller JDK with `jdeps` and `jlink`.

The application is the data plane of [Knative Eventing Kafka Broker](https://github.com/knative-sandbox/eventing-kafka-broker), an implementation of Knative `Broker` tailored on Kafka.

## Fat-jar

We didn't handwritten nor generated a `module-info.java` for our application, we just ship a fat-jar. To generate a fat-jar, configure the `maven-shade-plugin`:

```xml
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-shade-plugin</artifactId>
   <version>${maven.shade.plugin.version}</version>
   <configuration>
     <minimizeJar>true</minimizeJar>
   </configuration>
   <executions>
     <execution>
       <phase>package</phase>
       <goals>
         <goal>shade</goal>
       </goals>
       <configuration>
         <transformers>
           <transformer
             implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
             <mainClass>
               ${main-class}
             </mainClass>
           </transformer>
         </transformers>
       </configuration>
     </execution>
   </executions>
</plugin>
```

## Jdeps

`jdeps` is the tool you must use to figure which JDK modules you depend on. If you run `jdeps` just with the fat-jar as argument, you'll get a list of all the packages in your jar and the JDK modules it depends on:

```shell script
% jdeps receiver/target/receiver-1.0-SNAPSHOT.jar
[...]
   dev.knative.eventing.kafka.broker.core             -> java.lang                                          java.base
   dev.knative.eventing.kafka.broker.core             -> java.lang.invoke                                   java.base
   dev.knative.eventing.kafka.broker.core             -> java.net                                           java.base
   dev.knative.eventing.kafka.broker.core             -> java.time                                          java.base
   dev.knative.eventing.kafka.broker.core             -> java.time.format                                   java.base
   dev.knative.eventing.kafka.broker.core             -> java.util                                          java.base
   dev.knative.eventing.kafka.broker.core             -> java.util.concurrent                               java.base
   dev.knative.eventing.kafka.broker.core             -> java.util.function                                 java.base
   dev.knative.eventing.kafka.broker.core             -> java.util.stream                                   java.base
[...]
```

Note that `jdeps` analyzes only the imports so, if you perform some reflections at runtime of JDK classes, they won't be found by the tool.

To get an output that you can directly pass to `jlink`:

```shell script
% jdeps -q --print-module-deps --ignore-missing-deps receiver/target/receiver-1.0-SNAPSHOT.jar
java.base,java.compiler,java.naming,java.security.jgss,java.security.sasl,java.sql,jdk.management,jdk.unsupported
```

This is the list of jdk modules we depend on. Doing a quick check, I've found that:

* `java.base` it's the module that contains all the core features of the jdk
* `java.compiler` contains the compiler types. It's brought in by Guava and Vert.x CLI feature.
* `java.naming` contains some JNDI types, to perform names lookups. This is required to perform DNS queries by Vert.x
* `java.security.jgss` and `java.security.sasl` contains some security protocols implementation. They're used by the Java Kafka client.
* `java.sql` contains JDBC. Jackson databind and Google Gson (that we import transitively via Protobuf json) depends on it because they provide marshallers/unmarshallers for JDBC types.
* `jdk.management` contains some interfaces to manage the JDK. This is used by Micrometer to instrument the JVM and collect metrics.
* `jdk.unsupported` contains `sun.misc.Unsafe`. Netty and Protobuf use it to perform off-heap allocations.

## Zero days since it was DNS

Because some reflection is happening behind the hood to choose the Vert.x DNS resolver, `jdeps` doesn't discover the module `jdk.naming.dns`, that you need in your Vert.x application to enable the DNS.

To resolve the deps and add the dns module:

```shell script
MODS=$(jdeps -q --print-module-deps --ignore-missing-deps receiver/target/receiver-1.0-SNAPSHOT.jar)
echo "Computed mods = '$MODS'"
# Patch adding the dns
MODS="$MODS,jdk.naming.dns"
```

## Create the JDK

Now you just need to invoke `jlink` to generate your custom JDK:

```shell script
% jlink --verbose \
  --no-header-files \
  --no-man-pages \
  --compress=2 \
  --strip-debug \
  --add-modules "$MODS" \
  --output ./jdk
java.base file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.base.jmod
java.compiler file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.compiler.jmod
java.logging file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.logging.jmod
java.management file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.management.jmod
java.naming file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.naming.jmod
java.security.jgss file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.security.jgss.jmod
java.security.sasl file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.security.sasl.jmod
java.sql file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.sql.jmod
java.transaction.xa file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.transaction.xa.jmod
java.xml file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/java.xml.jmod
jdk.management file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/jdk.management.jmod
jdk.naming.dns file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/jdk.naming.dns.jmod
jdk.unsupported file:///usr/lib/jvm/java-15-openjdk-15.0.0.36-1.rolling.fc32.x86_64/jmods/jdk.unsupported.jmod

Providers:
  java.base provides java.nio.file.spi.FileSystemProvider used by java.base
  java.naming provides java.security.Provider used by java.base
  java.security.jgss provides java.security.Provider used by java.base
  java.security.sasl provides java.security.Provider used by java.base
  jdk.naming.dns provides javax.naming.spi.InitialContextFactory used by java.naming
  java.management provides javax.security.auth.spi.LoginModule used by java.base
  java.logging provides jdk.internal.logger.DefaultLoggerFinder used by java.base
  jdk.management provides sun.management.spi.PlatformMBeanProvider used by java.management
```

This command will grab the modules you provided from your local machine and will create a JDK without manual, header files, debug symbols:

```shell script
jdk
├── bin
│   ├── java
│   └── keytool
├── conf
│   ├── logging.properties
│   ├── net.properties
│   ├── sdp
│   └── security
[...]
├── lib
[...]
└── release
```

## Complete script and Dockerfile

This is the complete Dockerfile (simplified) that builds the project and generates the JDK:

```dockerfile
ARG JAVA_IMAGE=docker.io/adoptopenjdk:14-jdk-hotspot
ARG BASE_IMAGE=docker.io/ubuntu:bionic

FROM ${JAVA_IMAGE} as builder

WORKDIR /app

# https://github.com/AdoptOpenJDK/openjdk-docker/issues/260
RUN apt-get update && apt-get install -y binutils

# Copy all my project stuff inside the container
COPY . .

# Build the project
RUN ./mvnw package -DskipTests -Deditorconfig.skip --no-transfer-progress

# Generate the sdk using the script (shown below)
RUN ./generate_jdk.sh /app/receiver/target/receiver-1.0-SNAPSHOT.jar

# Prod image
FROM ${BASE_IMAGE} as running

# Create appuser and directories
RUN groupadd -g 999 appuser && useradd -r -u 999 -g appuser appuser && \
  mkdir /tmp/vertx-cache && chown -R appuser:appuser /tmp/vertx-cache && \
  mkdir /app

# Copy jar and jdk inside /app
COPY --from=builder /app/jdk /app/jdk
COPY --from=builder /app/receiver/target/receiver-1.0-SNAPSHOT.jar /app/app.jar
RUN chown -R appuser:appuser /app

# Set appuser and configure PATH
USER appuser
WORKDIR /app
# Add jdk bin to the path, so you can run the java command
ENV PATH="/app/jdk/bin:${PATH}"
```

And the `generate_jdk.sh` script:

```shell script
#!/usr/bin/env sh

if [ $# -eq 0 ]
  then
    echo "No arguments supplied, You must provide the jar"
fi

echo "Computing mods for $1"
MODS=$(jdeps -q --print-module-deps --ignore-missing-deps "$1" | sed -e 's/^[ \t]*//')
echo "Computed mods = '$MODS'"

# Patch adding the dns
MODS="$MODS,jdk.naming.dns"
echo "Patched modules shipped with the generated jdk = '$MODS'"

jlink --verbose --no-header-files --no-man-pages --compress=2 --strip-debug --add-modules "$MODS" --output /app/jdk
```

You can run the built container with:

```shell script
% docker run imagename java -jar /app/app.jar
```

## Conclusions

Using `jdeps` and `jlink` our container image size is 2.5x smaller, which is a great achievement for us. We also plan to reduce it even further, in fact we want to:

* Investigate some modules we depend on, but that we probably don't need: `java.compiler` and `java.sql` are two good candidates.
* Rebase our base image on `alpine`, which uses `musl` as libc. Look at [Project Portola](https://openjdk.java.net/projects/portola/) for more details.

Check out the full PR: https://github.com/knative-sandbox/eventing-kafka-broker/pull/265
