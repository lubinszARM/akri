# User Guide
To best understand the benefits of Akri and jump into using it, we recommend you start off by completing the [end to end
demo](./end-to-end-demo.md). In the demo, you will see Akri discover mock video cameras and a streaming app display the
footage from those cameras. It includes instructions on K8s cluster setup. If you would like to perform the demo on a
cluster of Raspberry Pi 4's, see the [Raspberry Pi 4 demo](./end-to-end-demo-rpi4.md).

## Getting Started
To get started using Akri, you must first decide what you want to discover and whether Akri current supports a protocol
that can be used to discover resources of that type. To see the list of currently supported protocols, see our
[roadmap](./roadmap.md).

### Understanding Akri Helm charts
Akri is most easily deployed with Helm charts.  Helm charts provide convenient packaging and configuration.

Starting in v0.0.36, an **akri-dev** Helm chart will be published for each build version.  Each Akri build is verified
with end-to-end tests on Kubernetes, K3s, and MicroK8s.  These builds may be less stable than our Releases.  You can
deploy these versions of Akri with this command (note: **akri-dev**):
```sh
helm repo add akri-helm-charts https://deislabs.github.io/akri/
helm install akri akri-helm-charts/akri-dev
```

Starting after Release v0.0.35, an **akri** Helm chart will be published for each
[Release](https://github.com/deislabs/akri/releases).  Releases will generally reflect milestones and will have more
rigorous testing.  You can deploy Release versions of Akri with this command (note: **akri**):
```sh
helm repo add akri-helm-charts https://deislabs.github.io/akri/
helm install akri akri-helm-charts/akri
```

### Setting up your cluster
1. Before deploying Akri, you must have a Kubernetes (v1.16 or higher) cluster running and `kubectl` installed. All
   nodes must be Linux. All of the Akri component containers are currently built for amd64 or arm64v8, so all nodes must
   have one of these platforms.

### Deploying Akri
1. Install Helm
    ```sh
    curl -L https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```
1. Provide runtime-specific configuration to enable Akri and Helm

    1. If using **K3s**, point to `kubeconfig` for Helm and configure Akri to use the K3s embedded crictl.
        ```sh
        # Helm uses $KUBECONFIG to find the Kubernetes configuration
        export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
        # Configure Akri to use K3s' embedded crictl and CRI socket
        export AKRI_HELM_CRICTL_CONFIGURATION="--set agent.host.crictl=/usr/local/bin/crictl --set agent.host.dockerShimSock=/run/k3s/containerd/containerd.sock"
        ```
    1. If using **MicroK8s**, enable CoreDNS, RBAC (optional), Helm, and privileged Pods. Also, install crictl, and
       configure Akri to use MicroK8s' CRI socket.
        ```sh
        # Enable CoreDNS, RBAC and Helm
        microk8s enable dns rbac helm3

        # Enable privileged pods and restart MicroK8s.
        echo "--allow-privileged=true" >> /var/snap/microk8s/current/args/kube-apiserver
        sudo microk8s stop && microk8s start

        # Install crictl locally (note: there are no known version
        # limitations, any crictl version is expected to work)
        VERSION="v1.17.0"
        curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
        sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
        rm -f crictl-$VERSION-linux-amd64.tar.gz

        # Configure Akri to use MicroK8s' CRI socket
        export AKRI_HELM_CRICTL_CONFIGURATION="--set agent.host.crictl=/usr/local/bin/crictl --set agent.host.dockerShimSock=/var/snap/microk8s/common/run/containerd.sock"
        ```
        If you don't have existing kubectl and helm installations, you can add aliases. If you do not want to set an
        alias, add microk8s in front of all kubectl and helm commands.
        ```sh
        alias kubectl='microk8s kubectl'
        alias helm='microk8s helm3'
        ```
    1. If using **Kubernetes**, Helm and crictl do not require additional configuration.

1. When installing the Akri Helm chart, you can specify what Configuration to apply by specifying the discovery protocol
   that will be used in the Configuration. This is done in the setting `--set <protocol>.enabled=true` below. Here,
   `<protocol>` could be `udev` or `ONVIF`. Helm will automatically apply the default Configuration for that protocol to
   the cluster. You can set values in the Helm install command to customize the Configuration. To explore the values you
   can set, see our documentation on customizing the provided [ONVIF](./onvif-configuration.md) and
   [udev](./udev-configuration.md) Configuration templates.

    The Helm settings can also be used to customize where the Akri Controller runs. By default the Controller can be
    deployed to any control plane or worker node. These settings can be changed by adding extra settings when installing
    Akri below. If you don't want the Controller to ever be scheduled to control plane nodes, add `--set
    controller.allowOnControlPlane=false` to your install command below. Conversely, if you only want the Controller to
    run on control plane nodes, add `--set controller.onlyOnControlPlane=true`. This will guarantee the Controller only
    runs on nodes with the label (key, value) of (`node-role.kubernetes.io/master`, ""), which is the default label for
    the control plane node for Kubernetes. However, control plane nodes on MicroK8s and K3s do not have this label by
    default, so you can add it by running `kubectl label node ${HOSTNAME,,} node-role.kubernetes.io/master=
    --overwrite=true`. 

    Run the following to fetch the Akri Helm chart, install Akri, and apply the default configuration for `<protocol>`,
    optionally specifying the image for the broker pod that should be deployed to utilize each discovered device.
    ```sh
    helm repo add akri-helm-charts https://deislabs.github.io/akri/
    helm install akri akri-helm-charts/akri-dev \
        $AKRI_HELM_CRICTL_CONFIGURATION \
        --set useLatestContainers=true \
        --set <protocol>.enabled=true \
        # --set <protocol>.brokerPod.image.repository=<your broker image> \
        # apply any additional settings here
    ```
    Run `kubectl get crd`, and you should see Akri's two CRDs listed. Run `kubectl get pods -o wide`, and you should see
    the Akri Controller pod, Agent pods, and broker pods if a broker was specified. Run `kubectl get akric`, and you
    should see the Configuration for the protocol you specified.  If devices were discovered, the instances can be seen
    by running `kubectl get akrii` and further inspected by running `kubectl get akrii <protocol>-<ID> -o yaml`.
1. Delete the configuration and watch the instances, pods, and services (if you specified a broker image) be deleted.
    ```sh
    kubectl delete akric akri-<protocol>
    watch kubectl get pods,services,akric,akrii -o wide
    ```

### Modifying your Akri installation
See the [Modifying a Akri Installation document](./modifying-akri-installation.md) for more information on how to modify
your already deployed Akri installation.
