# Get, run and contribute #
## Get the code ##

Requirement: This code is using ansible to bootstrap and update kerbernetes environment. Please ensure that ansible is installed on the machine you will run the code.

The code could be retrieved from github using the following command
```
git clone https://github.com/cornelius-keller/ansible-coroeos-kubernetes
```
You should now retrieve the submodules used by this code.

```
#go to the source repository
cd ansible-coreos-kubernetes
git submodule init
git submodule update
```

Finally the current code depends on ansible-coreos-bootstrap module which requires the following command to be run

```
ansible-galaxy install sigma.coreos-bootstrap
```

> be aware that this command should be run with root privileges to update /etc/ansible/roles


You are now ready to use the application to bootstrap a new cluster and later to update it.

## Bootstrap a cluster ##

Bootstraping a cluster will install coreos on each node of the cluster. After coreos installation, boostrapping will install and configure ceph, etcd, docker, kubernetes with kubectl.
Bootstrap is performed using the ansible playbook bootstrap_coreos.yml.

### Preparation ###

Before bootstraping you need to get at least three dedicated servers.
Curently hetzner, ovh and kimsufi should be supported. Deployment on lacal servers should be also supported. Please refer to Supported providers for detailed informations.

Get the IP adresses of all your node and start to create a new inventory.ini file.
A sample ini file is provided on the root folder.

#### Create a new ini file ####

1. Edit the etcd-node and set the IP address and the hostname for all your nodes. It is recommend to strat with three etcd nodes and no etc-proxy.
2. Edit kubernetes-master section and set the IP adresses. For high Availability it is recommended to have 2 master node.
3. Edit kubernetes-node and set the IP addresses. Adresses of nodes that should not be kubernetes master.
4. Edit the ceph-mon section and set the IP adresses. IP Adresses of ceph nodes that will act as Monitor. Ceph recommend to have an ood number of monitors. Therefore for high availablity it is recommend to have 3 monitors (not more as monitors generate some overload)
5. Edit the ceph-osd section and set IP adresses. ceph osd nodes will act as storage nodes for ceph.
6. Edit the all:vars section and set:
	a. the kube_master (FQDN) dns name [Optional]
	b. the kube_master IP address (being one of the IP defined in kubernetes-master section)
	c. the kube_cluster_name if needed
	d. change baremetal_provider and set webservice username and password if needed.
	e. change python_interpreter if needed
    > REMARK: why ansible_python_interpreter is also defined in some playbook

	f. set the ceph_fsid with a new UUID. UUID can be generated with the uuidgen shell command.
	g. set the ceph_key with key generated usinf the ceph-key.py script provided
	h. set ceph osd type and parameters. Currently there is no ceph_osd_type supported:
	- type is directory which mean that osd will use a directory for its partition. You must ensure that the following parameters are set in the ini file:
    ceph_osd_type=os_directory
	ceph_osd_path= ** path to a local dir **
	* type is disk which means that a fill disk will be used for osd partition. You must ensure that the following parameters are set in the ini file:
        ceph_osd_type=os_disk
		ceph osd drive= **path to a device (e.g. /dev/sdb)**

	i. Finally set the parameters specific to your provider (Please refer to the provider section for detailed informations on how to get username, password and fingerprint)
    - set the webservice username
    - set the webservice password
    - set the fingerprint public ssh key to be used in rescue mode.

#### etcd-ca #####

the bootstrap process requires to have etcd-ca tool installed. Yoiu can validate etcd-ca installation by using the following command

```
which etcd-ca
```
If you have no ouput you must perform the following steps otherwhise you can go directly to next step:

To build etcd-ca:

```
$ git clone https://github.com/coreos/etcd-ca
$ cd etcd-ca
$ ./build
```
this will build etcd-ca in the ./bin folder.
ensure that this etcd-ca is in the path or create a symbolic link to have it in the path.


### Supported providers ###
#### Hetzner ####
Find below some spefic steps required to use this playbook with hetzner provider

##### upload ýour public key to Hetzner
* Log into the the Hetzner robot.
* Navigate to the server list in the left menu.
![hetzner server list](assets/hetzner_server_list.png "Hetzner Server List")

* chose key management
  ![hetzner key management](assets/hetzner_key_management.png "Logo Key Managemet")
* chooose new key, give it a name, and copy the output of

  `cat ~/.ssh/id_rsa.pub`

 into the form and save it.
 ![hetzner new key](assets/hetzner_add_key.png)

* copy the id of the key for use in the inventory. ![hetzer key list](assets/hetzner_key_list.png)

##### Create a webservice account
* create a robot account and copy the username and password into the inventory.
![hetzner robot account](assets/hetzner_webservice_user.png)

  For Example:

      hetzner_webservice_username=#ws+7FPjagF7
      hetzner_webservice_password=<your password>

This webservice username, password and the fingerprint of the rescue key to be provided in the hetzner section of the inventory.ini

#### OVH ####
> TO BE DONE


### Launch bootstrap process ###

Ensure that you run ansible version 2.1 or greater. You can check it using the command:

```
ansible --version
```

Launch the bootstrap process using the following command:

```
ansible-playbook -i **path to your inventory file** bootstrap_coreos.yml
```

the playbook should finish without any errors.

### Check your new cluster ###

if your installation went well you should be able to get status of your cluster form your machine doing:

```

```

#### 1. Check ssh connection ####
You should be able to connect to each node of your cluster using ssh without password (authentication is done using ssh key). Use the following command to check:

```
	ssh core@**ip_address of your node**
```

**Expected result:** You should be able to connect to each of your node as core without password.

#### 2. Check status of essentials services
Connect to your nodes and check the status of the following services:

* etcd2
```
  systemctl status -l etcd2 --no-pager
```
**Expected result:** You should get the following output
```
etcd2.service - etcd2
   Loaded: loaded (/usr/lib/systemd/system/etcd2.service; disabled; vendor preset: disabled)
  Drop-In: /run/systemd/system/etcd2.service.d
           └─20-cloudinit.conf
        /etc/systemd/system/etcd2.service.d
           └─30-certificates.conf, 50-network-wait.conf
   Active: active (running) since Wed 2017-01-04 13:45:20 UTC; 1 day 3h ago
 Main PID: 1317 (etcd2)
    Tasks: 16
   Memory: 43.8M
      CPU: 10min 17.189s
   CGroup: /system.slice/etcd2.service
           └─1317 /usr/bin/etcd2
```
for this output you can validate that the service is up and running
* fleet
```
  systemctl status -l fleet --no-pager
```
**Expected result:** You should get the following output
```
● fleet.service - fleet daemon
   Loaded: loaded (/usr/lib/systemd/system/fleet.service; disabled; vendor preset: disabled)
  Drop-In: /run/systemd/system/fleet.service.d
           └─20-cloudinit.conf
   Active: active (running) since Sun 2017-01-08 06:16:25 UTC; 22min ago
 Main PID: 5974 (fleetd)
    Tasks: 5
   Memory: 9.3M
      CPU: 94ms
   CGroup: /system.slice/fleet.service
           └─5974 /usr/bin/fleetd
```

* kube api server
```
  systemctl status -l kube-apiserver.service --no-pager
```
**Expected result:** You should get the following output
```
● kube-apiserver.service - Kubernetes API Server
   Loaded: loaded (/etc/systemd/system/kube-apiserver.service; static; vendor preset: disabled)
   Active: active (running) since Sun 2017-01-08 10:53:52 UTC; 1min 44s ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 28752 (apiserver)
    Tasks: 13
   Memory: 54.1M
      CPU: 4.478s
   CGroup: /system.slice/kube-apiserver.service
           └─28752 /apiserver #--service-account-key-file=/opt/bin/kube-serviceaccount.key --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem --service-account-lookup=false --admission-control=NamespaceLifecycle,NamespaceAutoProvision,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota --runtime-config=api/v1 --allow-privileged=true --insecure-bind-address=127.0.0.1 --bind-address=xx.xx.xx.xx --insecure-port=8080 #--etcd-config=/etc/etcd-client.config.json --etcd-cafile=/etc/ssl/etcd/ca.crt --etcd-certfile=/etc/ssl/etcd/key.crt --etcd-keyfile=/etc/ssl/etcd/key.key --etcd-servers=https://xx.xx.xx.xx:2379 --kubelet-https=true --secure-port=6443 --runtime-config=extensions/v1beta1/daemonsets=true --service-cluster-ip-range=10.100.0.0/16 --ssh-keyfile=/etc/kubernetes/ssh/id_rsa --ssh-user=kubernetes #--token-auth-file=/srv/kubernetes/known_tokens.csv #--basic-auth-file=/srv/kubernetes/basic_auth.csv #--etcd-servers=http://127.0.0.1:2379 --logtostderr=true


```
* kube control manager
```
systemctl status -l --no-pager kube-controller-manager.service
```
**Expected result:** You should get the following output
```
● kube-controller-manager.service - Kubernetes Controller Manager
   Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; static; vendor preset: disabled)
   Active: active (running) since Sun 2017-01-08 11:15:52 UTC; 38min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 4317 (controller-mana)
    Tasks: 13
   Memory: 685.7M
      CPU: 34.222s
   CGroup: /system.slice/kube-controller-manager.service
           └─4317 /controller-manager --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --master=127.0.0.1:8080 --leader-elect=true --leader-elect-lease-duration=15s --leader-elect-renew-deadline=10s --leader-elect-retry-period=2s --logtostderr=true
```
* kube scheduler
```
systemctl status -l --no-pager kube-scheduler.service         
```
**Expected result:** You should get the following output
```
● kube-scheduler.service - Kubernetes Scheduler
   Loaded: loaded (/etc/systemd/system/kube-scheduler.service; static; vendor preset: disabled)
   Active: active (running) since Sun 2017-01-08 11:15:52 UTC; 42min ago
     Docs: https://github.com/GoogleCloudPlatform/kubernetes
 Main PID: 4335 (scheduler)
    Tasks: 13
   Memory: 24.5M
      CPU: 15.495s
   CGroup: /system.slice/kube-scheduler.service
           └─4335 /scheduler --master=127.0.0.1:8080 --leader-elect=true --leader-elect-lease-duration=15s --leader-elect-renew-deadline=10s --leader-elect-retry-period=2s
```
#### 3. Check etcd service ####
Connect to one of your node using ssh and run the following command

```
  etcdctl cluster-health
```
**Expected result:** You should get a list of all the etcd cluster member

```
member 1bf9eb3de326f477 is healthy: got healthy result from https://xx.xx.xx.xx:2379
member cd4c58ff55c64bf0 is healthy: got healthy result from https://xx.xx.xx.xx:2379
member ea7f964fa5adb1f1 is healthy: got healthy result from https://xx.xx.xx.xx:2379
cluster is healthy
```

#### 4. Check fleet service ####

### Troubleshooting common issues
#### Debugging cloud-config file
During the bootstarp process of your cluster a cloud-config file is created by the cloud-config task and pushed on your nodes as /var/lib/coreos/user-data.
This file is used by coreos to configure and setup your cluster.
This file could be one of the main source of issues for your cluster.

* Validate synthax of cloud-config file using teh command

```
  sudo coreos-cloudinit -validate --from-file /var/lib/coreos-install/user_data
```

**Expected result:** You should get the following output

```
2017/01/05 17:51:25 Checking availability of "local-file"
2017/01/05 17:51:25 Fetching user-data from datasource of type "local-file"
```
