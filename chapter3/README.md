# Chapter 3. Autoscaling Knative Services
### Before You Begin
All the recipes in this chapter will be executed from the directory $BOOK_HOME/basics,
so change to the recipe directory by running:
```bash
$ cd $BOOK_HOME/basics
```

The recipes of this chapter will be deployed in the chapter-3 namespace, so switch to
the chapter-3 namespace with the following command:
```bash
$ kubectl config set-context --current --namespace=chapter-3
```
## 3.1 Configuring Knative Service for Autoscaling
```
http://mikefarah.github.io/yq/
```bash
$ yq --version
yq (https://github.com/mikefarah/yq/) version 4.15.1

$ kubectl -n knative-serving get cm config-autoscaler -o yaml  | yq  e '.data'  -
```
The following code snippet provides an abridged version of the config-autoscaler ConfigMap contents. We focus on the few properties that impact the recipes included in this chapter:
```yaml
apiVersion: v1
data:
  container-concurrency-target-default: "100"    ❶
  enable-scale-to-zero: "true"                   ❷
  stable-window: "60s"                           ❸
  scale-to-zero-grace-period: "30s"              ❹ 
```
❶ The default container concurrency for each service pod; defaults to 100

❷ Flag to enable or disable scale down to zero; defaults to true

❸ The time period in which the requests are monitored for calls and metrics; defaults to 60 seconds

❹ The time period within which the inactive pods are terminated; defaults to 30 seconds

## 3.2 Observing Scale-to-Zero
After deployment of your Knative Service as described in Chapter 2, simply watch the pod lifecycle with the following command:
```bash
$ kubectl get pods -w
```
If you have not deployed the greeter Knative Serving Service, run:
```bash
$ kubectl -n chapter-3 apply -f  service.yaml
```
To make sure the pod is up and running, use the script call.sh:
```bash
export ksvc_url=$(kubectl get ksvc -n chapter-3 -o json |jq -r ".items[0].status.url")
export hostname=${ksvc_url:7}
export ingress_ip=$(kubectl -n kourier-system get svc  -o json |jq -r ".items[0].status.loadBalancer.ingress[0].ip")
curl -H "Host:$hostname"  $ingress_ip

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    34  100    34    0     0     11      0  0:00:03  0:00:02  0:00:01    11   Hi  greeter => '9861675f8845' : 1
```
## 3.3 Configuring Your Knative Service to Handle Request Spikes
### Problem
You want to configure your Knative Service to handle sudden request spikes by changing the default concurrency setting.

### Solution
In your Knative Serving Service YAML, you can add annotations that will override the default behavior and autoscaling parameters:
```yaml
autoscaling.knative.dev/target: "10"
```
The following listing illustrates the Knative Service Revision Template that adds the container concurrency annotation to reconfigure it from the default 100 to 10:
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: prime-generator
spec:
  template:
    metadata:
      name: prime-generator-v1
      annotations:
        # Target 10 in-flight-requests per pod.
        autoscaling.knative.dev/target: "10" ❶ 
    spec:
      containers:
      - image: quay.io/rhdevelopers/prime-generator:v27-quarkus
        livenessProbe:
          httpGet:
            path: /healthz
        readinessProbe:
          httpGet:
            path: /healthz
```
❶ Configure the container concurrency to be 10

### Discussion
By default, the Knative Service container concurrency is set to ``100`` requests per pod. With the ``autoscaling.knative.dev/target`` annotation you are now overriding that value to be ``10``. You may also set this value to 0, where Knative will autoconfigure the value. In the absence of the annotation ``autoscaling.knative.dev/target``, Knative by default sets this value to be ``0``.

Since we need to simulate the slowness in response to observe autoscaling, the service that you will use for doing the load test is a prime number generator using the Sieve of Eratosthenes. The Sieve of Eratosthenes is one of the slowest and least optimal ways to compute prime numbers within a range. The application tries to spice up the slowness by adding memory load, which makes the Knative Service respond slowly, thereby allowing it to autoscale.

Navigate to the recipe directory $BOOK_HOME/chapter3 and run:
```bash
$ kubectl apply -n chapter-3 -f service-10.yaml
```
The very first deployment of a Knative Serving Service will automatically scale to a single pod; wait for that service pod to come up:
```bash
$ kubectl get -n chapter-3 pods -w
NAME                                                READY   STATUS    RESTARTS   AGE
prime-generator-00001-deployment-645c7bd755-whwsn   0/2     Pending   0          0s
prime-generator-00001-deployment-645c7bd755-whwsn   0/2     Pending   0          0s
prime-generator-00001-deployment-645c7bd755-whwsn   0/2     ContainerCreating   0          0s
prime-generator-00001-deployment-645c7bd755-whwsn   1/2     Running             0          8s
prime-generator-00001-deployment-645c7bd755-whwsn   2/2     Running             0          8s
```
You can test the prime-generator service by using the script $BOOK_HOME/bin/call.sh with the service name prime-generator as a parameter:
```
export ksvc_url=$(kubectl get ksvc prime-generator  -n chapter-3 -o json |jq -r ".status.url")
export hostname=${ksvc_url:7}
export ingress_ip=$(kubectl -n kourier-system get svc  -o json |jq -r ".items[0].status.loadBalancer.ingress[0].ip")
curl -H "Host:$hostname"  $ingress_ip
```
Then you will see :
```bash
Value should be greater than 1 but recevied 0
```

In order to verify your updated concurrency setting (e.g., ``autoscaling.knative.dev/target: "10"``) you need to drive enough load into the system to observe its behavior.

Sending ``50`` concurrent requests will cause the Knative autoscaler to scale up ``7`` service pods. The formula to calculate the target number of pods is as follows:

```
number of pods = total number of requests / container-concurrency
```

In the sample code repository, we have provided a load testing script called load.sh, and it leverages a command-line utility called hey. Run the following command to send 50 concurrent requests to the ``prime-generator`` service:
```bash
#!/bin/bash
export ksvc_url=$(kubectl get ksvc prime-generator  -n chapter-3 -o json |jq -r ".status.url")
export hostname=${ksvc_url:7}
export ingress_ip=$(kubectl -n kourier-system get svc  -o json |jq -r ".items[0].status.loadBalancer.ingress[0].ip")
curl -H "Host:$hostname"  $ingress_ip

hey -c 50 -z 10s \
  -host "$hostname" \
 "http://$ingress_ip/?sleep=3&upto=10000&memload=100" 
```

❶ Invoke the hey load testing tool with a concurrency of 50 requests and for a duration of 10 seconds

❷ As you did earlier, pass the Host header; in this case it will be prime-generator.chapter-3.example.com

❸ The request URL parameters:
  * ``sleep``: Simulates slow-performing operations so that the requests pile up by sleeping for 3 seconds
  * ``upto``: Calculates the prime number up to this maximum
  * ``load``: Simulates the memory load of 100 megabytes(mb)


To watch the autoscaling in action, you should open two terminal windows, one to run the watch command  ``kubectl get pods -n chapter-3 -w`` and the other to run the load test script $BOOK_HOME/bin/load.sh.
```bash
kubectl get pods -n chapter-3 -w

prime-generator-00001-deployment-645c7bd755-wl2zb   0/2     Pending             0          1s
prime-generator-00001-deployment-645c7bd755-wl2zb   0/2     ContainerCreating   0          1s
prime-generator-00001-deployment-645c7bd755-4k9c9   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-bt42w   0/2     ContainerCreating   0          0s
prime-generator-00001-deployment-645c7bd755-hncn7   0/2     ContainerCreating   0          0s
prime-generator-00001-deployment-645c7bd755-4k9c9   0/2     ContainerCreating   0          0s
prime-generator-00001-deployment-645c7bd755-qvjkq   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-qvjkq   0/2     ContainerCreating   0          1s
prime-generator-00001-deployment-645c7bd755-hncn7   1/2     Running             0          3s
prime-generator-00001-deployment-645c7bd755-ssrvl   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-ssrvl   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-ncpkw   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-hncn7   2/2     Running             0          3s
prime-generator-00001-deployment-645c7bd755-ncpkw   0/2     Pending             0          0s
prime-generator-00001-deployment-645c7bd755-ssrvl   0/2     ContainerCreating   0          1s
prime-generator-00001-deployment-645c7bd755-wl2zb   1/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-wl2zb   2/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-bt42w   1/2     Running             0          4s
prime-generator-00001-deployment-645c7bd755-bt42w   2/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-ncpkw   0/2     ContainerCreating   0          2s
prime-generator-00001-deployment-645c7bd755-4k9c9   1/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-4k9c9   2/2     Running             0          6s
prime-generator-00001-deployment-645c7bd755-ssrvl   1/2     Running             0          3s
prime-generator-00001-deployment-645c7bd755-ssrvl   2/2     Running             0          4s
prime-generator-00001-deployment-645c7bd755-qvjkq   1/2     Running             0          6s
prime-generator-00001-deployment-645c7bd755-qvjkq   2/2     Running             0          6s
prime-generator-00001-deployment-645c7bd755-ncpkw   1/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-ncpkw   2/2     Running             0          5s
prime-generator-00001-deployment-645c7bd755-ncpkw   2/2     Terminating         0          64s
```
And the result:
```bash

Summary:
  Total:        14.2671 secs
  Slowest:      11.2607 secs
  Fastest:      3.0051 secs
  Average:      5.8423 secs
  Requests/sec: 6.8690

  Total data:   392 bytes
  Size/request: 4 bytes

Response time histogram:
  3.005 [1]     |■■
  3.831 [23]    |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  4.656 [0]     |
  5.482 [24]    |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  6.307 [22]    |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  7.133 [0]     |
  7.958 [0]     |
  8.784 [24]    |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  9.610 [2]     |■■■
  10.435 [0]    |
  11.261 [2]    |■■■


Latency distribution:
  10% in 3.0138 secs
  25% in 5.2688 secs
  50% in 5.9760 secs
  75% in 8.2991 secs
  90% in 8.3555 secs
  95% in 8.8689 secs
  0% in 0.0000 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0024 secs, 3.0051 secs, 11.2607 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0005 secs, 0.0000 secs, 0.0116 secs
  resp wait:    5.8393 secs, 3.0051 secs, 11.2571 secs
  resp read:    0.0001 secs, 0.0000 secs, 0.0006 secs

Status code distribution:
  [200] 98 responses
```
## 3.4 Cold Start Latency
### Problem
You want to avoid the wait time involved in scaling from zero to n pods based on request volume by setting a floor—a minScale number of pods. You may also want to set a ceiling—a maxScale number of pods.

### Solution
The ``minScale`` and ``maxScale`` annotations on the Knative Service Template allow you to set limits on the minimum and maximum number of pods that can be scaled:

#### minScale
By default, Knative will scale-to-zero—that is, your service will scale-to-zero pods when no requests arrive within the ``stable-window`` time period. When the next requests come in, Knative will autoscale to the appropriate number of pods to handle those requests. This starting from zero and the associated wait time is known as ``cold start latency``.

If your application needs to stay particularly responsive and/or has a long startup time, then it may be beneficial to keep a minimum number of pods always up. This technique is also called pod warming. With Knative Serving this is achieved by adding the annotation ``autoscaling.knative.dev/minScale`` to the Knative Service YAML.

#### maxScale
Knative by default does not set an upper limit to the number of pods. This means you are at risk of exceeding your computational resource limits. In order to mitigate the risk, Knative Serving allows you to add the annotation ``autoscaling.knative.dev/maxScale`` to the Knative Service YAML. With ``maxScale`` you can restrict the upper limit of the autoscaler.

In the following section you will set the ``minScale`` and ``maxScale`` on the Knative Service Revision Template and run a load test. You will notice that the autoscaling will max out at 5 pods and once the requests are responded to, it will scale down to 2 and not 0.

The following code snippet shows the Knative Service Revision Template with ``minScale`` and ``maxScale`` annotations configured:
```yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: prime-generator
spec:
  template:
    metadata:
      name: prime-generator-v2
      annotations:
        # the minimum number of pods to scale down to
        autoscaling.knative.dev/minScale: "2" 
        # the maximum number of pods to scale up to
        autoscaling.knative.dev/maxScale: "5" 
        # Target 10 in-flight-requests per pod.
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: quay.io/rhdevelopers/prime-generator:v27-quarkus
          livenessProbe:
            httpGet:
              path: /healthz
          readinessProbe:
            httpGet:
              path: /healthz
```
❶ The minimum number of pods is set to ``2``; these pods should always be available even after the Knative Service has exceeded the ``stable-window``.
❷ The maximum number of pods is set to ``5``, the number of pods the service can scale up to when it receives more requests than its container concurrency limits.

To see these settings in action, first watch your pod lifecycle with the following command:
```bash
$ kubectl get pods -w
No resources found.
```
Depending on when you last invoked call.sh or load.sh, there should be no pods available as Knative would have terminated the inactive pods.

Now, apply an update to the prime-generator service that includes the minScale and maxScale annotations:
```bash
$ kubectl apply -n chapter-3 -f service-min-max-scale.yaml
```
You should see an immediate response in your watch kubectl get pods terminal as shown here:
```bash
$ kubectl get pods -w
NAME                                             READY   STATUS    AGE
prime-generator-v2-deployment-84f459b57f-8kp6m   2/2     Running   14s
prime-generator-v2-deployment-84f459b57f-rlrqt   2/2     Running   10s
```
### Discussion
You will notice that the prime-generator has been scaled up to 2 replicas as described by the ``autoscaling.knative.dev/minScale`` value and those pods will not be automatically scaled down to zero even after the termination period.

The final test is to attempt to overload the service with too many requests by running the load test script $BOOK_HOME/bin/load.sh. You will observe that maxScale will limit the autoscaler to 5 pods:
```bash
$ $BOOK_HOME/bin/load.sh
$ watch kubectl get pods
NAME                                             READY   STATUS    AGE
prime-generator-v2-deployment-84f459b57f-6vxxx   2/2     Running   5s
prime-generator-v2-deployment-84f459b57f-8kp6m   2/2     Running   2m35s
prime-generator-v2-deployment-84f459b57f-8trh2   2/2     Running   5s
prime-generator-v2-deployment-84f459b57f-ldg8m   2/2     Running   5s
prime-generator-v2-deployment-84f459b57f-rlrqt   2/2     Running   2m39s
```
And if you wait long enough, without another spike in requests, Knative Serving will scale down the unwanted pods:
```bash
NAME                                             READY   STATUS        AGE
prime-generator-v2-deployment-84f459b57f-6vxxx   2/2     Terminating   68s
prime-generator-v2-deployment-84f459b57f-8kp6m   2/2     Running       10m
prime-generator-v2-deployment-84f459b57f-8trh2   2/2     Terminating   68s
prime-generator-v2-deployment-84f459b57f-ldg8m   2/2     Terminating   68s
prime-generator-v2-deployment-84f459b57f-rlrqt   2/2     Running       10m
```

In this chapter, you learned about Knative Serving autoscaling behaviors by observing the default configuration and behavior, overriding the default Knative Serving concurrency configuration, and addressing cold start latency and an unlimited upper boundary.

In the next chapter, you will learn how to make your Knative Service respond to external events, such as a message received at a message broker topic.