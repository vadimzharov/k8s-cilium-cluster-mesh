Playbooks to install simple one node k8s cluster as VM using KVM running on Linux OS (tested on Fedora).

Playbooks use CentOS 7 as base OS and download it as QCOW2 image from the official repo. After downloading playbook injects user's (from `$HOME/.ssh/id_rsa.pub) ssh-key into the image and creates VM based on the image.

After VM created it configures operating system and installs all required packages - to execute kubeadm command to initialize k8s cluster.

As a final step of cluster deployment playbook applies a manifest to install Flannel as CNI.
Also, it creates record for `k8s-lab.mypc.loc` hostname in `/etc/hosts` file on local machine so cluster is available using `k8s-lab.mypc.loc` hostname.

After cluster is initialized separate `k8s-config` role applies to the cluster:
* Local manifests from `files/manifests` subdirectories based on `k8s_config_post_install_local_manifests` variable in `group_vars/all/all.yaml`. As of now it has:
    * Manifests to install nginx ingress controller. After nginx deployed, ingresses are available via ports 80 and 443 on `k8s-lab.mypc.loc` hostname.
    * Manifests to install Tekton. After Tekton deployed, UI is available on `http://k8s-lab.mypc.loc/tekton` URL.
* Remote manifests (from remote HTTP servers) based on `k8s_config_post_install_remote_manifests` variable in `group_vars/all/all.yaml`.
  Now it has only one manifest to deploy k8s demo app (https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/).

Kubeconfig file is available on the VM at `/etc/kubernetes/admin.conf`, but also playbook fetches it as `kubeconfig` file in the playbook directory.
To work with the cluster after install execute from playbook directory:
```
$ export KUBECONFIG=kubeconfig
$ kubectl get pods -A
```

Local computer MUST have KVM/libvirt installed (with virsh tool) and running and also have `qemu-img` and `virt-sysprep` tools installed (to manipulate with CentOS image).

To execute playbook run:
```
ansible-playbook setup-k8s.yaml -i inventory/
```

Playbook is idempotent and will not destroy/recreate VM if re-executed.

VM parameters are in `roles/k8s-vm/defaults/main.yaml` - memory and CPU settings can be changed based on variables listed.

If required to reprovision the VM - execute:
```
ansible-playbook setup-k8s.yaml -i inventory/ -e k8s_vm_reprovision=true
```

If you need only to apply manifests based on vars `k8s_config_post_install_local_manifests` and `k8s_config_post_install_remote_manifests` run:
```
ansible-playbook config-k8s.yaml -i inventory/
```