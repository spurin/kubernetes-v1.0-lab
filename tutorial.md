![K8S 10 Years](https://www.linuxfoundation.org/hubfs/hbd_k8s-1.svg)

This tutorial will allow you to run Kubernetes v1.0 - in your browser by making use of the Free Google Cloud Shell tier that can be used without enrollment, you're using it now! 🚀

Credits to Carlos Santana, Amim Moises Salum Knabben & James Spurin

Carlos Santana kicked off the fun in the CNCF Ambassador chat room, sparking a brilliant idea to celebrate Kubernetes' 10-year anniversary! Amim dived deep, tackling key challenges and identifying core issues. Meanwhile, James Spurin whipped up some nifty workarounds. Together, they've crafted this engaging tutorial. Dive in, have fun, and join us in cheering for a fantastic decade of Kubernetes! 🎉

To make this work, we need to take a step back in time since container-based environments using cgroups v2 won’t be suitable. Container solutions rely on Kernel-level operations, and without altering Kernel configurations, we can't switch back from cgroups v2. Therefore, we're opting to construct a virtual machine that mirrors the past. We'll specifically use Ubuntu 15.04, building it with cloudimg to capture the essence of that era.

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
````

Create a cloudinit config that sets the ubuntu password to "update" and injects our public ssh key, we'll use these when we start our vm -

```bash
echo -e "#cloud-config\npassword: ubuntu\nchpasswd: { expire: False }\nssh_pwauth: True\nssh_authorized_keys:\n  - ${SSH_PUBLIC_KEY}" > user-data
cloud-localds user-data.img user-data
```

Open a new tab in cloudshell using the + icon, then, run the following in the new tab to emulate a vm using our disk image and cloudinit config. This will take a while to start -

```bash
qemu-system-x86_64 \
  -drive file=my-vm-disk.qcow2,format=qcow2 \
  -drive file=user-data.img,format=raw \
  -m 16384 \
  -smp cores=4 \
  -nographic \
  -netdev user,id=usernet,hostfwd=tcp::2222-:22 -device virtio-net,netdev=usernet
```

Wait for cloud init to complete, you'll see a message similar to - "Cloud-init v. 0.7.7 finished" after the initial login prompt, go back to the previous cloudshell tab when ready.

SSH to our instance and accept the host key, may take a while to connect as our host entry is incorrect and we will need to wait for a DNS timeout. We will fix this when we're inside the instance as the next step -

```bash
ssh -p 2222 -o StrictHostKeyChecking=no ubuntu@localhost
```

Fix our hostname resolution, run once and ignore the sudo error which relates to this current misconfiguration, after this step sudo can be used without any errors -

```bash
echo $(hostname -I | awk {'print $1'}) ubuntu | sudo tee -a /etc/hosts
```

We'll patch the system to give ourselves a working packages system for anything we may need by pointing repositories at the old-releases archives, may take a while to run -

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup && sudo sed -i 's|http://archive.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo sed -i 's|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo apt update
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

Run etcd in the background as root and follow the logs, press ctrl-c when you're ready, this will continue to run in background -

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

Run kube-apiserver in the background as root and follow the logs, press ctrl-c when you're ready, this will continue to run in background -

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
cat .kube/config
```

If we check with kubectl get nodes, although it will connect to the API server, currently we will have no nodes -

```bash
kubectl get nodes
```

Run the kubelet in the background as root and follow the logs, you're looking for a message similar to "Successfully registered node ubuntu", press ctrl-c when you're ready, this will continue to run in background. This will register this node with the api-server -

```bash
sudo bash -c 'kubelet --api-servers=http://localhost:8080 &> /var/log/kubelet.log &'; tail -f /var/log/kubelet.log
```

If we show nodes, we will now see one node -

```bash
kubectl get nodes
```

Run the kube-scheduler in the background as root and follow the logs (expect to see no output), press ctrl-c when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-scheduler --master=http://localhost:8080 &> /var/log/kube-scheduler.log &'; tail -f /var/log/kube-scheduler.log
```

Run the kube-controller-manager in the background as root and follow the logs, press ctrl-c when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-controller-manager --master=http://localhost:8080 &> /var/log/kube-controller-manager.log &'; tail -f /var/log/kube-controller-manager.log
```

Run the kube-proxy in the background as root and follow the logs (expect to see no output), press ctrl-c when you're ready, this will continue to run in background -

```bash
sudo bash -c 'kube-proxy --master=http://localhost:8080 &> /var/log/kube-proxy.log &'; tail -f /var/log/kube-proxy.log
```

Docker Hub will not work, owing to changes in the registry standards, therefore we will manually need to load images. We're going to load nginx:1.7 which at the time is 9 years old, we'll download this and pipe it direct to docker load -

```bash
curl -L https://github.com/spurin/docker-hub-legacy-images/raw/main/nginx-1.7.tar | sudo docker load
```

And we'll go back in time and make nginx:1.7 nginx:latest through a manual tag, allowing us to use nginx with no tag in kubernetes -

```bash
sudo docker tag nginx:1.7 nginx:latest
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
SVC_IP=$(kubectl get svc | grep nginx | awk {'print $4'}); echo $SVC_IP
```

And now you can curl that IP -

```bash
curl $SVC_IP
```

Congratulations, you've successfully used Kubernetes v1.0. Here's to looking forward to another 10 years of Kubernetes!
