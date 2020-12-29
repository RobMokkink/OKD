# OKD Single Node Lab

This repo contains information and pointer about how to setup your OKD lab for study purposes.
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

If you make changes while running VM's, always use ```sudo systemctl reload NetworkManager```, do not do a ```sudo systemctl restart NetworkManager```
Also make sure that ```/etc/nsswitch.conf``` is configure correctly, this must be set to ```hosts:       dns files```, otherwise you will get issues with the wildcard dns ingress domains.

## HA Proxy
Haproxy needs to be configured see the ```haproxy.cfg``` example. When running you can see status by pointing your brower to the ```http://<FQDN of your haproxy>:8443/stats``` 


## Webserver to host ignition files
You can install for example a apache webserver on the loadbalancer and make sure you have it listen on a different port, for example ```8080```.


## Install config
See the OKD documentation about the required tools etc, see https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html
See the example ```install-config.yaml```, we only have 1 master node. You will need to adjust the basedomain, [pullsecret](https://cloud.redhat.com/openshift/install/pull-secret) and ssh keys.
Create the manifests files:
```
$ openshift-install create manifests
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 
WARNING Discarding the Openshift Manifests that was provided in the target directory because its dependencies are dirty and it needs to be regenerated 
```

Ignore the WARNINGS!

Create the iginition files:
```
$ openshift-install create ignition-configs
INFO Consuming Openshift Manifests from target directory 
INFO Consuming Master Machines from target directory 
INFO Consuming Worker Machines from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming OpenShift Install (Manifests) from target directory 
```

Copy the iginition files to our webserver in this case it's also the loadbalancer running on port 8080:

```
scp *ign <username>@<loadbalancer>:/var/www/html
```

Make sure the files are readable on the web service

```
chmod 744 /var/www/html/*.ign
```

Test it by testing the url ```<fqdn loadbalancer>:8080/bootstrap.ign``` 


## Create bootstrap node and master node
Create the bootstrap node (2 cpu's, 8gb of ram, 120gb disk), you can create a dhcp reservation, or use nmcli/nmtui when it is booted up.
I prefer to create the dhcp reservation before the vm is created, i do this with the following command:

```
sudo virsh net-update <name of kvm network> add ip-dhcp-host "<host mac='<mac address>' name='<name of host>' ip='<ip address>'/>" --live --config
```

Real example:

```
sudo virsh net-update lab add ip-dhcp-host "<host mac='52:54:00:fd:66:1c' name='coreb' ip='10.0.0.40'/>" --live --config
```

Once the system is booted i prefer to set the hostname and ip adress correctly with ```sudo nmtui```

Then do the following:

```
sudo coreos-install install /dev/vda --copy-network --ignition-url=http://<FQDN>:8080/bootstrap.ign --insecure-igntion```
```

Then reboot the node.

Look at the ```http://<FQDN of your haproxy>:8443/stats``` and wait until api and machine-config service are up and running.
Or login with the user ```core`` like so ```ssh core@<fqdn of bootstrap node``` and run the command as shown:

```
journalctl -b -f -u release-image.service -u bootkube.service
```


Then create the master node (4 cpu's, 16gb of ram, 120gb disk)

Once the system is booted i prefer to set the hostname and ip adress correctly with ```sudo nmtui```
Or create a dhcp reservation like stated above with ```virsh``` command.

Then do the following:

```
sudo coreos-install install /dev/vda --copy-network --ignition-url=http://<FQDN>:8080/master.ign --insecure-igntion```
```

Then reboot the node.


## Bootstap fase

Run the following command to see how your bootstrap is progressing(this is also mentioned in the documentation)

```
$ openshift-install wait-for bootstrap-complete
INFO Waiting up to 20m0s for the Kubernetes API at https://api.okd.lab.local:6443... 
INFO API v1.18.3 up                               
INFO Waiting up to 40m0s for bootstrapping to complete... 
```

The bootstrap fase is finished when you see this:

```
INFO It is now safe to remove the bootstrap resources 
INFO Time elapsed: 7m50s                          
```

## Remove bootstrap from haproxy config
Remove the bootstrap node from ha proxy and reload (not restart) the haproxy service like so:

```
sudo systemctl reload haproxy
```

When you look at the status page of the loadbalancer you will notice the bootstrap node is gone.

## Wait for install to complete fase
Run the following command (also mentioned in the documentation) to see if the installation is completed:

```
$ openshift-install wait-for install-complete
INFO Waiting up to 30m0s for the cluster at https://api.okd.lab.local:6443 to initialize...
```

To see how the build-in operators are progressing i suggest you do the following:

```
export KUBECONFIG=auth/kubeconfig
```

```
watch -n5 'oc get co'
```

This way you will see how the operators are getting ready.

After a while you will see this message:
```
INFO Waiting up to 10m0s for the openshift-console route to be created... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/rob/Documents/training/openshift/labs/okd/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.okd.lab.local 
INFO Login to the console with user: "kubeadmin", and password: "whiskytime" 
INFO Time elapsed: 6m1s                           
```

Congratulations, you just installed a OKD cluster!

## After
After the installation is done you will have 1 master node, you can add more master nodes afterwards, by booting them pointing to the igintion files and approve the CSR's.
But that is up to you.

## Tip
If you shutdown the cluster node with ```ssh core@<node fqdn>  "sudo shutdown -h 1"``` and boot it up later and the services are not getting up, make sure to approve the CSR's, see the following [documentation](https://docs.openshift.com/container-platform/4.6/backup_and_restore/graceful-cluster-restart.html)
