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

## Setting Knative Minikube Enviroment by kn-plugin-quickstart
1. You can download the latest binaries from the [kn-plugin-quickstart  Releases page](https://github.com/knative-sandbox/kn-plugin-quickstart/releases) .
2. You can run it standalone, just put it on your system path and make sure it is executable.
3. You can install it as a plugin of the ``kn`` client to run:
    * Follow the [documentation](https://github.com/knative/client/blob/main/docs/README.md#installing-kn) to install ``kn client`` if you don't have it
    * Copy the ``kn-quickstart`` binary to a directory on your ``PATH`` (for example, /usr/local/bin) and make sure its filename is kn-quickstart
    * Run ``kn plugin list`` to verify that the ``kn-quickstart`` plugin is installed successfully
4. After the plugin is installed, you can use ``kn quickstart`` to run its related subcommands.Install Knative and Kubernetes in a minikube instance by running:
    ```bash
    kn quickstart minikube
    ```
5. After the plugin is finished, verify you have a cluster called knative:
    ```bash
    minikube profile list
    ```
6. If your minikube is in windows ,your knative application would fail to start. You need to [configure DNS](https://knative.dev/docs/install/serving/install-serving-with-yaml/#configure-dns):
    * Knative provides a Kubernetes Job called ``default-domain`` that configures Knative Serving to use ``sslip.io`` as the default DNS suffix.
        ```bash
        kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-default-domain.yaml
        ```
    * Let [``minikube tunnel``](https://minikube.sigs.k8s.io/docs/commands/tunnel/) is running.
    * Configure metallb (it's ip range must be nearby minikube's ip )
        ```bash
        $ minikube -p knative addons enable metallb
        - Using image metallb/speaker:v0.9.6
        - Using image metallb/controller:v0.9.6
        * The 'metallb' addon is enabled

        $ minikube -p knative addons configure metallb
        -- Enter Load Balancer Start IP: 192.168.59.200
        -- Enter Load Balancer End IP: 192.168.59.210
        - Using image metallb/controller:v0.9.6
        - Using image metallb/speaker:v0.9.6
        * metallb was successfully configured
        ```
## Creating Kubernetes Namespaces for This Book’s Recipes
The recipes in each chapter will be deployed in the namespace dedicated for the chapter. Each chapter will instruct you to switch to the respective namespace. Run the following command to create all the required namespaces for this book:
```bash
kubectl create namespace chapter-2
kubectl create namespace chapter-3
kubectl create namespace chapter-4
kubectl create namespace chapter-5
kubectl create namespace chapter-6
kubectl create namespace chapter-7
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