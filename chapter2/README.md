# Chapter 2 Understanding Knative Serving
### Before You Begin
All the recipes in this chapter will be executed from the directory $BOOK_HOME/basics,
so change to the recipe directory by running:
```bash
$ cd $BOOK_HOME/basics
```

The recipes of this chapter will be deployed in the chapter-2 namespace, so switch to
the chapter-2 namespace with the following command:
```bash
$ kubectl config set-context --current --namespace=chapter-2
```
## 2.1 Deploying a Knative Service
```bash
$ kubectl -n chapter-2 apply -f service.yaml
```
The very first deployment of the service will take additional time as the container image needs to be downloaded to your Kubernetes cluster. A successful deployment will result in a pod with a similar (though not identical) name in the chapter-2 namespace:
```bash
$ kubectl get pods -w
NAME                                        READY   STATUS    RESTARTS   AGE
greeter-00001-deployment-5cdfc7b889-vdphh   0/2     Pending   0          0s
greeter-00001-deployment-5cdfc7b889-vdphh   0/2     Pending   0          0s
greeter-00001-deployment-5cdfc7b889-vdphh   0/2     ContainerCreating   0          0s
greeter-00001-deployment-5cdfc7b889-vdphh   1/2     Running             0          2s
greeter-00001-deployment-5cdfc7b889-vdphh   2/2     Running             0          2s

```
The deployment of a Knative Serving Service results in a ksvc being created. You can query for available ksvc services using the command kubectl get ksvc. In order to invoke the service you will need its URL, which is created by the Knative Route. To discover greeter’s Knative Route, run the following command:

```bash
$ kubectl -n chapter-2 get ksvc greeter
```
### Using kourier not istio
  
```bash
export ksvc_url=$(kubectl get ksvc -n chapter-2 -o json |jq -r ".items[0].status.url")
export hostname=${ksvc_url:7}
export ingress_ip=$(kubectl -n kourier-system get svc  -o json |jq -r ".items[0].status.loadBalancer.ingress[0].ip")
curl -H "Host:$hostname"  $ingress_ip
```
The preceding command will output the following (some columns omitted for brevity):
```bash
Hi  greeter => 9861675f8845 : 1
```
The deployment of the ``greeter`` service has also created a Knative Configuration. The Knative Configuration holds the current state of the Knative Service—that is, which revision of the service should receive the requests. Currently you should only have one revision named ``greeter-v1``; therefore, running the command ``kubectl -n chapter-2 get configurations greeter`` should result in a single greeter configuration as shown here:
```bash
$  kubectl -n chapter-2 get configurations greeter
NAME      LATESTCREATED   LATESTREADY     READY   REASON
greeter   greeter-00001   greeter-00001   True
```

## 2.2 Updating a Knative Service Configuration
```bash
$ kubectl -n chapter-2 apply -f service-env.yaml
```
This command will result in a new Kubernetes Deployment:

```bash
$  kubectl get deployments
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
greeter-00001-deployment   0/0     0            0           58m
greeter-v2-deployment      1/1     1            1           24s
```
Your ``greeter-v1`` deployment had no requests for approximately 60 seconds, which is the Knative default scale-down time window. ``greeter-v1`` was automatically scaled down to zero since it lacked invocations. This is the key to how Knative Serving helps you save expensive cloud resources using serverless services. You will learn more about this feature in Chapter 3.

Any update to the Knative Service will create a new revision. You should now have two revisions of the Knative Service ``greeter``. Run the following command to see the available revisions:
```bash
$ kubectl -n chapter-2 get revisions
```
You should see two revisions, each one associated with ``greeter-v1`` and ``greeter-v2``, respectively. The following listing shows the available revisions for the greeter service:
```bash
$  kubectl -n chapter-2 get revisions
NAME            CONFIG NAME   K8S SERVICE NAME   GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
greeter-00001   greeter                          1            True             0                 0
greeter-v2      greeter                          2            True             0                 0
```
It uses the Knative Configuration, which is responsible for holding the state of a Knative Service—that is, how to distribute traffic between various revisions. By default, it routes 100% of the traffic to any newly created revision, which in this case is **greeter-v2**. To verify, run:
```bash
$ kubectl get configurations greeter
NAME      LATESTCREATED   LATESTREADY   READY   REASON
greeter   greeter-v2      greeter-v2    True
```
## 2.3 Distributing Traffic Between Knative Service Revisions
```bash
$ kubectl -n chapter-2 apply -f service-traffic.yaml
```
http://mikefarah.github.io/yq/
```bash
$ yq --version
yq (https://github.com/mikefarah/yq/) version 4.15.1

$ kubectl -n chapter-2 get ksvc greeter -o yaml  | yq  e '.status.traffic'  -
- latestRevision: false
  percent: 50
  revisionName: greeter-v2
  tag: v2
  url: http://v2-greeter.chapter-2.192.168.59.200.sslip.io
- latestRevision: false
  percent: 50
  revisionName: greeter-v3
  tag: v3
  url: http://v3-greeter.chapter-2.192.168.59.200.sslip.io
```
or
```bash
$ kubectl -n chapter-2 get ksvc greeter -o yaml  | yq  e '.status.traffic[$item].url'  -
http://v2-greeter.chapter-2.192.168.59.200.sslip.io
http://v3-greeter.chapter-2.192.168.59.200.sslip.io

```
## 2.4 Applying the Blue-Green Deployment Pattern
```bash
$ kubectl -n chapter-2 apply -f service-traffic-v2.yaml
```
Before you apply the resource ``$BOOK_HOME/basics/service-pinned.yaml`` or  ````, call the greeter service again to verify that it is still providing the response from greeter-v2 that includes Namaste
```bash
export ksvc_url=$(kubectl get ksvc -n chapter-2 -o json |jq -r ".items[0].status.url")
export hostname=${ksvc_url:7}
export ingress_ip=$(kubectl -n kourier-system get svc  -o json |jq -r ".items[0].status.loadBalancer.ingress[0].ip")
curl -H "Host:$hostname"  $ingress_ip
```
You will notice that the command does not create any new Configuration/Revision/Deployment as there was no application update (e.g., image tag, environment variable, etc.), but when you call the service, Knative scales up the greeter-v1 and the service responds with the text ``Hi greeter ⇒ 9861675f8845 : 1``.
```bash
$ kubectl get pods -w
NAME                                     READY   STATUS    RESTARTS   AGE
greeter-v3-deployment-54f5c8994f-dck9m   2/2     Running   0          4s
greeter-v3-deployment-54f5c8994f-dck9m   2/2     Terminating   0          63s
greeter-v2-deployment-fc5476676-wsbjm    0/2     Pending       0          0s
greeter-v2-deployment-fc5476676-wsbjm    0/2     Pending       0          0s
greeter-v2-deployment-fc5476676-wsbjm    0/2     ContainerCreating   0          0s
greeter-v2-deployment-fc5476676-wsbjm    1/2     Running             0          2s
greeter-v2-deployment-fc5476676-wsbjm    2/2     Running             0          2s
greeter-v3-deployment-54f5c8994f-dck9m   0/2     Terminating         0          94s
greeter-v3-deployment-54f5c8994f-dck9m   0/2     Terminating         0          94s
greeter-v3-deployment-54f5c8994f-dck9m   0/2     Terminating         0          94s
greeter-v2-deployment-fc5476676-wsbjm    2/2     Terminating         0          64s
greeter-v2-deployment-fc5476676-wsbjm    0/2     Terminating         0          95s
greeter-v2-deployment-fc5476676-wsbjm    0/2     Terminating         0          95s
greeter-v2-deployment-fc5476676-wsbjm    0/2     Terminating         0          95s
```
## 2.4 Applying the Canary Release Pattern
```bash
$ kubectl -n chapter-2 apply -f service-canary.yaml
```