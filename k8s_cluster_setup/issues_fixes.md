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

---

### Unable to connect k8s Cluster
<pre>
Issue1: After Node restart, unable to connect to k8s cluster

Fix: Ensure the ~/.kube/config file is configured

kubectl get pods -A -o wide
E1029 08:37:59.856378   60476 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"

The connection to the server localhost:8080 was refused - did you specify the right host or port?
</pre>
```bash
#Make sure following services are active and running.
sudo systemctl enable --now containerd
sudo systemctl enable --now kubelet

sudo systemctl status containerd
sudo systemctl status kubelet

#This could be due to missing kube config reference.
export KUBECONFIG=/etc/kubernetes/admin.conf

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### Accessing console UI through Load Balancer (Routing Issue)
<pre>
Issue1: K8s console UI not loading when accessed through browser.

Fix: Step by Step troubleshooting to identify and resolve the issue.
</pre>
```bash
#From k8s Nodes; request to Load Balancer URL gives proper response
echo | openssl s_client -connect 192.168.111.200:443 -servername console.192.168.111.200.nip.io 2>/dev/null | openssl x509 -noout -issuer
curl -ik https://console.192.168.111.200.nip.io 2> /dev/null | grep -iw http
```
<pre>
Why it failed: nobody owns 192.168.111.200 in BGP mode, so laptop got no answer and used the default network path.
Laptop:
    PS> Test-NetConnection 192.168.111.200 -Port 443
    WARNING: TCP connect to (192.168.111.200 : 443) failed
    WARNING: Ping to 192.168.111.200 failed with status: TimedOut
    InterfaceAlias         : Wi-Fi            <<< left through Wi-Fi, not VMnet8
    SourceAddress          : 192.168.8.34     <<< should be 192.168.111.1
    TcpTestSucceeded       : False

Find Interface Index
    PS> arp -a | findstr 192.168.111
    Interface: 192.168.111.1 --- 0x3
    192.168.111.130       00-0c-29-c6-ce-8c     dynamic
    192.168.111.131       00-0c-29-f5-a6-8e     dynamic
    192.168.111.132       00-0c-29-00-6d-ae     dynamic
    192.168.111.254       00-50-56-e8-3e-8a     dynamic
    192.168.111.255       ff-ff-ff-ff-ff-ff     static

Fix1: Add route
    PS> route -p add 192.168.111.200 mask 255.255.255.255 192.168.111.130 metric 1 if 3

    PS> Test-NetConnection 192.168.111.200 -Port 443
    WARNING: TCP connect to (192.168.111.200 : 443) failed
    WARNING: Ping to 192.168.111.200 failed with status: DestinationHostUnreachable
    InterfaceAlias         : VMware Network Adapter VMnet8
    SourceAddress          : 192.168.111.1
    PingSucceeded          : False

After adding route in Laptop, the network path is now corrected. Checking at Master Node (192.168.111.130) whether it have the route to reach 192.168.111.200

Master Node:
    [root@k8s-master ~] ip route get 192.168.111.200
    192.168.111.200 dev ens160 src 192.168.111.130 uid 0        <<< no "via <worker>"

    [root@k8s-master ~] vtysh -c "show ip bgp 192.168.111.200/32"
    Paths: (2 available, no best path)                          <<< no best path
    192.168.111.131 (inaccessible, import-check enabled) ...
    Origin IGP, invalid, external                         <<< routes marked invalid

    [root@k8s-master ~] vtysh -c "show ip nht"
    192.168.111.131(Connected)
    unresolved(Connected)                                      <<< can't resolve the next hop

    [root@k8s-master ~] vtysh -c "show interface ens160"
    inet 192.168.111.130/24 noprefixroute               <<< address added without a connected route

    [root@k8s-master ~] vtysh -c "show ip route connected"
    C>* 10.244.235.192/32 is directly connected, vxlan.calico   <<< no 192.168.111.0/24 entry

Fix2: Tell FRR to accept neighbours without connected-route test
    configure terminal
    router bgp 64512
        neighbor 192.168.111.131 disable-connected-check
        neighbor 192.168.111.132 disable-connected-check
    end
    write memory

    vtysh -c "show ip bgp"
    ip route show 192.168.111.200
    vtysh -c "show ip bgp summary"

Even after adding neighbours, the Master’s own kube-proxy dropped the traffic. Traefik’s service used "externalTrafficPolicy: Local", which means only a node that runs a Traefik pod may handle this. The master has none, so its kube-proxy discarded the packets.

Master Node:
    [root@k8s-master ~] iptables-save | grep 192.168.111.200
    -A KUBE-EXTERNAL-SERVICES -d 192.168.111.200/32 -p tcp ... "traefik/traefik:websecure has no local endpoints" ... --dport 443 -j DROP   <<< KEY
    -A KUBE-EXTERNAL-SERVICES -d 192.168.111.200/32 -p tcp ... "traefik/traefik:web has no local endpoints" ... --dport 80 -j DROP

    [root@k8s-master ~] nft list ruleset | grep 192.168.111.200
    ip daddr 192.168.111.200  tcp dport 443 counter packets 15 bytes 780 drop   <<< 15 of your packets dropped here

Fix 3: Update Traefik externalTrafficPolicy from Local to Cluster
    helm upgrade traefik traefik/traefik -n traefik --reuse-values --set service.spec.externalTrafficPolicy=Cluster
</pre>