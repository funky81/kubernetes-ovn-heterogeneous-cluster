# Worker nodes

## Linux

### Node set-up

Let's provision the Linux worker VM:
```sh
gcloud compute instances create "sig-windows-worker-linux-1" \
    --zone "us-east1-d" \
    --machine-type "custom-2-2048" \
    --can-ip-forward \
    --image-family "ubuntu-1604-lts" \
    --image-project "ubuntu-os-cloud" \
    --boot-disk-size "50" \
    --boot-disk-type "pd-ssd"
```

When it's ready, SSH into it:
```sh
gcloud compute ssh --zone "us-east1-d" "sig-windows-worker-linux-1"
```

**ATTENTION**: From now on, it's assumed you're logged-in as `root`.

Let's install OVS/OVN:
```sh
curl -fsSL https://yum.dockerproject.org/gpg | apt-key add -
echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" > sudo tee /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker.io dkms
```

```sh
cd ~
git clone https://github.com/funky81/kubernetes-ovn-heterogeneous-cluster
cd kubernetes-ovn-heterogeneous-cluster/deb

dpkg -i openvswitch-common_2.7.2-1_amd64.deb \
openvswitch-datapath-dkms_2.7.2-1_all.deb \
openvswitch-switch_2.7.2-1_amd64.deb \
ovn-common_2.7.2-1_amd64.deb \
ovn-central_2.7.2-1_amd64.deb \
ovn-docker_2.7.2-1_amd64.deb \
ovn-host_2.7.2-1_amd64.deb \
python-openvswitch_2.7.2-1_all.deb
```

We'll need to make sure `vport_geneve` kernel module is loaded at boot:
```sh
echo vport_geneve >> /etc/modules-load.d/modules.conf
```

Finally, reboot:
```sh
reboot
```

SSH again into the machine, become root and proceed to configure OVS/OVN.

**ATTENTION**:
* From now on, it's assumed you're logged-in as `root`.
* Pay attention to the environment variables below, particularly:
  * `LOCAL_IP` must be the public IP of this node
  * `MASTER_IP` must be the remote public IP address of the master node

Create the OVS bridge interface:
```sh
export TUNNEL_MODE=geneve
export LOCAL_IP=10.142.0.3
export MASTER_IP=10.142.0.2

ovs-vsctl set Open_vSwitch . external_ids:ovn-remote="tcp:$MASTER_IP:6642" \
  external_ids:ovn-nb="tcp:$MASTER_IP:6641" \
  external_ids:ovn-encap-ip="$LOCAL_IP" \
  external_ids:ovn-encap-type="$TUNNEL_MODE"

ovs-vsctl get Open_vSwitch . external_ids
```

You should see something like:
```
{hostname=sig-windows-worker-linux-1.c.apprenda-project-one.internal, ovn-encap-ip="10.142.0.3", ovn-encap-type=geneve, ovn-nb="tcp:10.142.0.2:6641", ovn-remote="tcp:10.142.0.2:6642", system-id="c6364c4c-8069-4bfd-ace8-ec701572feb7"}
```

We are now ready to set-up Kubernetes Linux worker node.

### Kubernetes set-up

**CA keypair set-up**:

In order to generate certificates for the worker node, and to establish trust between
the kubelet and the API server, we need to copy the CA keypair from the master node to
the worker node. On GCE, and _only for Linux VMs_, you may use the `gcloud compute copy-files` command to copy
the keypair from the master node to your local machine, and then from your local machine
to the new worker node.

**Attention**:
* From now on, it's assumed you're logged-in as `root`.
* You **must** copy the CA keypair that's available in the master node over to the 
worker node at the following paths:
  * /etc/kubernetes/tls/ca.pem
  * /etc/kubernetes/tls/ca-key.pem
* If you exited your shell, make sure to export the environment variables defined in previous steps
before proceeding.
* Pay attention to the environment variables below

```sh
cd ~/kubernetes-ovn-heterogeneous-cluster/worker/linux

rm -rf tmp
mkdir tmp
cp -R ../make-certs ../openssl.cnf ../kubeconfig.yaml systemd tmp/

export HOSTNAME=`hostname`
export K8S_VERSION=1.7.3
export K8S_POD_SUBNET=10.244.0.0/16
export K8S_NODE_POD_SUBNET=10.244.2.0/24
export K8S_DNS_SERVICE_IP=10.100.0.10
export K8S_DNS_DOMAIN=cluster.local

sed -i"*" "s|__K8S_VERSION__|$K8S_VERSION|g" tmp/systemd/kubelet.service

sed -i"*" "s|__MASTER_IP__|$MASTER_IP|g" tmp/systemd/kubelet.service
sed -i"*" "s|__MASTER_IP__|$MASTER_IP|g" tmp/openssl.cnf
sed -i"*" "s|__MASTER_IP__|$MASTER_IP|g" tmp/kubeconfig.yaml

sed -i"*" "s|__LOCAL_IP__|$LOCAL_IP|g" tmp/systemd/kubelet.service
sed -i"*" "s|__LOCAL_IP__|$LOCAL_IP|g" tmp/openssl.cnf

sed -i"*" "s|__HOSTNAME__|$HOSTNAME|g" tmp/systemd/kubelet.service
sed -i"*" "s|__HOSTNAME__|$HOSTNAME|g" tmp/make-certs

sed -i"*" "s|__K8S_DNS_SERVICE_IP__|$K8S_DNS_SERVICE_IP|g" tmp/systemd/kubelet.service
sed -i"*" "s|__K8S_DNS_DOMAIN__|$K8S_DNS_DOMAIN|g" tmp/systemd/kubelet.service

cd tmp
chmod +x make-certs
./make-certs
cd ..

mkdir -p /etc/kubernetes/

cp tmp/kubeconfig.yaml /etc/kubernetes/

cp -R tmp/systemd/*.service /etc/systemd/system/
systemctl daemon-reload

curl -Lskj -o /usr/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v$K8S_VERSION/bin/linux/amd64/kubectl
chmod +x /usr/bin/kubectl

kubectl config set-cluster default-cluster --server=https://$MASTER_IP --certificate-authority=/etc/kubernetes/tls/ca.pem
kubectl config set-credentials default-admin --certificate-authority=/etc/kubernetes/tls/ca.pem --client-key=/etc/kubernetes/tls/node-key.pem --client-certificate=/etc/kubernetes/tls/node.pem
kubectl config set-context local --cluster=default-cluster --user=default-admin
kubectl config use-context local
 ```

Now, let's configure pod networking for this node:
```sh
export TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')

ovs-vsctl set Open_vSwitch . \
  external_ids:k8s-api-server="https://$MASTER_IP" \
  external_ids:k8s-api-token="$TOKEN"

ln -fs /etc/kubernetes/tls/ca.pem /etc/openvswitch/k8s-ca.crt

mkdir -p /opt/cni/bin && cd /opt/cni/bin
curl -Lskj -o cni.tar.gz https://github.com/containernetworking/cni/releases/download/v0.4.0/cni-v0.4.0.tgz
tar zxf cni.tar.gz
rm -f cni.tar.gz

apt install -y python-pip

pip install --upgrade pip

cd ~
git clone https://github.com/openvswitch/ovn-kubernetes
cd ovn-kubernetes

pip install --upgrade --prefix=/usr/local --ignore-installed .

ovn-k8s-overlay minion-init \
  --cluster-ip-subnet="$K8S_POD_SUBNET" \
  --minion-switch-subnet="$K8S_NODE_POD_SUBNET" \
  --node-name="$HOSTNAME"
```

By this time, your Linux worker node is ready to run Kubernetes workloads:
```sh
systemctl enable kubelet
systemctl start kubelet
kubectl get nodes
kubectl -n kube-system get pods
```

Let's proceed to setup the Windows worker node.

[**Go back**](../README.md#cluster-deployment).
