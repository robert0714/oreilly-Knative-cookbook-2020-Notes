# oreilly-Knative-cookbook-2020-Notes
## Tutorial Sources
```bash
git clone -b master https://github.com/redhat-developer-demos/knative-tutorial
export TUTORIAL_HOME="$(pwd)/knative-tutorial"
```

[Redhat reference](https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial/setup/setup.html#download-tutorial-sources)
## Tutorial Tools
[#tools]
== Tools

The following CLI tools are required for running the exercises in this tutorial. Please have them installed and configured before you get started with any of the tutorial chapters.

[cols="4*^,4*.",options="header,+attributes"]
|===
|**Tool**|**macOS**|**Fedora**|**windows**

| Git
| https://git-scm.com/download/mac[Download]
| https://git-scm.com/download/win[Download]
| https://git-scm.com/download/linux[Download]

| `Docker`
| https://docs.docker.com/docker-for-mac/install[Docker for Mac]
| `dnf install docker`
| https://docs.docker.com/docker-for-windows/install[Docker for Windows]

| `kubectl {kubernetes-version}`
| https://storage.googleapis.com/kubernetes-release/release/{kubernetes-version}/bin/darwin/amd64/kubectl[Download]
| https://storage.googleapis.com/kubernetes-release/release/{kubernetes-version}/bin/linux/amd64/kubectl[Download]
| https://storage.googleapis.com/kubernetes-release/release/{kubernetes-version}/bin/windows/amd64/kubectl.exe[Download]

| https://github.com/wercker/stern[stern]
| `brew install stern`
| https://github.com/wercker/stern/releases/download/1.6.0/stern_linux_amd64[Download]
| https://github.com/wercker/stern/releases/download/1.11.0/stern_windows_amd64.exe[Download]

| https://github.com/mikefarah/yq[yq v2.4.1]
| https://github.com/mikefarah/yq/releases/download/2.4.1/yq_darwin_amd64[Download]
| https://github.com/mikefarah/yq/releases/download/2.4.1/yq_linux_amd64[Download]
| https://github.com/mikefarah/yq/releases/download/2.4.1/yq_windows_amd64.exe[Download]

| https://httpie.org/[httpie]
| `brew install httpie`
| `dnf install httpie`
| https://httpie.org/doc#windows-etc

| https://github.com/rakyll/hey[hey]
| https://storage.googleapis.com/hey-release/hey_darwin_amd64[Download]
| https://storage.googleapis.com/jblabs/dist/hey_linux_v0.1.2[Download]
| https://storage.googleapis.com/jblabs/dist/hey_win_v0.1.2.exe[Download]

| watch
| `brew install watch`
| `dnf install procps-ng`
|

| kubectx and kubens
| `brew install kubectx`
| https://github.com/ahmetb/kubectx
|

|===

## Creating Kubernetes Namespaces for This Book’s Recipes
The recipes in each chapter will be deployed in the namespace dedicated for the chapter. Each chapter will instruct you to switch to the respective namespace. Run the following command to create all the required namespaces for this book:
```bash
$ kubectl create namespace chapter-2
$ kubectl create namespace chapter-3
$ kubectl create namespace chapter-4
$ kubectl create namespace chapter-5
$ kubectl create namespace chapter-6
$ kubectl create namespace chapter-7
```
### Why Switch Namespaces?
Kubernetes by default creates the ``default`` namespace. You can control the namespace of the resource by specifying the ``--namespace`` or ``-n`` option to all your Kubernetes commands. By switching to the right namespace, you can be assured that your Kubernetes resources are created in the correct place as needed by the recipes.

You can use ``kubectl`` to switch to the required namespace. The following command shows how to use kubectl to switch to a namespace called ``chapter-1``:
```bash
$ kubectl config set-context --current --namespace=chapter-1
```
Or you can use the kubens utility to set your current namespace to be ``chapter-1``:
```bash
$ kubens chapter-1
```