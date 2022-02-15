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