---
title: Reproducibility of benchmarks in Kubernetes
description: "Today I'm going to talk you about how I've managed to run a benchmark inside Kubernetes in a reproducible manner"
date: 2019-12-02 15:49:53
tags: cloud, kubernetes, serverless, knative
---

Hi everybody! Today I'm going to talk you about how I've managed to run a benchmark **inside** Kubernetes in a **reproducible** manner.

## Reproducibility: What and Why

**Reproducibility** means that different runs of the same benchmark, testing the same system, running in the same environment, should lead to similar results.

This is one of the most important traits that every benchmark should respect, because without it, the test can't be trusted.

For example, let's assume your boss gave you to optimize the most important system in production. You begin writing a benchmark to understand how it performs and, without caring about reproducibility, you jump to start searching where the performance hotspots are and operate to solve them. Now you run again the tests and results are better than the beginning but, while you're already feeling the bonus on the next paycheck, a colleague comments your "40% performance boost" PR saying "I tried to run the benchmark and results look worse than the beginning!". What the heck? Did your PR improves the performance or not?

You can't really answer that question, because your benchmark is not reproducible! You can try to run it several times but you'll continue to get deniable and not correlated results, that can answer positively or negatively to your question.

Making the test reproducible, for a good part, depends on the environment where you run the test. 

Kubernetes is a virtualized environment designed to scale up & down workloads, depending on resource demands. So it can arbitrarily schedule your application to run where it wants, imposing precise cpu/memory resources. I'll show you   what countermeasures I've took in my methodology to run benchmarks **inside** K8s to prevent such problems.

## System under test: Knative Eventing

A couple of months I've started working on a project called Knative Eventing, an event mesh for Kubernetes. One of the goals of Knative Eventing is to enable message consuming & producing through HTTP, acting as a bridge between a "traditional" messaging system (such as Kafka) and an HTTP application.

I won't cover all aspects of Knative Eventing, If you want to learn more about it check out the [Knative Eventing documentation](https://knative.dev/docs/eventing/)

Knative, among the others, provides the concept of `Channel`, a flow of events from one or more producers to one or more subscribed consumers:

{%fi /images/reproducibility-k8s/channel.png, Knative Channel, From source to user application, through a Channel %}

To push events into the channel you interact with its HTTP interface, while to receive events from the channel you subscribe to it, specifying at what HTTP endpoint the channel should send the events. Behind the hood, a pod called _dispatcher_ is actually serving the HTTP interface for inbound events, managing the interaction with the messaging system and dispatching the events to the subscribers.

In this post I will use the test that calculates `KafkaChannel`'s throughput and latency.

## Test & Cluster

The test components are:

* a sender container that forces a configurable load for a certain amount of time on the Channel
* a receiver container that subscribes to the channel
* an aggregator container that fetches results from sender and receiver containers and calculates latencies and throughputs

All these three components runs inside the cluster.

I won't spend words in this post to explain how such components, designed together with Knative community, work in deep. If you want to know more about it, look at the [`README`](https://github.com/knative/eventing/tree/master/test/test_images/performance).

The cluster where I'm running the tests is composed by three bare metal machines in the same rack and they're running **only these tests**. It's quite important to run on bare metal, otherwise you will need to make further steps to make your virtualization environment reproducible, depending on the VM system you use. 

## Step 0: How to determine reproducibility?

The question that arises is: what metric should be used to determine reproducibility? A wise answer could be that the standard deviation of the metric used to determine a performance improvement should be used to determine reproducibility. 

In my case I'm going to use standard deviation of the **percentiles** of E2E latency (from sender to receiver) across several runs. The lower is the standard deviation, more reproducible is the test.

To improve reproducibility, I'll start by configuring and running the test 5 times, to calculate a baseline standard deviation. Then I'll show you the tweaks I've made to reduce the standard deviation to an acceptable value:

1. Configure the test to don't blow up the system
1. Pin containers to nodes
1. Restart the system after each run
1. Configure the resource limits

## Step 1: Configure the test to don't blow up the system

The first step is to configure the test to correctly generate a load that doesn't blow up the system. System must be stressed, but in such a way that doesn't lead to a complete degradation, or even a crash.

I've configured my test to force 500 requests per second for 30 seconds, which I've found, experimentally, that is a good configuration the system can hold. Bear in mind that different "requests per second" configurations leads to different latencies!

I've collected the 99%, 99.9% and 99.99% percentiles but I'll focus on 99% percentile because I've managed to do only few and short test runs, and in such situations outliers are more visible and not filtered out in higher percentiles. In a "production run" of the test, you should run it for more than 30 seconds, to understand if higher latencies happens frequently.

After a first run, just configuring the test and running it for 5 times, I've these results:

|       | P99        | P99.9      | P99.99     |
|-------|------------|------------|------------|
| Run 1 | 266.266179 | 276.945500 | 284.709000 |
| Run 2 | 264.750750 | 278.127000 | 283.149000 |
| Run 3 | 250.629000 | 263.994500 | 271.937000 |
| Run 4 | 250.594875 | 261.605000 | 272.635000 |
| Run 5 | 266.224393 | 282.690500 | 290.529000 |


The SD of P99 is 8.312 and, in particular, the relative standard deviation is 3.2%.

From experimental evidence I've found that the relative standard deviation is not linearly related with the test configuration, which means that the more stress is applied by the load generator, the more could be the **relative** standard deviation.

Let's try to dig into why these numbers are so different and how I've lowered them.

## Step 2: Pin containers to nodes

The first thing you can notice is that the third an fourth run performed with generally lower numbers than the others. Digging a bit with `kubectl describe nodes` I've found that Kubernetes was scheduling on each run pods in different nodes. Sometimes it scheduled the sender and receiver in the same node of Kafka Channel dispatcher, letting them communicate with lower latencies!

To let Kubernetes deploy the pods always in the same nodes, I've configured the affinity of sender, receiver and all SUTs (system under test, which in my case means the Kafka Channel dispatcher and the Kafka cluster).
To do it, I've defined three labels:

- `bench-role: kafka`: Where Kafka cluster and Zookeeper are deployed
- `bench-role: eventing`: Where the kafka dispatcher is deployed
- `bench-role: sender`: Where both sender and receiver are deployed

And then, I set these labels in my cluster using:

```bash
kubectl label nodes node_name bench-role=eventing
```

On every deployment/pod descriptor, I've configured affinity in my various deployment descriptors.

I deployed Kafka using [Strimzi](https://strimzi.io/) and, thanks to its `Kafka` CRD, I can easily configure the [affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity) too (I've omitted irrelevant parts of this config):

```yaml
kafka:
  template:
    pod:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: bench-role
                    operator: In
                    values:
                      - kafka
zookeeper:
  template:
    pod:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: bench-role
                    operator: In
                    values:
                      - kafka
```

For `kafka-ch-dispatcher`, I just modified the original [dispatcher yaml](https://github.com/knative/eventing-contrib/blob/master/kafka/channel/config/500-dispatcher.yaml) adding the [`nodeSelector`](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector) (which is in fact a short version of the `nodeAffinity`) and I redeployed from source using ko:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ch-dispatcher
  namespace: knative-eventing
spec:
  template:
    spec:
      nodeSelector:
        bench-role: eventing
      containers:
        - name: dispatcher
```

Now the `sender` and `receiver`:

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: channel-perf-send
  namespace: perf-eventing
spec:
  nodeSelector:
    bench-role: sender
  containers:
    - name: sender

---

apiVersion: v1
kind: Pod
metadata:
  name: channel-perf-receive
  namespace: perf-eventing
spec:
  nodeSelector:
    bench-role: sender
  containers:
    - name: receiver
```

## Step 3: Restart the system after each run

After pinning the workload to different nodes, I've ran again the tests: 

|       | P99        | P99.9      | P99.99     |
|-------|------------|------------|------------|
| Run 1 | 263.552250 | 268.646500 | 272.223000 |
| Run 2 | 266.060133 | 280.979000 | 285.983000 |
| Run 3 | 266.994500 | 282.858000 | 292.864000 |
| Run 4 | 268.234000 | 297.516000 | 326.862000 |
| Run 5 | 265.809929 | 281.717000 | 288.665000 |

As you may notice, the first four runs looks incrementally worse. This happens because every run depends on the SUTs states caused by the previous run. The Kafka cluster and/or the Kafka channel dispatcher could be in a degradated state before a new run begins and this obviously reduces the chances to have same results over multiple runs. All systems involved in the road from sender to receiver must be reset, so every run starts stressing the system under the same conditions, ensuring that the latency of a run doesn't depend on previous runs.

In my case just deleting all pods does the trick, since the `Deployment`s spin up a new ones:

```bash
kubectl delete pods -n knative-eventing --all
kubectl delete kt --all-namespaces --all # Delete all KafkaTopics
kubectl delete pods -n kafka --all

kubectl wait pod -n knative-eventing --for=condition=Ready --all
kubectl wait pod -n kafka --for=condition=Ready --all
```



## Step 4: Configure the resource limits

As explained at beginning of this post, Kubernetes is designed to scale up & down workloads. What if the scheduler decides to schedule up and down our benchmark resources while the test is running? The benchmark needs to have granted the resources it needs & these should not change while is running. To do so, resource `request` & `limits` must be configured the same for every test and SUT, like: 

```yaml
resources:
  requests:
    cpu: 16
    memory: "8Gi"
  limits:
    cpu: 16
    memory: "8Gi"
```

This leads Kubernetes to schedule pods with QoS class [`Guaranteed`](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed), so it can't scale up & down resources.

The nodes where I'm running the benchmarks are configured with AMD EPYC 7401P 24 cores CPUs (so 48 logical cores) and 24Gb of RAM.
I've tried to match these limits as following: 

* Kafka has 16 cpus and 8Gi of memory, same for Zookeeper
* Kafka channel dispatcher has 44 cpus and 22Gi of memory
* Sender has 16 cpus and 8Gi of memory, same for receiver

The problem is, even if containers are configured with `Guaranteed` QoS, there are no guarantees that the workload is pinned and it has exclusive access to the cores. By default, even in `Guaranteed` QoS, Kubernetes can move the workload to different cores depending on whether the pod is throttled and which CPU cores are available at scheduling time. The Kube scheduler does it defining the [CFS Quota](https://www.kernel.org/doc/Documentation/scheduler/sched-bwc.txt) for the running container, so it asks to the kernel scheduler to allocate a fixed time to such containers. 

Luckily there is a way to force the CPU pinning, enabling the [static CPU management](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy). This can be done only configuring the [Kubelet config file](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/) for each node. To do so:

1. If the node is already running and connected to the cluster, it must be [drained](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain) using `kubectl drain <node name> --delete-local-data --force --ignore-daemonsets`
1. Kubelet config file should contain the following entry: `cpuManagerPolicy: static`
1. Kubernetes system containers should have statically assigned resources. To do it, Kubelet config file should contain a configuration like:

```yaml
systemReserved:
  cpu: "1"
  memory: "1Gi"
```

## Results

I've tried to ran the tests after all these tweaks:

|       | P99         | P99.9      | P99.99     |
|-------|-------------|------------|------------|
| Run 1 | 265.955238  | 271.344500 | 276.415000 |
| Run 2 | 264.850000  | 271.462000 | 279.283000 |
| Run 3 | 266.283643  | 291.772500 | 335.116000 |
| Run 4 | 266.065179  | 272.497000 | 279.553000 |
| Run 5 | 264.828300  | 271.254500 | 278.362000 |

This results looks far better! Now relative SD of P99 is down to 0.26% (0.7014114246) vs the initial 3.2%!

I still have some outliers at higher percentiles, but now the results looks more trusty than the previous 3.2% of relative SD.

To wrap up, I want to underline that these tweaks worked for me but they could not be enough for all benchmark configurations.

Get in touch with me if you have more tweaks to show, and stay tuned for more updates!
