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

Download the Ubuntu 15.04 (uses cgroups v1) cloudimg disk image, we'll be using this to run our virtual machine -

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

Wait for cloud init to complete, you'll see something like - "Cloud-init v. 0.7.7 finished", go back to the previous tab when ready.

SSH to our instance, may take a while to connect as our host entry is incorrect and we will need to wait for a DNS timeout, we'll fix this when we're inside the instance -

```bash
ssh -p 2222 ubuntu@localhost
```

Fix ubuntu hostname resolution, run once and ignore errors -

```bash
echo $(hostname -I | awk {'print $1'}) ubuntu | sudo tee -a /etc/hosts
```

We'll patch the system to give ourselves a working packages system for anything we may need, may take a while to run as the archives are slower -

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup && sudo sed -i 's|http://archive.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo sed -i 's|http://security.ubuntu.com/ubuntu|http://old-releases.ubuntu.com/ubuntu|g' /etc/apt/sources.list && sudo apt update
```

Check for cgroups v1, it should say tmpfs -

```bash
stat -fc %T /sys/fs/cgroup/
```

Install docker using the ubuntu pkg version available at the time -

```bash
sudo apt install -y docker.io
```

Download and install etcd -

```bash
curl -L https://github.com/coreos/etcd/releases/download/v2.0.12/etcd-v2.0.12-linux-amd64.tar.gz -o etcd-v2.0.12-linux-amd64.tar.gz
tar xzvf etcd-v2.0.12-linux-amd64.tar.gz
sudo install etcd-v2.0.12-linux-amd64/etcd /usr/local/bin
```

Run etcd and follow the logs, press ctrl-c when ready, will continue to run in background -

```bash
sudo bash -c 'etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls http://localhost:2379 &> /var/log/etcd.log &'; tail -f /var/log/etcd.log
```

Download and unpack the Kubernetes binaries, place in /usr/local/bin -

```bash
wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.0/kubernetes.tar.gz
tar zxvf kubernetes.tar.gz
tar zxvf kubernetes/server/kubernetes-server-linux-amd64.tar.gz
sudo install kubernetes/platforms/linux/amd64/kubectl /usr/local/bin
sudo install kubernetes/server/bin/kube-apiserver /usr/local/bin
sudo install kubernetes/server/bin/kube-controller-manager /usr/local/bin
sudo install kubernetes/server/bin/kube-proxy /usr/local/bin
sudo install kubernetes/server/bin/kube-scheduler /usr/local/bin
sudo install kubernetes/server/bin/kubelet /usr/local/bin
```

Run kube-apiserver -

```bash
sudo bash -c 'kube-apiserver --etcd-servers=http://localhost:2379 --service-cluster-ip-range=10.0.0.0/16 --bind-address=0.0.0.0 --insecure-bind-address=0.0.0.0 &> /var/log/kube-apiserver.log &'; tail -f /var/log/kube-apiserver.log
```

Create a kubeconfig -

```bash
kubectl cluster-info
kubectl config set-cluster k8s-v1.0 --server=http://localhost:8080
kubectl config set-context k8s-v1.0 --cluster=k8s-v1.0
kubectl config use-context k8s-v1.0
```

Show kubectl configuration -

```bash
kubectl config view
```

And show that a .kube/config now exists with the same data -

```bash
cat .kube/config
```

Get nodes, currently we will have no nodes -

```bash
kubectl get nodes
```

Start the kubelet and register with the api server, follow logs, press ctrl-c to exit -

```bash
sudo bash -c 'kubelet --api-servers=http://localhost:8080 &> /var/log/kubelet.log &'; tail -f /var/log/kubelet.log
```

Show nodes (we will now see one node) -

```bash
kubectl get nodes
```

Start the scheduler (no output when successful), press ctrl-c to stop following logs -

```bash
sudo bash -c 'kube-scheduler --master=http://localhost:8080 &> /var/log/kube-scheduler.log &'; tail -f /var/log/kube-scheduler.log
```

Start the controller-manager, press ctrl-c to stop following logs -

```bash
sudo bash -c 'kube-controller-manager --master=http://localhost:8080 &> /var/log/kube-controller-manager.log &'; tail -f /var/log/kube-controller-manager.log
```

Start kube proxy (no output when successful), press ctrl-c to stop following logs -

```bash
sudo bash -c 'kube-proxy --master=http://localhost:8080 &> /var/log/kube-proxy.log &'; tail -f /var/log/kube-proxy.log
```

Docker Hub will not work, owing to changes in the registry standards, therefore we will manually need to load images. We're going to load nginx:1.7 which at the time is 9 years old -

```bash
curl -L https://github.com/spurin/docker-hub-legacy-images/raw/main/nginx-1.7.tar | sudo docker load
```

And we'll go back in time and make nginx:1.7 nginx:latest -

```bash
sudo docker tag nginx:1.7 nginx:latest
```

Show images -

```bash
sudo docker images
```

Let's try out Kubernetes v1.0. The syntax is different, we'll run 5 pods which in turn, will create a replicationcontroller for us -

```bash
kubectl run nginx --image=nginx --replicas=5
```

Check pods until they are running -

```bash
kubectl get pods
```

As Docker is the container runtime, you can see the pods running in docker as well -

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

And now you can curl that IP if you wish! Congratulations and looking forward to another 10 years of Kubernetes!
