### Installing Calico (CNI)
<pre>
Issue1: While creating calico, k8s verion and calico operator compatability issue.

Fix: On All Nodes K8s version upgraded from 1.30 to 1.34
</pre>
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube
sudo dnf remove -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes

sudo sed -i 's#v1\.30#v1.34#g' /etc/yum.repos.d/kubernetes.repo
grep -n 'v1\.' /etc/yum.repos.d/kubernetes.repo   # both URLs should show v1.34

sudo dnf clean all
sudo dnf install -y kubelet kubeadm kubectl --setopt=disable_excludes=kubernetes
sudo systemctl enable --now kubelet
    
#Should print new k8s version - v1.34.x
kubeadm version -o short   

#Initialize k8s Cluster
sudo kubeadm init --apiserver-advertise-address=192.168.111.130 --pod-network-cidr=10.244.0.0/16
```
<pre>
After fixing the compatability issue between k8s version and Calico version, follow the steps mentioned under: Installing Calico (CNI)
</pre>

<pre>	
Issue2: While joining Node with Master encountered preflight error

Fix: Reset kubeadm, and cleanup leftovers from previous join command execution

[preflight] Some fatal errors occurred:
[ERROR FileAvailable--etc-kubernetes-bootstrap-kubelet.conf]: /etc/kubernetes/bootstrap-kubelet.conf already exists
[ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: /etc/kubernetes/pki/ca.crt already exists

A previous join attempt, probably against the old 1.30 k8s cluster, left files behind. Clean the worker and join again.
</pre>    
```bash
#On Worker Node where preflight error occured
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /etc/kubernetes /var/lib/kubelet $HOME/.kube
sudo systemctl restart containerd
kubeadm version -o short

#Then join Worker Nodes by executing the join command as root
```	
<pre>
Issue3: While joining 2nd Worker Node to Master Node, encountered error.
error: error execution phase kubelet-start: a Node with name "k8s-worker1" and status "Ready" already exists in the cluster. You must delete the existing Node or change the name of this new joining Node
	
Fix:2nd Worker Node was also having same hostname as 1st Worker Node, updated the 2nd Worker hostname, reset the kubeadm and join again.
</pre>
```bash    
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/kubelet

#Then join Worker Nodes by executing the join command as root
```