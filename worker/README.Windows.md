## Windows

**Attention**:
* Connectivity may flicker and you may need to reconnect your RDP session. This is expected at least a couple times when setting up networking interfaces.
* Make sure to disable the Windows Firewall for all profiles. Follow the instructions at https://technet.microsoft.com/en-us/library/cc766337%28v=ws.10%29.aspx for more details on how to do this.

For the sake of simplicity when setting up Windows node, we will not be relying on TLS for now.

### Node set-up

Let's provision the Windows worker VM:
```sh
gcloud compute instances create "sig-windows-worker-windows-1" \
  --zone "us-east1-d" \
  --machine-type "custom-4-4096" \
  --can-ip-forward \
  --image-family "windows-2016" \
  --image-project "windows-cloud" \
  --boot-disk-size "50" \
  --boot-disk-type "pd-ssd"
```

After VM is provisioned, establish a new connection to it. If you are running on GCE,
you may find instructions on how to RDP into the Windows machine [here](https://cloud.google.com/compute/docs/instances/windows/connecting-to-windows-instance).

Now, start a new Powershell session with administrator privileges and execute:
```sh
cd \
mkdir ovs
cd ovs

Start-BitsTransfer https://cloudbase.it/downloads/openvswitch-hyperv-2.7.0-certified.msi
Start-BitsTransfer https://cloudbase.it/downloads/k8s_ovn_service_prerelease.zip

netsh netkvm setparam 0 *RscIPv4 0
netsh netkvm restart 0

Install-WindowsFeature -Name Containers

Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
Install-Package -Name docker -ProviderName DockerMsftProvider

Register-PSRepository -Name DockerPS-Dev -SourceLocation https://ci.appveyor.com/nuget/docker-powershell-dev
Install-Module -Name Docker -Repository DockerPS-Dev -Scope CurrentUser
```

A reboot is mandatory:
```sh
Restart-Computer -Force
```

Re-establish connection to the VM, and install the Open vSwitch MSI:
```
cd c:\ovs
cmd /c 'msiexec /i openvswitch-hyperv-2.7.0-certified.msi ADDLOCAL="OpenvSwitchCLI,OpenvSwitchDriver,OVNHost" /qn' 
```

After installation of the Open vSwitch MSI, close your powershell window, and start a new Powershell session with administrator privileges.  This is to ensure that the Open vSwitch binaries are on your $PATH.

Now, one needs to set-up the overlay (OVN) network. On a per node basis, download `install_ovn.ps1` over to `C:\ovs` on the Windows node.

```sh
cd c:\ovs
Start-BitsTransfer https://raw.githubusercontent.com/funky81/kubernetes-ovn-heterogeneous-cluster/master/worker/windows/install_ovn.ps1
```
Now, **edit the contents of install_ovn.ps1 accordingly before running the Powershell script**.
```sh
.\install_ovn.ps1
```

We are now ready to set-up Kubernetes Windows worker node.

### Kubernetes set-up

On a per node basis, download `install_k8s.ps1` over to `c:\ovs` on the Windows node.
```
cd c:\ovs
Start-BitsTransfer https://raw.githubusercontent.com/funky81/kubernetes-ovn-heterogeneous-cluster/master/worker/windows/install_k8s.ps1
```
Now, **edit the contents of install_k8s.ps1 accordingly before running the Powershell script**.
Then, start a new Powershell session with administrator privileges to install Kubernetes:
```sh
.\install_k8s.ps1
```

Before you run the kubelet, **edit the flag values below according to your environment**:
```sh
$env:CONTAINER_NETWORK = "external"
.\kubelet.exe -v=3 --hostname-override=sig-windows-worker-windows-1 --cluster-dns=10.100.0.10 --cluster-domain=cluster.local --pod-infra-container-image="apprenda/pause" --resolv-conf="" --api_servers="http://10.142.0.2:8080"
```

If everything is working, you should see all three nodes and several pods in the output of these kubectl commands:
```sh
cd C:\kubernetes
.\kubectl.exe -s 10.142.0.2:8080 get nodes
.\kubectl.exe -s 10.142.0.2:8080 -n kube-system get pods
```

[**Go back**](../README.md#cluster-deployment).
