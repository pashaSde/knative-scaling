# knative-scaling
Project to compare scaling policies in knative

## Setup Kubernetes Cluster

1. **Disable Swap:**
    ```sh
    sudo su
    swapoff -a
    vi /etc/fstab
    # Comment out the swap line
    ```

2. **Configure UFW:**
    ```sh
    vi /etc/ufw/sysctl.conf
    # Add the following lines:
    net/bridge/bridge-nf-call-ip6tables = 1
    net/bridge/bridge-nf-call-iptables = 1
    net/bridge/bridge-nf-call-arptables = 1
    ```

3. **Reboot the system.**

4. **Install necessary packages:**
    ```sh
    sudo su
    apt-get install ebtables ethtool
    sudo apt install -y curl apt-transport-https
    sudo sed -i '/swap/d' /etc/fstab # Can skip this if you already did it
    ```

5. **Load kernel modules:**
    ```sh
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ```

6. **Configure sysctl:**
    ```sh
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF
    sudo sysctl --system
    ```

7. **Install Containerd:**
    ```sh
    sudo apt install -y containerd
    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    sudo systemctl restart containerd
    sudo systemctl enable containerd
    ```

8. **Install Kubernetes Packages:**
    ```sh
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.asc
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.asc] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo apt update
    sudo apt install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

9. **Initialize the Master Node:**
    ```sh
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16
    ```

10. **Configure kubectl (On Master Node only):**
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

11. **Install CNI:**
    ```sh
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ```

12. **Join the worker nodes:**
    ```sh
    kubeadm token create --print-join-command
    # Run the join command on worker nodes
    ```

13. **Run pods on the master node:**
    ```sh
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```

## Install Knative

1. **Install Metrics Server:**
    ```sh
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    ```

2. **Install Knative Serving:**
    ```sh
    kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-crds.yaml
    kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.17.0/serving-core.yaml
    ```

3. **Install Kourier:**
    ```sh
    kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.17.0/kourier.yaml
    kubectl patch configmap/config-network --namespace knative-serving --type merge --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
    ```

## Install Prometheus and Grafana

1. **Install Prometheus:**
    ```sh
    kubectl apply -f https://github.com/prometheus-operator/prometheus-operator/blob/main/bundle.yaml
    ```

2. **Install Grafana:**
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/grafana/helm-charts/main/charts/grafana/templates/deployment.yaml
    ```

## Port Forwarding

1. **Port forward Prometheus:**
    ```sh
    kubectl port-forward svc/prometheus-k8s 9090:9090
    ```

2. **Port forward Grafana:**
    ```sh
    kubectl port-forward svc/grafana 3000:3000
    ```

## Setting Screens

1. **Create a new screen session:**
    ```sh
    screen -S mysession
    ```

2. **Detach from the screen session:**
    ```sh
    Ctrl-a d
    ```

3. **Reattach to the screen session:**
    ```sh
    screen -r mysession
    ```

## Creating Services

1. **Create a Knative service:**
    ```yaml
    apiVersion: serving.knative.dev/v1
    kind: Service
    metadata:
      name: ksvc-concurrency-scale-to-zero
      namespace: production
    spec:
      template:
        metadata:
          annotations:
            autoscaling.knative.dev/metric: "concurrency"
            autoscaling.knative.dev/target: "10"
            autoscaling.knative.dev/scale-to-zero-pod-retention-period: "0s"
        spec:
          containers:
            - image: ghcr.io/knative/autoscale-go:latest
    ```

## Running Artillery Tests

1. **Install Artillery:**
    ```sh
    npm install -g artillery
    ```

2. **Run Artillery tests:**
    ```sh
    artillery run <test-script>.yaml
    ```

Replace `<test-script>.yaml` with the path to your Artillery test script.
