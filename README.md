## Testing Cilium Cluster Mesh installation/configuration using Helm Charts (GitOps approach) - part 1.

[Cilium Cluster Mesh](https://cilium.io/use-cases/cluster-mesh/) allows administrators to connect multiple Kubernetes clusters into one mesh network - where all pod networks are interconnected. Pods from one cluster can communicate with pods or Kubernetes services in any other cluster using IP addresses or DNS name. This can be useful in various scenarios - for instance when an application runs on one Kubernetes cluster but the database required for the application to work runs in another Kubernetes cluster. Or if the application is spread across multiple Kubernetes clusters and administrators want to balance traffic from (one or many) Kubernetes ingresses across all available application pods within these clusters.

I use Cilium CNI in my home lab Kubernetes clusters for a long time, and recently implemented Cilium Cluster Mesh to be able to distribute applications between my clusters and move them from one cluster to another cluster (almost) without service interruption if I need to re-install, maintain or upgrade one of my Kubernetes clusters.

To manage my Kubernetes clusters I use the GitOps approach by utilizing ArgoCD to deploy and configure everything using ArgoCD ApplicationSets/Applications, where I define Git repository with Kubernetes manifests or Helm charts with the required values to deploy an application - and the same approach I want to use to deploy and configure Cilium CNI/Cilium Mesh.

Until not too long ago, configuring Cilium Cluster Mesh involved manual steps that required administrators to do some manual steps (or by using bash scripts) to create and copy certificates needed for Cluster Mesh to establish connections between Kubernetes clusters. But starting from Cilium version 1.14.0, the Cilium Helm chart provides administrators with the possibility to configure Cluster Mesh using the GitOps approach - just by installing the Cilium Helm chart with proper values, and Cluster Mesh becomes operational immidiately.

In this article I will outline the steps how to deploy and configure Kubernetes cluster(s) interconnected with Cilium Cluster Mesh using Cilium Helm Chart (version 1.14.2).

### Pre-requisites

Detailed description can be found in the [official documentation.](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/#prerequisites)

Noticeable requirements are:
* PODS and SERVICES CIDR ranges must be **unique** across **all** Kubernetes clusters (to ensure pods can communicate with each other by IP address)
* Nodes CIDR must also be unique, and nodes **must** have IP connectivity between each other (In my environment I use a direct connection, but per Cilium documentation - VPN is also possible)
* Cilium pods must connect to the Clustermesh API service exposed on each interconnected cluster. This service can be exposed as NodePort (though not recommended for production; however I use it in my test environment) or Kubernetes ingress.
* **To configure Cilium Cluster Mesh using Helm chart without any additional manual steps - Cilium CNI on all Kubernetes clusters must be configured by using the same Certificate Authority (CA)**. This setup allows clusters to trust each other, eliminating the need to manually create or distribute certificates from one cluster to another. You can specify this configuration in the Helm Chart values file to enable Cilium Cluster Mesh.

### A brief description of the steps

To test this feature, I perform the following steps:
* Create at least two Kubernetes clusters on my KVM environment (automated via Ansible), without configuring any CNI - this step is not necessary if you already have Kubernetes clusters.
* Configure Cilium CNI using Helm on each cluster without Cluster Mesh to ensure everything is working (using Helm Chart and values files) - this step is not necessary if you already have Kubernetes clusters with Cilium CNI configured - but ensure you meet the pre-requisites.
* Update Cilium CNI configuration using Helm chart with the same values file but with Cluster Mesh configuration added
* Make validation by checking if pods from different clusters can connect to each other.
* Deploy ingress controller (nginx), create Kubernetes ingresses, and balance ingress traffic across Kubernetes clusters.

### Configuring Cilium Cluster Mesh using HELM

#### Creating Kubernetes clusters

I created a simple Ansible role to create a Kubernetes cluster on my local KVM host. If you already have Kubernetes clusters or want to deploy Kubernetes clusters by yourself - just skip this step, **however, ensure K8S clusters meet the pre-requisites (have unique POD CIDR ranges, etc..)**.

The playbook works only for KVM and was tested only on Fedora 38.

The playbook downloads the official Ubuntu Jammy 22.04 KVM image, creates a VM based on this image, and then installs/configures the Kubernetes cluster using kubeadm. Since we need two Kubernetes clusters, this playbook needs to be executed twice.

Before executing the playbook, check [group_vars/all/all.yaml](group_vars/all/all.yaml) file and check inventory files for both Kubernetes clusters [inventory/k8s-01.yaml](inventory/k8s-01.yaml) and [inventory/k8s-02.yaml](inventory/k8s-02.yaml) - notice they have different POD and service CIDRs, and also different NodePort values (now there is a bug in Cilium and Clustermesh API cannot be exposed on the same port on different interconnected Kubernetes clusters).

To install/configure the first Kubernetes cluster, execute:
```
ansible-playbook 01-install-k8s.yaml -i inventory/k8s-01.yaml
```

After the playbook completes, kubeconfig for the K8S01 cluster is available in the playbook directory as `kubeconfig-k8s01`:
```
[vadim@fedora k8s-kvm-lab]$ export KUBECONFIG=kubeconfig-k8s01
[vadim@fedora k8s-kvm-lab]$ kubectl get pods -A
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-5dd5756b68-l5x9p                      0/1     Running   0          4m13s
kube-system   coredns-5dd5756b68-w92ft                      0/1     Running   0          4m13s
kube-system   etcd-k8s-lab-01.mypc.loc                      1/1     Running   0          4m27s
kube-system   kube-apiserver-k8s-lab-01.mypc.loc            1/1     Running   0          4m27s
kube-system   kube-controller-manager-k8s-lab-01.mypc.loc   1/1     Running   0          4m30s
kube-system   kube-scheduler-k8s-lab-01.mypc.loc            1/1     Running   0          4m30s
[vadim@fedora k8s-kvm-lab]$ 
```

Kubernetes cluster installed, but no CNI is configured.

Create the second Kubernetes cluster:
```
ansible-playbook 01-install-k8s.yaml -i inventory/k8s-02.yaml
```

#### Installing Cilium CNI using Helm

The next step is to install Cilium CNI but without Cluster Mesh configuration - to see on the next step what changes after you enable Cluster Mesh. The crucial part of this step is to use the same CA certificate for Cilium on both clusters.
Another ansible playbook can generate CA, create values file, and install Cilium.

The Jinja2 template I use to generate Cilium values is in the [files/cilium/values.yaml.j2](files/cilium/values.yaml.j2) file.

To just generate a Cilium values file for both clusters, execute:
```
ansible-playbook 02-generate-cilium-config.yaml -i inventory/k8s-01.yaml
```

and 
```
ansible-playbook 02-generate-cilium-config.yaml -i inventory/k8s-02.yaml
```

The output values files along with the generated CA are available in [files/tmp](files/tmp/) directory. The content of the files is common for any Cilium CNI configuration, but all clusters share the same CA.

You can install Cilium CNI on your Kubernetes clusters with these files by executing: 
```
helm upgrade --install cilium cilium/cilium --version {{cilium_version}} --namespace kube-system --values {{playbook_dir}}/files/tmp/cilium-values-{{ cluster_name }}.yaml
```

And I have the playbook to do this on both clusters:

```
ansible-playbook 03-install-k8s-cilium.yaml -i inventory/k8s-01.yaml
```

and 
```
ansible-playbook 03-install-k8s-cilium.yaml -i inventory/k8s-02.yaml
```

Ensure Cilium CNI is configured and all pods are in Ready state:

On the first Kubernetes cluster:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl get pods -A -o wide
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE     IP              NODE                  NOMINATED NODE   READINESS GATES
kube-system   cilium-9lxcw                                  1/1     Running   0          2m22s   192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
kube-system   cilium-operator-86cdbc694f-tkwwg              1/1     Running   0          2m22s   192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
kube-system   coredns-5dd5756b68-fg4dk                      1/1     Running   0          116s    10.244.0.118    k8s-lab-01.mypc.loc   <none>           <none>
kube-system   coredns-5dd5756b68-gm5qj                      1/1     Running   0          101s    10.244.0.72     k8s-lab-01.mypc.loc   <none>           <none>
kube-system   etcd-k8s-lab-01.mypc.loc                      1/1     Running   0          39m     192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
kube-system   hubble-relay-6cc7d455f7-kl8px                 1/1     Running   0          2m22s   10.85.0.4       k8s-lab-01.mypc.loc   <none>           <none>
kube-system   hubble-ui-6cc5887659-9x7t5                    2/2     Running   0          2m22s   10.85.0.5       k8s-lab-01.mypc.loc   <none>           <none>
kube-system   kube-apiserver-k8s-lab-01.mypc.loc            1/1     Running   0          39m     192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
kube-system   kube-controller-manager-k8s-lab-01.mypc.loc   1/1     Running   0          39m     192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
kube-system   kube-scheduler-k8s-lab-01.mypc.loc            1/1     Running   0          39m     192.168.124.6   k8s-lab-01.mypc.loc   <none>           <none>
```

On the second Kubernetes cluster:

```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl get pods -A -o wide
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE     IP               NODE                  NOMINATED NODE   READINESS GATES
kube-system   cilium-operator-99f8bd7dd-wnh2v               1/1     Running   0          4m1s    192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
kube-system   cilium-vgl5l                                  1/1     Running   0          4m1s    192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
kube-system   coredns-5dd5756b68-d4czb                      1/1     Running   0          3m30s   10.245.0.157     k8s-lab-02.mypc.loc   <none>           <none>
kube-system   coredns-5dd5756b68-rzjcj                      1/1     Running   0          3m45s   10.245.0.242     k8s-lab-02.mypc.loc   <none>           <none>
kube-system   etcd-k8s-lab-02.mypc.loc                      1/1     Running   0          30m     192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
kube-system   hubble-relay-6cc7d455f7-77fkd                 1/1     Running   0          4m1s    10.85.0.5        k8s-lab-02.mypc.loc   <none>           <none>
kube-system   hubble-ui-6cc5887659-lzvdg                    2/2     Running   0          4m1s    10.85.0.4        k8s-lab-02.mypc.loc   <none>           <none>
kube-system   kube-apiserver-k8s-lab-02.mypc.loc            1/1     Running   0          30m     192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
kube-system   kube-controller-manager-k8s-lab-02.mypc.loc   1/1     Running   0          30m     192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
kube-system   kube-scheduler-k8s-lab-02.mypc.loc            1/1     Running   0          30m     192.168.124.20   k8s-lab-02.mypc.loc   <none>           <none>
```

Now I have two Kubernetes clusters with Cilium CNI configured, these clusters have unique POD and Service CIDRs, and nodes can reach each other by IP. Also, the playbook created two records in `/etc/hosts` file on my local machine, so both nodes are available as `k8s-lab-01.mypc.loc` and `k8s-lab-02.mypc.loc` - I can expose ClusterMesh API on these nodes and use these hostnames in Cluster Mesh API server configuration.

#### Configuring Cilium CNI Cluster Mesh using Helm

The next step is to update the existing Cilium CNI configuration with a values file containing the Cluster Mesh configuration.

Generate updated values file, for the first cluster:
```
ansible-playbook 04-generate-cilium-mesh-config.yaml -i inventory/k8s-01.yaml
```

and for the second:
```
ansible-playbook 04-generate-cilium-mesh-config.yaml -i inventory/k8s-02.yaml
```
The output values files are in [files/tmp](files/tmp/) directory, named as `cilium-values-k8s01-mesh.yaml` and `cilium-values-k8s02-mesh.yaml`, and contain updated Cilium CNI configuration with Cluster Mesh enabled, here is this part with some comments:
```
cluster:
  id: 01                               # <= this is cluster ID and must be unique for each cluster
  name: k8s01                          # <= this is cluster name and must be uniqie for each cluster
clustermesh:
  apiserver:
    service:
      type: NodePort                   # ClusterMesh API is exposed using NodePort (in my case - for simplicity). All Cilium pods from all other clusters
      nodePort: 32379                  # will try to reach this endpoint based on settings in {clustermesh.config.clusters[].address/port}
    tls:
      authMode: cluster                # authMode if all clusters sharing the same CA
      server:
        extraDnsNames:
          - *.mypc.loc                 # To properly generate certificate you MUST specify your domain here, otherwise you will get "authentication handshake failed"
  config:                              # since all certificates are generated for domain *mesh.cilium.io by default.
    enabled: true
    domain: mypc.loc                   # Domain must be the same for all clusters
    clusters:                          # List with all clusters connected to Cluster Mesh
      - name: k8s02                    # Cluster name of the second cluster
        address: k8s-lab-02.mypc.loc   # Hostname with clustermesh API service exposed
        port: 32380                    # Port for Clustermesh API. So cilium agents on k8s01 cluster should be able to curl https://k8s-lab-02.mypc.loc:32380
  useAPIServer: true                   # Enabling Clustermesh API.
```

You can update the Cilium CNI configuration by executing: 
```
helm upgrade cilium cilium/cilium --version {{cilium_version}} --namespace kube-system --values files/tmp/cilium-values-{{ cluster_name }}-mesh.yaml
```
for each cluster, but I have playbooks to do the same, for the first:
```
ansible-playbook 05-configure-cilium-mesh.yaml -i inventory/k8s-01.yaml
```
and for the second:
```
ansible-playbook 05-configure-cilium-mesh.yaml -i inventory/k8s-02.yaml
```

After you update the Cilium configuration with new values file or playbooks finished, restart all Cilium pods on both clusters so they pick a new configuration with new certificates (secret `cilium-clustermesh` is now mounted to all cilium agent pods):
```
KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system delete pods -l app.kubernetes.io/name=cilium-agent
pod "cilium-2phq5" deleted
KUBECONFIG=kubeconfig-k8s02 kubectl -n kube-system delete pods -l app.kubernetes.io/name=cilium-agent
pod "cilium-b5kcf" deleted
```

#### Validating Cilium CNI Cluster Mesh configuration

After `helm upgrade` / ansible playbooks completed, both clusters should have Cilium Cluster Mesh configured. Let's see what changed.

Now clusters have clustermesh-apiserver pod running and exposed as NodePort service:

Pods on both clusters:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl get pods -n kube-system -o wide -l app.kubernetes.io/name=clustermesh-apiserver
NAME                                     READY   STATUS    RESTARTS   AGE   IP             NODE                  NOMINATED NODE   READINESS GATES
clustermesh-apiserver-5759c4ffff-xhrcz   2/2     Running   0          18m   10.244.0.184   k8s-lab-01.mypc.loc   <none>           <none>
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl get pods -n kube-system -o wide -l app.kubernetes.io/name=clustermesh-apiserver
NAME                                     READY   STATUS    RESTARTS   AGE     IP             NODE                  NOMINATED NODE   READINESS GATES
clustermesh-apiserver-55d558985c-f72rh   2/2     Running   0          5m38s   10.245.0.127   k8s-lab-02.mypc.loc   <none>           <none>
```

Services on both clusters, notice different NodePorts (32379 and 32380):
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl get svc -n kube-system -o wide -l app.kubernetes.io/name=clustermesh-apiserver
NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE   SELECTOR
clustermesh-apiserver           NodePort    10.96.44.129   <none>        2379:32379/TCP      19m   k8s-app=clustermesh-apiserver
clustermesh-apiserver-metrics   ClusterIP   None           <none>        9962/TCP,9964/TCP   19m   k8s-app=clustermesh-apiserver
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl get svc -n kube-system -o wide -l app.kubernetes.io/name=clustermesh-apiserver
NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)             AGE     SELECTOR
clustermesh-apiserver           NodePort    10.97.21.70   <none>        2379:32380/TCP      6m17s   k8s-app=clustermesh-apiserver
clustermesh-apiserver-metrics   ClusterIP   None          <none>        9962/TCP,9964/TCP   6m17s   k8s-app=clustermesh-apiserver
```

There are clustermesh secrets generated:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system get secrets | grep clustermesh
cilium-clustermesh                  Opaque                          1      20m
clustermesh-apiserver-admin-cert    kubernetes.io/tls               3      20m
clustermesh-apiserver-ca-cert       Opaque                          2      20m
clustermesh-apiserver-remote-cert   kubernetes.io/tls               3      20m
clustermesh-apiserver-server-cert   kubernetes.io/tls               3      20m
```

All clustermesh-apiserver-*certs* secrets are certificates for Clustermesh API service, generated using CA from secret `clustermesh-apiserver-ca-cert` which is actually the same as `cilium-ca` and is the same for all clusters in Cluster Mesh - it was provided in Cilium Helm values files.

Secret `cilium-clustermesh` has a list of all remote clusters that are part of Cluster Mesh, except the current cluster, for example - for **k8s01** cluster:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system get secret cilium-clustermesh -o jsonpath='{.data.k8s02}' | base64 -d
endpoints:
- https://k8s-lab-02.mypc.loc:32380
trusted-ca-file: /var/lib/cilium/clustermesh/common-etcd-client-ca.crt
key-file: /var/lib/cilium/clustermesh/common-etcd-client.key
cert-file: /var/lib/cilium/clustermesh/common-etcd-client.crt
```
It contains endpoint(s) definition (clustermesh API for k8s02 cluster) and certificates to use to connect to this endpoint. This secret is mounted to all Cilium agent pods.

Cluster Mesh status can be checked within any cilium agent pod (since pod is connected to Cluster Mesh). Execute `cilium status` command from any cilium agent pod:

Get pod name:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system get pods -l app.kubernetes.io/name=cilium-agent
NAME           READY   STATUS    RESTARTS   AGE
cilium-g2dxd   1/1     Running   0          4m28s
```
Execute `cilium status` from the pod and notice ClusterMesh status:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system get pods -l app.kubernetes.io/name=cilium-agent
NAME           READY   STATUS    RESTARTS   AGE
cilium-g2dxd   1/1     Running   0          4m28s
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system exec -c cilium-agent cilium-g2dxd -- cilium status
KVStore:                 Ok   Disabled
Kubernetes:              Ok   1.28 (v1.28.3) [linux/amd64]
Kubernetes APIs:         ["EndpointSliceOrEndpoint", "cilium/v2::CiliumClusterwideNetworkPolicy", "cilium/v2::CiliumEndpoint", "cilium/v2::CiliumNetworkPolicy", "cilium/v2::CiliumNode", "cilium/v2alpha1::CiliumCIDRGroup", "core/v1::Namespace", "core/v1::Pods", "core/v1::Service", "networking.k8s.io/v1::NetworkPolicy"]
.............
ClusterMesh:             1/1 clusters ready, 0 global-services
............
```

Also, now any agent can see all other nodes from all other clusters, execute `cilium node list`:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n kube-system exec -c cilium-agent cilium-g2dxd -- cilium node list
Name                            IPv4 Address      Endpoint CIDR   IPv6 Address   Endpoint CIDR
k8s01/k8s-lab-01.mylaptop.loc   192.168.125.208   10.244.0.0/24                  
k8s02/k8s-lab-02.mylaptop.loc   192.168.125.197   10.245.0.0/24    
```

## Testing Cilium Cluster Mesh installation/configuration using Helm Charts (GitOps approach) - part 2.

In the previous steps, I deployed two Kubernetes clusters interconnected by using Cilium Cluster Mesh and validated the configuration using Cilium CLI. Now let's test different scenarios how to use Cilium Cluster Mesh.

#### Testing Cilium Cluster Mesh - pods interconnection.

After Cilium Cluster Mesh is working, we can deploy pods and check if they can reach each other. To do this I use nginx example pod deployed in `cluster-mesh-test` namespace.

Create namespace and deploy pod in both clusters using manfiests in [files/manifests/pod](files/manifests/pod/) directory:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl apply -f files/manifests/pod/
namespace/cluster-mesh-test created
pod/static-web created
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl apply -f files/manifests/pod/
namespace/cluster-mesh-test created
pod/static-web created
```

Check pods status and IPs on both Kubernetes clusters:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl get pods -n cluster-mesh-test -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP             NODE                      NOMINATED NODE   READINESS GATES
static-web   1/1     Running   0          59s   10.244.0.175   k8s-lab-01.mylaptop.loc   <none>           <none>
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl get pods -n cluster-mesh-test -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP             NODE                      NOMINATED NODE   READINESS GATES
static-web   1/1     Running   0          57s   10.245.0.143   k8s-lab-02.mylaptop.loc   <none>           <none>
```

Try to curl from one pod to another, first cluster - it is responding with code 200:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n cluster-mesh-test exec static-web -- curl -s -I http://10.245.0.143:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   615    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.25.3
Date: Tue, 31 Oct 2023 04:05:46 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 24 Oct 2023 13:46:47 GMT
Connection: keep-alive
ETag: "6537cac7-267"
Accept-Ranges: bytes
```

and second cluster:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl -n cluster-mesh-test exec static-web -- curl -s -I http://10.244.0.175:80
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0   615    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
HTTP/1.1 200 OK
Server: nginx/1.25.3
Date: Tue, 31 Oct 2023 04:06:52 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 24 Oct 2023 13:46:47 GMT
Connection: keep-alive
ETag: "6537cac7-267"
Accept-Ranges: bytes
```

Pods from one cluster can reach pod in another cluster by using it's endpoint IP.

#### Testing Cilium Cluster Mesh - using shared services.

Traffic balancing across all interconnected Kubernetes clusters is achieved by using Kubernetes services with the same name in the same namespace and by using annotation `service.cilium.io/global: "true"` applied to the service.

To test this, create a separate namespace `shared-service` with sample nginx deployment and service with the annotation mentioned above. All manifests are in [files/manifests/shared-svc](files/manifests/shared-svc) directory.

Create namespace, deployment, and service in both clusters by using manifests in [files/manifests/shared-svc](files/manifests/shared-svc/) directory:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl apply -f files/manifests/shared-svc/
namespace/shared-service created
deployment.apps/nginx-deployment created
service/shared-service created
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl apply -f files/manifests/shared-svc/
namespace/shared-service created
deployment.apps/nginx-deployment created
service/shared-service created
```

The service has been created on both clusters:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get svc shared-service -o wide
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
shared-service   ClusterIP   10.96.7.69   <none>        80/TCP    69s   app.kubernetes.io/name=nginx
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s02 kubectl -n shared-service get svc shared-service -o wide
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
shared-service   ClusterIP   10.97.47.5   <none>        80/TCP    74s   app.kubernetes.io/name=nginx
```
Both services on Kubernetes clusters have the same name, and same namespace - which means it is available by using the hostname `shared-service.shared-service.svc.cluster.local`, and since services are annotated with `service.cilium.io/global: "true"` - it balances traffic across Kubernetes clusters. Let's validate this.

On the FIRST Kubernetes, from pod `static-web` in namespace `cluster-mesh-test` created on the previous step curl this shared service:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n cluster-mesh-test exec static-web -- curl -s -I http://shared-service.shared-service.svc.cluster.local
HTTP/1.1 200 OK
Server: nginx/1.14.2
```

Now traffic is balancing across both Kubernetes clusters - since they all have pods serving this service. Let's scale deployment to 0 on the FIRST K8S cluster and try the same command again.

Scale deployment down and ensure no pods are running:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service scale deployment/nginx-deployment --replicas=0
deployment.apps/nginx-deployment scaled
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get pods 
No resources found in shared-service namespace.
```

But the same command is still working from the FIRST Kubernetes cluster, even if no pods are running (and deployment can be completely deleted) on the FIRST Kubernetes cluster, and traffic goes from the first Kubernetes cluster to the second Kubernetes cluster without any user intervention using shared Kubernetes service:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n cluster-mesh-test exec static-web -- curl -s -I http://shared-service.shared-service.svc.cluster.local
HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Tue, 31 Oct 2023 14:38:23 GMT
```

Scale the deployment back:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service scale deployment/nginx-deployment --replicas=1
deployment.apps/nginx-deployment scaled
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get pods 
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86dcfdf4c6-pcrsd   1/1     Running   0          4s
```

#### Testing Cilium Cluster Mesh - using shared services with Nginx ingress

Now let's explore the more advanced scenario - configuring the ingress controller to balance ingress traffic across all available pods across all Kubernetes clusters interconnected with Cilium Cluster Mesh.

Before configuring ingress with shared services, it is important to understand that by default all (as far as I know) Kubernetes ingress controllers use list of Kubernetes endpoints (POD IPs) to balance traffic from ingress pods to application pods in Kubernetes cluster - means **traffic goes directly from Ingress pods to Application pods bypassing Kubernetes service IP/hostname** - to increase performance and remove potential bottlenecks. However, this is not possible if we use shared services - because if deployment on the local Kubernetes cluster is scaled down - then the service - even if it is configured as shared - has no pods on the local Kubernetes cluster - and has no endpoints. Hence Ingress controller doesn't know where to send traffic (for Ingress it looks like the application doesn't have any pods running) and the application is not available via Kubernetes ingress. To configure ingress traffic balancing across Kubernetes clusters interconnected into Cilium Cluster Mesh, we need to define for ingress to balance traffic based on **service IP**, not based on **endpoints IP**. Not all ingress controllers support that, and I migrated from Traefik to Nginx because of this - Traefik can work only with endpoints, but the Nginx ingress controller has special [annotation](https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/nginx-configuration/annotations.md#service-upstream) for Ingress Kubernetes object `nginx.ingress.kubernetes.io/service-upstream`. If this annotation is specified for the Ingress object - then the Ingress controller will balance traffic based on service IP, not based on endpoint IP - and will balance traffic across multiple Kubernetes clusters.

To test this we can use the same service created in the previous step, but exposed via the Nginx ingress controller.

First, deploy the Nginx controller on the first Kubernetes cluster using default manifests:
```
KUBECONFIG=kubeconfig-k8s01 kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```
and expose it using port 80 on Kubernetes nodes (don't do this in production), by applying patch available in [files/manifests/ingress-nginx](files/manifests/ingress-nginx/) directory:
```
KUBECONFIG=kubeconfig-k8s01 kubectl -n ingress-nginx patch deployment ingress-nginx-controller --patch-file files/manifests/ingress-nginx/nginx.yaml
```

Now nginx controller is available and responding on port 80:
```
[vadim@fedora k8s-kvm-lab]$ curl k8s-lab-01.mylaptop.loc
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

To expose existing service `shared-service` from namespace `shared-service` create ingress Kubernetes resource without providing any additional annotations by applying manifest from [files/manifests/shared-ingress](files/manifests/shared-ingress/) directory:

```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl apply -f files/manifests/shared-ingress/
ingress.networking.k8s.io/ingress created
```

Check if ingress created:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get ingress
NAME      CLASS   HOSTS                  ADDRESS   PORTS   AGE
ingress   nginx   ingress.mylaptop.loc             80      19s
```

and available - since there are pods running on both clusters, including the first cluster, ingress should work:

Get the node's IP:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl get nodes -o wide
NAME                      STATUS   ROLES           AGE   VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-lab-01.mylaptop.loc   Ready    control-plane   12h   v1.28.1   192.168.125.208   <none>        Ubuntu 22.04.3 LTS   5.15.0-86-generic   cri-o://1.28.1
```

Run curl command if nginx application is working (should get code 200):
```
[vadim@fedora k8s-kvm-lab]$ curl -I -s --resolve ingress.mylaptop.loc:80:192.168.125.208 ingress.mylaptop.loc:80
HTTP/1.1 200 OK
```

This ingress does not have the annotation `nginx.ingress.kubernetes.io/service-upstream=true`, and traffic is balanced from the Ingress Controller using endpoint IP (on local, first Kubernetes cluster). This can be validated in the nginx controller pod:
```
[vadim@fedora k8s-kvm-lab]$ NGINX_POD=$(KUBECONFIG=kubeconfig-k8s01 kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n ingress-nginx logs $NGINX_POD --tail=3
I1031 18:34:53.359293       7 controller.go:207] "Backend successfully reloaded"
I1031 18:34:53.359534       7 event.go:285] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-7f8dd58d99-bd6wg", UID:"35fdc5aa-1f4a-4aaf-8286-62bbbb746f9d", APIVersion:"v1", ResourceVersion:"78113", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration
192.168.125.1 - - [31/Oct/2023:18:37:16 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.0.1" 84 0.000 [shared-service-shared-service-80] [] 10.244.0.78:80 0 0.000 200 28f980c3dd7e66f9e72a3f8eae99feca
```

where IP 10.244.0.78 is IP of nginx application:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get endpoints
NAME             ENDPOINTS        AGE
shared-service   10.244.0.78:80   4h28m
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE                      NOMINATED NODE   READINESS GATES
nginx-deployment-86dcfdf4c6-7j4f5   1/1     Running   0          29m   10.244.0.78   k8s-lab-01.mylaptop.loc   <none>           <none>
```

So without `nginx.ingress.kubernetes.io/service-upstream=true` nginx controller is balancing traffic only to local pods because the local K8S controller doesn't provide information about pods/IPs from all other clusters interconnected with Cilium Cluster Mesh. This is also can be validated if local deployment is scaled down - the ingress will respond with error 503:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service scale deployment/nginx-deployment --replicas=0
deployment.apps/nginx-deployment scaled

[vadim@fedora k8s-kvm-lab]$ curl -I -s --resolve ingress.mylaptop.loc:80:192.168.125.208 ingress.mylaptop.loc:80
HTTP/1.1 503 Service Temporarily Unavailable
```
And saying there are no active endpoints for the service in the Nginx controller logs:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n ingress-nginx logs $NGINX_POD --tail=3
W1031 18:55:30.205839       7 controller.go:1207] Service "shared-service/shared-service" does not have any active Endpoint.
W1031 18:55:33.539328       7 controller.go:1207] Service "shared-service/shared-service" does not have any active Endpoint.
192.168.125.1 - - [31/Oct/2023:18:55:47 +0000] "HEAD / HTTP/1.1" 503 0 "-" "curl/8.0.1" 84 0.000 [shared-service-shared-service-80] [] - - - - 744af87a0efb2268445099ab57b91573
```

Now let's scale back the deployment and annotate the Ingress with `kubectl -n shared-service annotate ingress ingress nginx.ingress.kubernetes.io/service-upstream=true` command:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service scale deployment/nginx-deployment --replicas=1
deployment.apps/nginx-deployment scaled
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service annotate ingress ingress nginx.ingress.kubernetes.io/service-upstream=true
ingress.networking.k8s.io/ingress annotated
```

Now the curl command is working back:
```
[vadim@fedora k8s-kvm-lab]$ curl -I -s --resolve ingress.mylaptop.loc:80:192.168.125.208 ingress.mylaptop.loc:80
HTTP/1.1 200 OK
```
But the nginx controller is balancing traffic based on service IP, not based on pod IP:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get svc
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
shared-service   ClusterIP   10.96.7.69   <none>        80/TCP    4h34m
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n ingress-nginx logs $NGINX_POD --tail=3
I1031 18:58:42.014562       7 main.go:110] "successfully validated configuration, accepting" ingress="shared-service/ingress"
I1031 18:58:42.022771       7 event.go:285] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"shared-service", Name:"ingress", UID:"699830ad-a263-42c8-888a-bd28ce8d9fc8", APIVersion:"networking.k8s.io/v1", ResourceVersion:"101339", FieldPath:""}): type: 'Normal' reason: 'Sync' Scheduled for sync
192.168.125.1 - - [31/Oct/2023:18:58:59 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.0.1" 84 0.002 [shared-service-shared-service-80] [] 10.96.7.69:80 0 0.002 200 9519f0b0beabf04d97e32d2f2ad52453
```

The IP you see in the logs - 10.96.7.69 - is the IP of `shared-service` service, and not the nginx application pod. So now, if deployment on the first K8S cluster is scaled to 0, the traffic goes to the second K8S cluster:
```
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service scale deployment/nginx-deployment --replicas=0
deployment.apps/nginx-deployment scaled
[vadim@fedora k8s-kvm-lab]$ KUBECONFIG=kubeconfig-k8s01 kubectl -n shared-service get pods
No resources found in shared-service namespace.
[vadim@fedora k8s-kvm-lab]$ curl -I -s --resolve ingress.mylaptop.loc:80:192.168.125.208 ingress.mylaptop.loc:80
HTTP/1.1 200 OK
```

### Conclusion

Cilium Cluster Mesh offers a powerful solution to route pod-to-pod and pod-to-service traffic between multiple Kubernetes clusters (if your clusters and infrastructure meet requirements). The way it is working and configuring is pretty easy and straightforward - because basically it creates L3 routing between POD/SERVICEs CIDRs, and in some cases can be a great option if you want to share resources/applications across some/all your Kubernetes clusters.
