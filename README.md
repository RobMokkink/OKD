# OKD Lab
This repo contains information and point about how to setup your OKD lab for study purposes.
At first this lab will consist of a single master OKD node, which you can expand with more nodes.

## Prerequisites
- laptop/workstion with at least 32gb of ram, more is better!
- linux distro preferably Fedora or Ubuntu.
- KVM as hypervisor
- DNSMasq plugin enabled in networkmanager, see https://fedoramagazine.org/using-the-networkmanagers-dnsmasq-plugin/
- linux vm with haproxy
- webserver to host ignition files
- Fedora CoreOS stable iso, see https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable

##  Network
Enable dnsmasq plugin in NetworkManager, see the above link.
Create DNS entry's for the nodes, api, api-int and wildcard ingress domain. Note that for api, api-int and wildcard ingress domain, this is set to the haproxy loadbalancer.


```/etc/hosts```

```
10.0.0.10   lb.lab.local lb api.okd.lab.local api-int.okd.lab.local
10.0.0.40   coreb.lab.local coreb
10.0.0.41   corem1.lab.local corem1
```

```/etc/NetworkManager/dnsmasq.d/okd.conf```

```
address=/.apps.okd.lab.local/10.0.0.10
```

If you make changed while running VM's, always use ```sudo systemctl reload NetworkManager```, do not do a ```restart```

## HA Proxy
Haproxy needs to be configured see the ```haproxy.cfg``` example. When running you can see status by pointing your brower to the ```http://<FQDN of your haproxy>:8443/stats``` 


## Webserver to host ignition files
You can install for example a apache webserver on the loadbalancer and make sure you have it listen on a different port, for example ```8080```.


## Install config
See the OKD documentation about the required tools etc, see https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html
See the example ```install-config.yaml```, we only have 1 master node. You will need to adjust the basedomain, pullsecret and ssh keys.
Create the manifests and ignition files, ignore the warnings. Then copy over the ignition files to your webserver, make sure they are readable!


## Create bootstrap node and master node
Create the bootstrap node (2 cpu's, 8gb of ram, 120gb disk), you can create a dhcp reservation, or use nmcli/nmtui when it is booted up.

Once the system is booted i prefer to set the hostname and ip adress correctly with ```sudo nmtui```

Then do the following:

```
sudo coreos-install install /dev/vda --copy-network --ignition-url=http://<FQDN>:8080/bootstrap.ign --insecure-igntion```
```

Then reboot the node.

Look at the ```http://<FQDN of your haproxy>:8443/stats``` and wait until api and machine-config service are up and running.

Create the master node (4 cpu's, 16gb of ram, 120gb disk)

Once the system is booted i prefer to set the hostname and ip adress correctly with ```sudo nmtui```

Then do the following:

```
sudo coreos-install install /dev/vda --copy-network --ignition-url=http://<FQDN>:8080/master.ign --insecure-igntion```
```

Then reboot the node.

## After
After the installation is done you will have 1 master node, you can add more master nodes afterwards, by booting them pointing to the igintion files and approve the CSR's.
But that is up to you.
