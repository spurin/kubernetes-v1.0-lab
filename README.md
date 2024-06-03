## Kubernetes 10 Year Anniversary - Try out Kubernetes v1.0 - Free in your browser!


<p align="center">
  <img width="30%" src="https://www.linuxfoundation.org/hubfs/hbd_k8s-1.svg" />
</p>


✨ This tutorial will allow you to run Kubernetes v1.0 - in your browser by making use of the Free Google Cloud Shell tier that can be used without enrollment. Click the "Open in Google Cloud Shell" button, sign in with Google, do not trust the repo (as we want to run this as an ephemeral environment that is discarded after use) and then follow the tutorial on the right hand side 🚀

Credits to [Carlos Santana](https://github.com/csantanapr), [Amim Moises Salum Knabben](https://github.com/knabben) & [James Spurin](https://github.com/spurin)

Carlos Santana kicked off the fun in the CNCF Ambassador chat room, sparking a brilliant idea to celebrate Kubernetes' 10-year anniversary! Amim dived deep, tackling key challenges and identifying core issues. Meanwhile, James Spurin whipped up some nifty workarounds with this virtual environment.

Together, they've crafted this engaging tutorial. Dive in, have fun, and join us in cheering for a fantastic decade of Kubernetes! 🎉 ✨

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://ssh.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/spurin/kubernetes-v1.0-lab.git&cloudshell_git_branch=main&cloudshell_tutorial=tutorial.md&shellonly=true)

## Run in Lima VM

[Anders Björklund](https://github.com/afbjorklund) has also done some great work, allowing you to run the pre-requisite VM environment yourself with [Lima VM](https://github.com/lima-vm/lima).

Save the following gist as a file - [lima.yaml](https://gist.github.com/afbjorklund/c99634a2a34aa3315f7d7db0f54526f6)

Then start the instance with `limactl start --cpus 2 --memory 8 lima.yaml`

You can then follow the [tutorial](tutorial.md) onwards from the VM initialisation and access stage.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=spurin/kubernetes-v1.0-lab&type=Date)](https://star-history.com/#spurinkubernetes-v1.0-lab&Date)
