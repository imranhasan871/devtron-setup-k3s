# 🚀 Deploying and Managing K3s Kubernetes Cluster with Metallb, Nginx Ingress, and Cert-Manager 🛠️

## 📑 Table of Contents

1. Introduction
2. Prerequisites
3. Installation Steps

    3.1. **Install K3s on the Server Node** 🌐

    ```bash
    # Replace [OPTIONS] with your desired configurations
    curl -sfL https://get.k3s.io | sh -
    ```

    3.2. **Join Agent Nodes to the Cluster** 🤝

    ```bash
    # Run this on the agent nodes
    curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_NODE_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
    ```

    3.3. **Install Metallb** ⚙️

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
    kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
    ```

    3.4. **Install Nginx Ingress Controller** 🌐

    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
    ```

    3.5. **Install Cert-Manager** 🛡️

    ```bash
    kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.crds.yaml
    kubectl create namespace cert-manager
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml
    ```

4. **Configure Metallb** ⚙️

    4.1. **Create ConfigMap for Metallb** 📝

    ```bash
    kubectl apply -f metallb-configmap.yaml
    ```

    (Provide a sample `metallb-configmap.yaml` file with your specific configurations)

5. **Configure Nginx Ingress Controller** 🌐

    5.1. **Create Ingress Resources** 🚧

    ```yaml
    # Example Ingress resource for SSL termination
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
        name: example-ingress
        namespace: default
    spec:
        rules:
            - host: example.com
              http:
                  paths:
                      - path: /
                        pathType: Prefix
                        backend:
                            service:
                                name: example-service
                                port:
                                    number: 80
        tls:
            - hosts:
                  - example.com
              secretName: example-tls
    ```

6. **Configure Cert-Manager** 🛡️

    6.1. **Create Issuer for Let's Encrypt** 🌐

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
        name: letsencrypt-prod
    spec:
        acme:
            server: https://acme-v02.api.letsencrypt.org/directory
            email: your-email@example.com
            privateKeySecretRef:
                name: letsencrypt-prod
            solvers:
                - http01:
                      ingress:
                          class: nginx
    ```

7. **Adding More Agent Nodes** 🔄

    7.1. **Generate Join Token on the Server Node** 🗝️

    ```bash
    sudo cat /var/lib/rancher/k3s/server/node-token
    ```

    7.2. **Run Join Command on New Agent Nodes** 🌐

    ```bash
    curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_NODE_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
    ```

## Installation of Devtron on K3s Cluster 🛠️

### Before you begin 🚀

Before we get started and install Devtron, you must set up the cluster on your server and install the prerequisite requirements:

- Create a cluster using [Minikube](https://minikube.sigs.k8s.io/docs/start/) or [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) or [K3s](https://rancher.com/docs/k3s/latest/en/installation/).
- Install [Helm3](https://helm.sh/docs/intro/install/).
- Install [kubectl](https://kubernetes.io/docs/tasks/tools/).

### Install Devtron 🛠️

To install Devtron on the `k3s` cluster, run the following command:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

helm repo add devtron https://helm.devtron.ai

helm install devtron devtron/devtron-operator \
--create-namespace --namespace devtroncd \
--set components.devtron.service.type=NodePort

```

### Devtron Admin credentials 👩‍💼

When you install Devtron for the first time, it creates a default admin user and password (with unrestricted access to Devtron). You can use those credentials to log in as an administrator.

After the initial login, we recommend you set up any SSO service like Google, GitHub, etc., and then add other users (including yourself). Subsequently, all the users can use the same SSO (let's say, GitHub) to log in to Devtron's dashboard.

The section below will help you understand the process of getting the administrator credentials.

#### For Devtron version v0.6.0 and higher

**Username**: `admin` 🧑‍💼 <br>
**Password**: Run the following command to get the admin password:

```bash
kubectl -n devtroncd get secret devtron-secret \
-o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

## Ingress Setup 🌐

After Devtron is installed, Devtron is accessible through the service `devtron-service`.
If you want to access Devtron through ingress, edit `devtron-service` and change the load balancer to ClusterIP. You can do this using the `kubectl patch` command:

```bash
kubectl patch -n devtroncd svc devtron-service -p '{"spec": {"ports": [{"port": 80,"targetPort": "devtron","protocol": "TCP","name": "devtron"}],"type": "ClusterIP","selector": {"app": "devtron"}}}'
```

After this, create an ingress by applying the ingress YAML file.
You can use [this YAML file](<https://github.com/devtron-labs/devtron/blob/main/manifests/yamls/devtron-ingress.yaml>) to create ingress to access Devtron:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/app-root: /dashboard
    cert-manager.io/cluster-issuer: letsencrypt-production
  labels:
    app: devtron
    release: devtron
  name: devtron-ingress
  namespace: devtroncd
spec:
  rules:
    - host: devtron.example.com
      http:
        paths:
          - backend:
              service:
                name: devtron-service
                port:
                  number: 80
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - devtron.example.com
      secretName: devtron-tls
```

## Uninstall Devtron 🗑️

To uninstall Devtron, run the following command:

```bash
helm uninstall devtron -n devtroncd
```

## Troubleshooting  🚑 

### Devtron pods are not running 🚫

If the Devtron pods are not running, check the logs of the pods to find the issue. You can use the following command to check the logs of the pods:

```bash
kubectl -n devtroncd logs <pod-name>
```

### Devtron pods are not running due to insufficient memory 🚫

If the Devtron pods are not running due to insufficient memory, you can increase the memory of the nodes by following the steps below:

1. Run the following command to get the name of the node:

```bash
kubectl get nodes
```

2. Run the following command to edit the node:

```bash
kubectl edit node <node-name>
```

3. Add the following lines under `spec:`:

```yaml
  kubelet:
    evictionHard:
      memory.available: "500Mi"
```

4. Save the file and exit.

### Devtron pods are not running due to insufficient CPU 🚫

If the Devtron pods are not running due to insufficient CPU, you can increase the CPU of the nodes by following the steps below:

1. Run the following command to get the name of the node:

```bash
kubectl get nodes
```

2. Run the following command to edit the node:

```bash
kubectl edit node <node-name>
```

3. Add the following lines under `spec:`:

```yaml
  kubelet:
    evictionHard:
      memory.available: "500Mi"
```

4. Save the file and exit.

## K3s Advanced Troubleshooting Guide 🚑

### Issue 1: K3s Server Not Starting 🚫

If your K3s server fails to start, follow these steps:

#### Check Logs 📋

```bash
journalctl -u k3s -xe
```

#### Review Configurations 🛠️

Ensure your configurations are correct and check for any typos or errors.

### Issue 2: Node Not Joining Cluster 🌐

If a node is unable to join the K3s cluster, troubleshoot as follows:

#### Check Node Token on Server 🗝️

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### Verify Node Connectivity 🌐

Ensure the node can reach the server and there are no network issues.

### Issue 3: Metallb Load Balancer Not Working ⚙️

If Metallb is not functioning correctly, follow these steps:

#### Check Metallb Pods 🚧

```bash
kubectl get pods -n metallb-system
```

#### Review Metallb Configurations 📝

Verify the ConfigMap settings and ensure they match your network setup.

### Issue 4: Nginx Ingress Controller Errors 🌐

If Nginx Ingress is causing issues, troubleshoot with these steps:

#### Inspect Ingress Controller Pods 🚧

```bash
kubectl get pods -n ingress-nginx
```

#### Check Ingress Resources 📋

Verify your Ingress resources for errors and correct configurations.

### Issue 5: Cert-Manager SSL Certificate Errors 🛡️

If SSL certificates are not working as expected, address the following:

#### Inspect Cert-Manager Pods 🚧

```bash
kubectl get pods -n cert-manager
```

#### Review Certificate Issuers 📝

Check ClusterIssuers and verify they are configured correctly.

### Issue 6: Devtron Installation Problems 🚧

If Devtron installation fails, troubleshoot as follows:

#### Check Devtron Operator Logs 📋

```bash
kubectl logs -n devtroncd deployment/devtron-operator
```

#### Review Helm Install Command 🛠️

Ensure Helm install commands are accurate and properly configured.

### Issue 7: Ingress Setup for Devtron Not Working 🌐

If Ingress setup for Devtron is problematic, consider the following:

#### Inspect Ingress YAML 📋

Review the Ingress YAML file and check for errors or misconfigurations.

#### Check Devtron Ingress Service Type ⚙️

Ensure the service type is correctly set in the Devtron Helm values.

Remember to always backup your configurations before making changes, and consult the official documentation for each component for more in-depth troubleshooting. Happy troubleshooting! 🛠️

## 📂🔒 K3s Backup and Restore Guide 📂🔒

### Backup K3s Cluster 📂

#### K3s Server State 📋

To back up the K3s server state, including configuration and data:

```bash
  sudo cp -r /etc/rancher/k3s/ /path/to/backup/folder
  sudo cp -r /var/lib/rancher/k3s/ /path/to/backup/folder
```

#### Kubeconfig File 🌐

Back up the kubeconfig file for accessing the cluster:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml /path/to/backup/folder
```

### Backup Persistent Volumes 🗄️

If your applications use persistent volumes, ensure you back them up:

1. Identify the Persistent Volumes (PVs) used by your applications.
2. Use your preferred method (snapshot, backup tool) to create backups of the PVs.

### Backup Etcd Data 🗃️

If you want to back up the etcd data:

```bash
sudo cp -r /var/lib/rancher/k3s/server/db/ /path/to/backup/folder
```

### Restore K3s Cluster 🔙

#### Restore Server State 🔄

To restore the K3s server state:

```bash
sudo cp -r /path/to/backup/folder/k3s/ /etc/rancher/
sudo cp -r /path/to/backup/folder/k3s/ /var/lib/rancher/
```

#### Restore Kubeconfig 🔄

Restore the kubeconfig file:

```bash
sudo cp /path/to/backup/folder/k3s.yaml /etc/rancher/k3s/
```

### Restore Persistent Volumes 🔄

Restore the Persistent Volumes using your backup tool or method.

### Restore Etcd Data 🔄

To restore etcd data:

```bash
sudo cp -r /path/to/backup/folder/db/ /var/lib/rancher/k3s/server/
```

# Conclusion

In the exhilarating voyage of deploying and managing a K3s Kubernetes cluster adorned with Metallb, Nginx Ingress, and Cert-Manager, we've navigated through the intricacies of container orchestration and fortified infrastructure resilience. Let's encapsulate our journey with the pivotal insights and achievements:

## Key Milestones 🏆

1. **Seamless K3s Installation:** The deployment of K3s on the server node unfolded effortlessly, courtesy of a user-friendly installation script. Configurations were tailored with ease, ensuring the cluster's alignment with specific requirements.

2. **Adaptable Cluster Expansion:** The addition of agent nodes seamlessly expanded the cluster's horizons. With a straightforward join command, new nodes seamlessly integrated, contributing to the cluster's scalability and robustness.

3. **Dynamic Network Balancing via Metallb:** Metallb introduced network load balancing prowess, streamlining the distribution of incoming traffic. Configuring Metallb was a smooth process, laying a robust foundation for our networking requirements.

4. **Efficient Ingress Routing with Nginx:** Nginx Ingress Controller empowered us to define and manage intricate ingress rules effortlessly. SSL termination, path-based routing, and other advanced features were within easy reach with Nginx's robust capabilities.

5. **Secure Certificate Management via Cert-Manager:** Cert-Manager simplified SSL certificate management with its automated processes. Secure communication within the cluster and external access through HTTPS were seamlessly achieved.

6. **Configurability with Custom Resources:** Harnessing the power of ConfigMaps and custom YAML configurations, we fine-tuned Metallb, Nginx Ingress, and Cert-Manager to cater to our specific use cases and preferences.

7. **Application Management with Devtron:** The incorporation of Devtron elevated our capabilities with a comprehensive continuous delivery and deployment solution. Helm charts and declarative configurations empowered us to manage applications efficiently.

## Valuable Insights Gained 🧠

1. **Backup and Restore Best Practices:** Regular backups, encompassing server state, kubeconfig files, persistent volumes, and etcd data, emerged as crucial practices for ensuring resilience and swift recovery.

2. **Troubleshooting Proficiency:** The troubleshooting guide equipped us with valuable insights into diagnosing and resolving issues related to K3s, Metallb, Nginx Ingress, Cert-Manager, and Devtron. Proactive troubleshooting is imperative for maintaining a healthy cluster.

3. **Security Prioritization:** Throughout our journey, we upheld security as a paramount concern, from server installations to SSL certificate management. Implementing best practices in security safeguards our cluster against potential threats.

## Continuation of the Expedition 🌐

As we celebrate the triumphant deployment and management of our K3s Kubernetes cluster, the odyssey continues. Continuous exploration of Kubernetes ecosystem tools, staying abreast of best practices, and embracing emerging technologies will further augment the cluster's capabilities.

In this ever-evolving realm of container orchestration, adaptability and a thirst for knowledge are pivotal. Here's to the success of our K3s cluster deployment, and may our future endeavors be even more riveting and rewarding! 🚀🌟

# 📚 References

During our journey of deploying and managing the K3s Kubernetes cluster with Metallb, Nginx Ingress, Cert-Manager, and Devtron, we drew insights from a mix of written documentation and engaging video content. Here's a nod to the resources that illuminated our path:

1. **K3s Documentation:** The official documentation of K3s provided comprehensive guidance on installation, configuration, and troubleshooting. [K3s Documentation](https://rancher.com/docs/k3s/)

2. **Metallb Documentation:** Metallb's official documentation proved invaluable in understanding and implementing network load balancing within our cluster. [Metallb Documentation](https://metallb.universe.tf/)

3. **Nginx Ingress Controller Documentation:** The official documentation of Nginx Ingress Controller was our go-to resource for configuring and managing ingress resources. [Nginx Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)

4. **Cert-Manager Documentation:** Cert-Manager's documentation guided us through the intricacies of managing SSL certificates within our Kubernetes cluster. [Cert-Manager Documentation](https://cert-manager.io/docs/)

5. **Devtron Documentation:** The official documentation of Devtron provided insights into installing, configuring, and using the Devtron platform for continuous delivery and deployment. [Devtron Documentation](https://docs.devtron.ai/)

6. **YouTube Videos:**
   - [K3s Installation Tutorial by Kubernetes Academy](https://www.youtube.com/watch?v=OZSuDvYaQac&t=34s&pp=ygUJazNzIHNldHVw)
   - [Metallb Load Balancer Setup by Techworld with Nana](https://www.youtube.com/watch?v=1hwGdey7iUU)
   - [Nginx Ingress Controller Explained by Techworld with Nana](https://www.youtube.com/watch?v=deLW2h1RGz0&t)
   - [Cert-Manager Tutorial by Techworld with Nana](https://www.youtube.com/watch?v=DvXkD0f-lhY)
   - [Devtron Installation Walkthrough by Devtron Labs](https://www.youtube.com/watch?v=QDwhbMvikGQ)

These references, both textual and visual, served as pillars of knowledge, offering clarity and expertise at every step of our Kubernetes journey. Huge thanks to the creators and contributors for sharing their insights through documentation and engaging video content.
