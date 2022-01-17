# OKD (Open-Source OpenShift) UPI install with Fedora COREOS & vSphere 6.7U3

### After mastering various installs of OCP, I tried my hand at OKD and quickly learned it's not the same. Follow along in this script for your install.

### Prereq's
1. Configure an external load balancer for the api.<cluster>.<domain>.<tld>:6443 interface and direct this port to the three master (control-plane) nodes.
2. Setup DNS to resolve the api.<cluster>.<domain>.<tld> & api-int.<cluster>.<domain>.<tld> external interface. \
    In my setup, I used an internal IP .19 and pointed both api & api-int to it (api.okd.redcloud.land & api-int.okd.redcloud.land)
3. Unlike OCP, your loadbalancer must have a healthcheck on port 1936 or the install will fail. Use the provided haproxy.conf as a working example from the OKD documentation [Load balancing requirements for user-provisioned infrastructure.](https://docs.okd.io/4.9/installing/installing_vsphere/installing-vsphere.html#installation-load-balancing-user-infra_installing-vsphere)

As a quick side note, I attempted to install OKD using the guide and downloaded the Fedora CoreOS 3.4 & 3.5, along with the approprate [okd tools](https://github.com/openshift/okd/releases) per their version. Each attempt failed. When I attempted an IPI install it was successful. In order to make my okd UPI install work, I removed the IPI install except for the downloaded template. Using this template, I was able to clone my VM's and have a successful install. This tells me they pay more attention to IPI than any other install... if you become frustrated with the documented install, this may be an option for you to attempt as well.

4. Be familure with the [installation documentation](https://docs.okd.io/4.9/architecture/architecture-installation.html#installation-overview_architecture-installation), lots of good stuff here regarding the above prereq's. 
5. You must have an http service running that we can pull the bootstrap.ign file from. This file is too large for VMware to imbed so we have to use another file to point to the http server file for extraction.

### Configure your install for UPI on VMware

1. Download the openshift-install and client tools, extract them into either /usr/local/bin or somewhere else that you'd like to referece from. I put everthing into /usr/local/bin.
2. Generate your install-config.yaml\
a. If this is your first attempt, I recommend running throught the install script using `openshift-install create install-config --dir=.` as it will verifiy you can successfully connect to your VMware vCenter as well as configure the pull-secret and ssh access for you.\
b. If you already have a install-config.yaml, proceed with step 3.
3. Let's modify the install-config.yaml\
a. Set the worker replica's to "0" as we'll join these manually.\
b. If you followed 2/a above, remove your IPI generated entries for API and the Ingress Controller.\

### Configure your installation files
1. Create your manifests:\
`openshift-install create manifests --dir=$BUILD_DIR`
2. Edit $BUILD_DIR/manifests/cluster-scheduler-02-config.yml and replace 'true' with 'false' so the master nodes will not run application pods.
3. Remove machinesets since we're building these manualy:\
`rm -f $BUILD_DIR/openshift/99_openshift-cluster-api_master-machines-*.yaml openshift/99_openshift-cluster-api_worker-machineset-*.yaml`
4. Generate your ignition files:\
`openshift-install create ignition-configs --dir=$BUILD_DIR`
5. Copy your bootstrap.ign file to your http server and make sure it's globally readable. By default, the file is 640, we need this 644 or just run `"chmod +r bootstrap.ign"`.
6. Create your bootstrap fetch file that points to your bootstrap.ign, I named this file append-bootstrap.ign:
    ```
    {
    "ignition": {
        "config": {
        "merge": [
            {
            "source": "http://{$HTTPSERVERIP}/bootstrap.ign", 
            "verification": {}
            }
        ]
        },
        "timeouts": {},
        "version": "3.2.0"
    },
    "networkd": {},
    "passwd": {},
    "storage": {},
    "systemd": {}
    }
    ```
7. Let's encode our ign files, I'm using a Mac your way may be different:\
    `base64 -i append-bootstrap.ign -o append-bootstrap.64`\
    `base64 -i $BUILD_DIR/master.ign -o master.64`\
    `base64 -i $BUILD_DIR/worker.ign -o worker.64`
8. Clone your vm-template to their VM's. We'll add some settings to the advanced options. Under "Customize hardware" tab, click "VM Options", then scroll down and select "Advanced" and then at "Configuration Parameters" select "EDIT CONFIGURATION...". Since I'm including static IP's, I'll have 4 entries.\
a. disk.EnableUUID = TRUE\
b. guestinfo.ignition.config.data.encoding = base64\
c. guestinfo.ignition.config.data = $YOUR_ENCODED_DATA\
d. guestinfo.afterburn.initrd.network-kargs = ip=<ip>::<gateway>:<netmask>:<hostname>:<iface>:none nameserver=srv1 [nameserver=srv2 [nameserver=srv3 [...]]]\

### Start the VM's
1. Boot up the bootstrap machine first, confirm in your http logs that the bootstrap.ign file was accessed. The vm will pull down a bunch of containers and reboot. Once the VM is back online, proceed with the next step.
2. Boot all 3 master vm's, likewise they will pull down a bunch of containers and form the cluster. I generally watch one of the node consoles, once it reboot's, I will export the kubeconfig and start monitoring the status of the masters.\
a. Export the kubeconfig:\
`export KUBECONFIG=$PATH_TO_BUILD_DIR/$BUILD_DIR/auth/kubeconfig`\
b. Monitor the master status:\
`oc get nodes`
3. Once the master nodes are in a "Ready" state, boot up the worker nodes. Like everything above, the nodes will pull their containers and reboot. When the machines are back online, we'll need to approve their certificates in order to join the cluster.\
a. Check for Pending cert requests:\
`oc get csr | grep Pending`\
b. Approve all CSR's, keep checking until no further CSR's are in a Pending state.\
`oc adm certificate approve $CSR $CSR $CSR $CSR... `
4. Monitor the operators for a ready state... this will take about 20mins.\
`oc get co`

Now your cluster is online, your kubeadmin password is under $BUILD_DIR/auth/kubeadmin-password; this will give you access to the UI console. If you don't have the route, you can pull it with:\
 `oc get route -n openshift-console`

Login with username kubeadmin, and your kubeadmin-password... now you're ready to configure your OKD (OpenShift) Platform.
