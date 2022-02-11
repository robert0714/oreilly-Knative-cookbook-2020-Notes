# oreilly-Knative-cookbook-2020-Notes
## Tutorial Sources
```bash
git clone -b master https://github.com/redhat-developer-demos/knative-tutorial
export TUTORIAL_HOME="$(pwd)/knative-tutorial"
```

[Redhat reference](https://redhat-developer-demos.github.io/knative-tutorial/knative-tutorial/setup/setup.html#download-tutorial-sources)
## Tutorial Tools


The following CLI tools are required for running the exercises in this tutorial. Please have them installed and configured before you get started with any of the tutorial chapters.


|**Tool**|**macOS**|**Fedora**|**windows**
|--------------------|----------------------|-----------------------------------|------------------------------------|
| Git                | [Download](https://git-scm.com/download/mac)|[Download]( https://git-scm.com/download/win)|[Download](https://git-scm.com/download/linux)|
| `Docker`           |                      |  `dnf install docker`             |  choco install docker-cli          |
| kubectl            |                      |                                   |  choco install kubernetes-cli      |
| stern              | brew install stern   | https://github.com/wercker/stern  |  choco install stern               |
| yq                 |                      |                                   |  choco install yq                  |
| httpie             | brew install httpie  | dnf install httpie                |  choco install httpie              |
| hey                | brew install hey     |                                   |                                    |
| watch              | brew install watch   | dnf install procps-ng             |  choco install docker-watch-forwarder |
| kubectx and kubens | brew install kubectx | https://github.com/ahmetb/kubectx |  choco install kubectx    kubens        |

  

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