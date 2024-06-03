![K8S 10 Years](https://www.linuxfoundation.org/hubfs/hbd_k8s-1.svg)

This tutorial will allow you to run Kubernetes v1.0 - in your browser by making use of the Free Google Cloud Shell tier that can be used without enrollment, you're using it now! 🚀

>Credits to [Carlos Santana](https://github.com/csantanapr), [Amim Moises Salum Knabben](https://github.com/knabben) & [James Spurin](https://github.com/spurin)

Carlos Santana kicked off the fun in the CNCF Ambassador chat room, sparking a brilliant idea to celebrate Kubernetes' 10-year anniversary! Amim dived deep, tackling key challenges and identifying core issues. Meanwhile, James Spurin whipped up some nifty workarounds. Together, they've crafted this engaging tutorial. Dive in, have fun, and join us in cheering for a fantastic decade of Kubernetes! 🎉

Kubernetes was open sourced on June 6th, 2014, the first version with stable API [v1.0.0](https://github.com/kubernetes/kubernetes/releases/tag/v1.0.0) was released on July 13th, 2015, the first tag [v0.2](https://github.com/kubernetes/kubernetes/releases/tag/v0.2) was cut in September 9th, 2014

To make this work, we need to take a step back in time since container-based environments using cgroups v2 won’t be suitable. Container solutions rely on Kernel-level operations, and without altering Kernel configurations, we can't switch back from cgroups v2. Therefore, we're opting to construct a virtual machine that mirrors the past. We'll specifically use Ubuntu 15.04 (released on April 23rd, 2015), building it with cloudimg to capture the essence of that era.

Read the [documentation](https://github.com/kubernetes/kubernetes/blob/v1.0.0/docs/README.md) for Kubernetes v1.0, and explore which APIs have been removed and added since then at the [API Timeline](https://kube-api.ninja).

To begin, change to the root directory, update apt and ignore any warnings -

```bash
cd; sudo apt update -y
```


We're going to use qemu to run a virtual machine to host Kubernetes v1.0, install the requirements for qemu -

```bash
sudo apt install -y qemu qemu-kvm libvirt-daemon libvirt-clients bridge-utils virt-manager cloud-image-utils
```

Download the Ubuntu 15.04 cloudimg disk image (uses cgroups v1), we'll be using this to run our virtual machine -

```bash
wget http://cloud-images-archive.ubuntu.com/releases/vivid/release-20160203/ubuntu-15.04-server-cloudimg-amd64-disk1.img
```

Create a disk image -

```bash
qemu-img create -f qcow2 -F qcow2 -b ubuntu-15.04-server-cloudimg-amd64-disk1.img my-vm-disk.qcow2 20G
```

And configure an SSH Key for connectivity, we'll set this as a variable and later on we will use this to access our vm instance -

```bash
ssh-keygen -q -t rsa -N '' -f ~/.ssh/id_rsa <<<y
SSH_PUBLIC_KEY=$(cat ~/.ssh/id_rsa.pub)
```

Create a cloudinit config that sets the ubuntu password to "update" and injects our public ssh key, we'll use these when we start our vm -

```bash
echo -e "#cloud-config\nhostname: ubuntu\nmanage_etc_hosts: true\npassword: ubuntu\nchpasswd: { expire: False }\nssh_pwauth: True\nssh_authorized_keys:\n  - ${SSH_PUBLIC_KEY}" > user-data; cloud-localds user-data.img user-data
```

Open a new tab in cloudshell using the + icon, then, run the following in the new tab to emulate a vm using our disk image and cloudinit config. This will take a while to start -

```bash
qemu-system-x86_64 \
  -drive file=my-vm-disk.qcow2,format=qcow2 \
  -drive file=user-data.img,format=raw \
  -m 8192 \
  -smp cores=2 \
  -nographic \
  -netdev user,id=usernet,hostfwd=tcp::2222-:22 -device virtio-net,netdev=usernet
```

Wait for cloud init to complete, you'll see a message similar to - "Cloud-init v. 0.7.7 finished" after the initial login prompt, go back to the previous cloudshell tab when ready.

SSH to our instance and accept the host key -

```bash
ssh -p 2222 -o StrictHostKeyChecking=no ubuntu@localhost
```

We'll patch the system to give ourselves a working packages system for anything we may need by pointing repositories at the old-releases archives, may take a while to run -

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup && sudo sed -i 's|http://archive.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo sed -i 's|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo apt update
```

Our system certificates are also out of date, preventing the use of TLS with external targets, refresh the system certificates, this may also take a while to run -

```bash
sudo wget --no-check-certificate https://curl.se/ca/cacert.pem -O /usr/local/share/ca-certificates/cacert.crt; sudo rm -rf /etc/ssl/certs/*; sudo update-ca-certificates --fresh
```

Check for cgroups v1, it should say tmpfs -

```bash
stat -fc %T /sys/fs/cgroup/
```

Install docker using the ubuntu pkg version available at the time to match the release period of Kubernetes v1.0 -

```bash
sudo apt install -y docker.io
```

Download and install etcd, version 2, to also match the release period of Kubernetes v1.0, install to /usr/local/bin -

```bash
curl -L https://github.com/coreos/etcd/releases/download/v2.0.12/etcd-v2.0.12-linux-amd64.tar.gz -o etcd-v2.0.12-linux-amd64.tar.gz
tar xzvf etcd-v2.0.12-linux-amd64.tar.gz
sudo install etcd-v2.0.12-linux-amd64/etcd /usr/local/bin
```

Run etcd in the background as root and follow the logs, press `Ctrl-C` when you're ready, this will continue to run in background -

```bash
sudo bash -c 'etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://localhost:2379 &> /var/log/etcd.log &'; tail -f /var/log/etcd.log
```

Download and unpack the Kubernetes v1.0 release, within the package is another package for kubernetes-server, unpack this also -

```bash
wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.0/kubernetes.tar.gz
tar zxvf kubernetes.tar.gz
tar zxvf kubernetes/server/kubernetes-server-linux-amd64.tar.gz
```

Move the core kubernetes binaries to /usr/local/bin -

```bash
sudo install kubernetes/platforms/linux/amd64/kubectl /usr/local/bin
sudo install kubernetes/server/bin/kube-apiserver /usr/local/bin
sudo install kubernetes/server/bin/kube-controller-manager /usr/local/bin
sudo install kubernetes/server/bin/kube-proxy /usr/local/bin
sudo install kubernetes/server/bin/kube-scheduler /usr/local/bin
sudo install kubernetes/server/bin/kubelet /usr/local/bin
```

Run kube-apiserver in the background as `root` and follow the logs, press `Ctrl-C` when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-apiserver --etcd-servers=http://localhost:2379 --service-cluster-ip-range=10.0.0.0/16 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0 &> /var/log/kube-apiserver.log &'; tail -f /var/log/kube-apiserver.log
```

With kube-apiserver running, we will be able to see this via cluster-info -

```bash
kubectl cluster-info
```

Create a kubeconfig configuration file, we'll set a cluster, set the context to use this cluster and then, use this context -

```bash
kubectl config set-cluster k8s-v1.0 --server=http://localhost:8080
kubectl config set-context k8s-v1.0 --cluster=k8s-v1.0
kubectl config use-context k8s-v1.0
```

Show kubectl configuration -

```bash
kubectl config view
```

And show that a .kube/config exists with the same data -

```bash
cat ~/.kube/config
```

If we check with kubectl get nodes, although it will connect to the API server, currently we will have no nodes -

```bash
kubectl get nodes
```

Run the kubelet in the background as root and follow the logs, you're looking for a message similar to "Successfully registered node ubuntu", press `Ctrl-C` when you're ready, this will continue to run in background. This will register this node with the api-server. Click "Reject" when you get a popup with the title "Authorize Cloud Shell" this is not required, kubelet is checking if the node is running on a cloud provider vm.  -

```bash
sudo bash -c 'kubelet --api-servers=http://localhost:8080 --pod-infra-container-image="registry.k8s.io/pause:0.8.0" &> /var/log/kubelet.log &'; tail -f /var/log/kubelet.log
```

If we show nodes, we will now see one node -

```bash
kubectl get nodes
```

Run the kube-scheduler in the background as root and follow the logs (expect to see no output), press `Ctrl-C` when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-scheduler --master=http://localhost:8080 &> /var/log/kube-scheduler.log &'; tail -f /var/log/kube-scheduler.log
```

Run the kube-controller-manager in the background as root and follow the logs, press `Ctrl-C` when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-controller-manager --master=http://localhost:8080 &> /var/log/kube-controller-manager.log &'; tail -f /var/log/kube-controller-manager.log
```

Run the kube-proxy in the background as root and follow the logs (expect to see no output), press `Ctrl-C` when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-proxy --master=http://localhost:8080 &> /var/log/kube-proxy.log &'; tail -f /var/log/kube-proxy.log
```

Pulling container images from Docker Hub may fail due to changes in registry standards. Originally, Docker Hub utilized v1 registry standards, which have been deprecated in favor of the newer v2 standards recognized today. As a result, attempts to pull container images from Kubernetes to Docker Hub using the old standards will not succeed. However, there is a workaround to this issue.

To bypass the v1/v2 compatibility problem, preload the necessary images. This involves saving the images into a tar file and then manually loading them using the docker load command. For a more convenient method, Skopeo can be used to directly save images from Docker Hub into a tar file. Start by downloading and preparing Skopeo for this purpose.

```bash
sudo curl -L https://github.com/lework/skopeo-binary/releases/download/v1.14.4/skopeo-linux-amd64 -o /usr/bin/skopeo && sudo chmod 755 /usr/bin/skopeo; sudo mkdir -p /etc/containers; sudo echo '{ "default": [ { "type": "insecureAcceptAnything" } ] }' | sudo tee /etc/containers/policy.json
```

Create a convenient shell function that uses skopeo and docker load to preload images, you can use this function for any other images that you wish to use -

```bash
skopeo-save-load() { local safe_image_name=$(echo "$1" | tr '/:' '_'); local tar_path="/tmp/${safe_image_name}.tar"; skopeo copy "docker://$1" "docker-archive:${tar_path}:$1"; sudo docker load -i "$tar_path"; rm -f "$tar_path"; }
```

Preload nginx:latest -

```bash
skopeo-save-load nginx:latest
```

Preload registry.k8s.io/pause:0.8.0 -

```bash
skopeo-save-load registry.k8s.io/pause:0.8.0
```

Show the available Docker images -

```bash
sudo docker images
```

Let's try out Kubernetes v1.0. The syntax is different, we'll run 5 pods which in turn, will create a replicationcontroller for us -

```bash
kubectl run nginx --image=nginx --replicas=5
```

Check the pods until they are running -

```bash
kubectl get pods
```

As Docker is the container runtime, you will be able to see the pods running in docker as well -

```bash
sudo docker ps -a
```

We can also expose these as a service -

```bash
kubectl expose rc nginx --port=80 --target-port=80
```

Query the available services -
```bash
kubectl get svc
```

Capture the svc IP -

```bash
SVC_IP=$(kubectl get svc nginx -o template --template={{.spec.clusterIP}})
```

And now you can curl that IP -

```bash
curl $SVC_IP
```

Congratulations, you've successfully used Kubernetes v1.0. Here's to looking forward to another 10 years of Kubernetes!
