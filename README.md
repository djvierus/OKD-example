# Setting Up and OKD Single Node Cluster

## <a name="description"></a> Description
This is a walkthrough on how to setup a SCOS (Centos Stream CoreOS) OKD cluster using the example yaml files provided by this project. 
[OKD's Latest Single Node Cluster Documentation](https://docs.okd.io/latest/installing/installing_sno/install-sno-installing-sno.html) references the Fedora images as well as the original [OKD Github Project](https://github.com/okd-project/okd) instead of the [OKD SCOS Github Project](https://github.com/okd-project/okd-scos). There are Pull Requests on the documenation project that should address this in future documentation releases, but in the mean time, I am writing up a walkthrough here until that Pull Request is accepted into the documentation project. This project will walk through installing OKD on a bare-metal setup using an ISO file on a blank machine or virtual machine.

## Prerequisites 
This first version of the walkthrough expects that you have the following in your environment and can make modifications to these services

#### DHCP Server With MAC Address Reservations
Simple OKD setups require a DHCP router\server on your network to assign an IP address on a new server when setting up your server. Be sure you have the MAC address of the network adapter of your physical\virtual machine.

#### DNS Server
The Simple OKD setup requires a DNS server with a reverse pointer record assigned to the reserved IP address on the DHCP server. This will let the server name itself as it boots and installs. On top of that, OKD is heavily DNS Based and is required to use DNS based routing to applications and cluster management. **Ensure your DHCP server uses this DNS server as the primary DNS server for clients requesting DHCP requests**

DNS Example configs for "yourdomain.com" provided in the "examples/dns" folder of this project. DNS assumes your DHCP reservation was for a "192.168.1.101" IP address. These config files work for a bind DNS server. Not sure how well it works with other DNS servers.

#### Redhat Account with Developer's license.
OKD is Redhat based and will pull images using your pull secret. After creating a Redhat account, you can access your pull secret [here](https://console.redhat.com/openshift/install/pull-secret)

#### RECOMMENDED: Linux workstation
To run through this walkthrough, a Linux workstation, or at the very least, Windows with WSL (Windows Subsystem for Linux) is preferred. Windows may be doable, but this guide does make the assumption that if there is any missing info here, you can reference the official OKD documentation linked in the [Description](#description). Their documentation has most of their examples in a Fedora type Linux workstation environment. 

**It will help tremendously if you use a Fedora based Linux workstation.** Podman typically get's installed out-of-the-box and will be needed later in this walkthough. Docker may work instead, but that is untested.

## Download the Release Client
Go to the [OKD SCOS Github Project releases page](https://github.com/okd-project/okd-scos/releases) and download the latest\stable tagged release's client tools. Current Linux Client download is "https://github.com/okd-project/okd-scos/releases/download/4.16.0-okd-scos.1/openshift-client-linux-4.16.0-okd-scos.1.tar.gz".

Rename, Untar the client, and make it executable with the following commands:

Rename the package:  
```mv openshift-client-linux-<version>.tar.gz oc.tar.gz```  

Untar the file:  
```tar zxf oc.tar.gz```  

Make it executable:  
```chmod +x oc```  

You can verify it works with the ```oc version``` command. If Linux cannot find the file, you may need to re-run the command with the ```./oc version``` command to let Linux know the command is in the current directory you are in.

You can make the client available to all users on the system without needing to be in the directory the client is in by moving it the the ```/usr/bin``` directore
```
sudo mv ./oc /usr/bin/
```

## Download the Release Installer
On the same releases page, download the client installer. Current Linux Client downlaod is 
"https://github.com/okd-project/okd-scos/releases/download/4.16.0-okd-scos.1/openshift-install-linux-4.16.0-okd-scos.1.tar.gz".

Untar and make it executable like the client tool:  
Rename the release installer:  
```mv openshift-install-linux<version>.tar.gz openshift-install-linux.tar.gz```  
Untar the package:  
```tar zxvf openshift-install-linux.tar.gz```  
Make the package executable:    
```chmod +x openshift-install```  

## Download the base ISO
This is where the documentation really breaks for OKD's walkthrough. The previous 4.16.0-okd.scos.0 release did not download the correct Centos Based ISO. That is an issue being addressed [here](https://github.com/okd-project/okd-scos/issues/11).

Find the latest SCOS ISO image by going to "https://okd-scos.s3.amazonaws.com/okd-scos/builds/builds.json" and record the name of the "id". For example "414.9.202306050735-0"

Download the blank ISO by going to " https://okd-scos.s3.amazonaws.com/okd-scos/builds/'version you recorded'/x86_64/scos-'version you recorded'-live.x86_64.iso". 

Example with example version " https://okd-scos.s3.amazonaws.com/okd-scos/builds/414.9.202306050735-0/x86_64/scos-414.9.202306050735-0-live.x86_64.iso"

NOTE: To keep a good copy of the iso, you may copy the iso file and rename it "scos-live.iso"

## Create your Ignition file
The ignition file will be used to setup the new server. Copy your public ssh key from your .ssh/id_rda.pub key if you have one to the sshKey section. This will help troubleshoot the node if the install starts to go south.

```yaml
apiVersion: v1
baseDomain: yourdomain.com 
compute:
- name: worker
  replicas: 0 
controlPlane:
  name: master
  replicas: 1 
metadata:
  name: homelab 
networking: 
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16 
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
bootstrapInPlace:
  installationDisk: /dev/sda
pullSecret: '<pull_secret>' 
sshKey: |
  <ssh_key> 
```

## Setup sno directory
Make the sno directory
```mkdir sno```
```cp install-config.yaml sno```
```./openshift-install --dir=sno create single-node-ignition-config```

## Copy the kubeconfig file to your config location
```
cp sno/auth/kubeconfig .kube/config
```

## Setup ISO for imaging
NOTE: Podman is needed for this, or replace podman with "docker".

Set the alias"
```
alias coreos-installer='podman run --privileged --pull always --rm \
        -v /dev:/dev -v /run/udev:/run/udev -v $PWD:/data \
        -w /data quay.io/coreos/coreos-installer:release' 
```
Bake the ignition into the ISO file you copied earlier:
```
coreos-installer iso ignition embed -fi sno/bootstrap-in-place-for-live-iso.ign scos-live.iso
```
## Boot your OKD Machine with the ISO
Attach the ISO to the machine and boot it. Assuming the disk is an sda type disk, it will start installing. This part typically takes upwards to 30 or so minutes. Once the install is complete.

Once complete, you should be able to see the node with the ```oc get nodes``` command.

If the above command works, congratulations! You setup a single-node OKD cluster!