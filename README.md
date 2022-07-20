# Setup a Kubernetes cluster using Ansible playbooks

## Main concepts
Kubernetes is an open source platform that automates Linux container operations. It eliminates many of the manual processes involved in deploying and scaling containerized applications. In other words, we can cluster together groups of hosts running Linux containers, and Kubernetes helps us easily and efficiently manage those clusters.

In this article, I'll use Docker as the containerization technology to create and use Linux containers.

Finally, the installation and configuration of the Kubernetes cluster will be performed through Ansible playbooks.

### Kubernetes main concepts
**Master**: The machine that controls Kubernetes nodes. This is where all task assignments originate.

**Node/Worker**: These machines perform the requested, assigned tasks. The Kubernetes master controls them.

**Pod**: A group of one or more containers deployed to a single node. All containers in a pod share an IP address, IPC, hostname, and other resources. Pods abstract network and storage away from the underlying container. This lets you move containers around the cluster more easily.

**Replication controller**:  This controls how many identical copies of a pod should be running somewhere on the cluster.

**Service**: This decouples work definitions from the pods. Kubernetes service proxies automatically get service requests to the right pod—no matter where it moves to in the cluster or even if it’s been replaced.

**Kubelet**: This service runs on nodes and reads the container manifests and ensures the defined containers are started and running.

**kubectl**: This is the command line configuration tool for Kubernetes.

### Ansible main concepts
**Control Node**: Any machine with Ansible installed. You can run commands and playbooks, invoking /usr/bin/ansible or /usr/bin/ansible-playbook, from any control node. You can use any computer that has Python installed on it as a control node - laptops, shared desktops, and servers can all run Ansible. However, you cannot use a Windows machine as a control node. You can have multiple control nodes.

**Managed Nodes**: The network devices (and/or servers) you manage with Ansible. Managed nodes are also sometimes called “hosts”. Ansible is not installed on managed nodes.

**Inventory**: A list of managed nodes. An inventory file is also sometimes called a “hostfile”. Your inventory can specify information like IP address for each managed node. An inventory can also organize managed nodes, creating and nesting groups for easier scaling.

**Modules**: The units of code Ansible executes. Each module has a particular use, from administering users on a specific type of database to managing VLAN interfaces on a specific type of network device. You can invoke a single module with a task, or invoke several different modules in a playbook.

**Tasks**: The units of action in Ansible. You can execute a single task once with an ad-hoc command.

**Playbooks**: Ordered lists of tasks, saved so you can run those tasks in that order repeatedly. Playbooks can include variables as well as tasks. Playbooks are written in YAML and are easy to read, write, share and understand.

## Environment
The environment built following this article is a cluster made up of three nodes, 1 master + 2 workers.
The playbooks have been tested for the following environment:

- VMs with OS CentOS 7

- Ansible version 2.9.10

- Kubernetes version 1.23.7

- Flannel as networking solution

### Sources
The playbooks used to build the cluster are available in this repository

### Create and bootstrap the nodes/workers
Before running the Ansible playbooks, we need to preconfigure the environment.

Update the /etc/hosts file in each server to provide a recognisable hostname:

**/etc/hosts**
```
10.0.2.18    master
10.0.2.19    worker1
10.0.2.20    worker2
```
Set the hostname permanently in each node:
```
$ sudo hostnamectl set-hostname [master|worker1|worker2]
```
Make sure the master and nodes have all access to Internet.

Master VM has to be created with two CPU at least.

In the Control Node (where Ansible is installed), create a hosts file to reflect the right IP addresses and hostnames:

**hosts**
```
[masters]
master ansible_host=10.0.2.18 ansible_user=root

[workers]
worker1 ansible_host=10.0.2.19 ansible_user=root
worker2 ansible_host=10.0.2.20 ansible_user=root
```

### Initial playbook
That’s the first playbook to run. It will create a user in the nodes to be used when running the cluster. The list of tasks that this playbook will performed is:

- Create the 'kubi' user

- Allow 'kubi' to have passwordless sudo

- Set up authorized keys for the kubi user

```
$ ansible-playbook -i hosts initial.yml -k
-i to specify the location of the hosts file

-k to prompt the SSH password

The output should be something like this:

SSH password:

PLAY [all] ********************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************
ok: [worker1]
ok: [worker2]
ok: [master]

TASK [create the 'kubi' user] *************************************************************************************************************************************************************************
changed: [worker2]
changed: [master]
changed: [worker1]

TASK [allow 'kubi' to have passwordless sudo] *********************************************************************************************************************************************************
changed: [worker1]
changed: [worker2]
changed: [master]

TASK [set up authorized keys for the kubi user] *******************************************************************************************************************************************************
changed: [worker1] => (item=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCdf/bhsenbPW2+043Gfa+YrDXMIrzTldc94UYcFxZjVlzHywgILwGBfYjadC25cykHfOt42s56uvmD469bug+nz4VXVs1JkJLEepdWLO8P6yGvxSrDJ4aLLUOD6wc2v5vhbrhfsV/5l7mZhYWOYtMRL9hU+M+ICZKHTTWfJQ22YKCgxKNOIAUkP5F0eRQB1AiI/ttpPKVVRpTWWiHWF7Ir381nNyYkP65hRXICow/s+SvfSnfurXRfv41ZWv8x3xQfhG5etEmKiAjDhKGiUQGxGzAguqhdLp7DmIKsf2JjsMGfQ+JYIvj9LjcKzqxGyY7WmZzdKpW/uh1gfl4qwgLN cmoya@local.trickynickel)
changed: [worker2] => (item=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCdf/bhsenbPW2+043Gfa+YrDXMIrzTldc94UYcFxZjVlzHywgILwGBfYjadC25cykHfOt42s56uvmD469bug+nz4VXVs1JkJLEepdWLO8P6yGvxSrDJ4aLLUOD6wc2v5vhbrhfsV/5l7mZhYWOYtMRL9hU+M+ICZKHTTWfJQ22YKCgxKNOIAUkP5F0eRQB1AiI/ttpPKVVRpTWWiHWF7Ir381nNyYkP65hRXICow/s+SvfSnfurXRfv41ZWv8x3xQfhG5etEmKiAjDhKGiUQGxGzAguqhdLp7DmIKsf2JjsMGfQ+JYIvj9LjcKzqxGyY7WmZzdKpW/uh1gfl4qwgLN cmoya@local.trickynickel)
changed: [master] => (item=ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCdf/bhsenbPW2+043Gfa+YrDXMIrzTldc94UYcFxZjVlzHywgILwGBfYjadC25cykHfOt42s56uvmD469bug+nz4VXVs1JkJLEepdWLO8P6yGvxSrDJ4aLLUOD6wc2v5vhbrhfsV/5l7mZhYWOYtMRL9hU+M+ICZKHTTWfJQ22YKCgxKNOIAUkP5F0eRQB1AiI/ttpPKVVRpTWWiHWF7Ir381nNyYkP65hRXICow/s+SvfSnfurXRfv41ZWv8x3xQfhG5etEmKiAjDhKGiUQGxGzAguqhdLp7DmIKsf2JjsMGfQ+JYIvj9LjcKzqxGyY7WmZzdKpW/uh1gfl4qwgLN cmoya@local.trickynickel)

PLAY RECAP ********************************************************************************************************************************************************************************************
master                     : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker1                    : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker2                    : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
In case you receive a message as follows:
```
fatal: [worker1]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this. Please add this host's fingerprint to your known_hosts file to manage this host."}
```
you have two options:

1. Either you disable the SSH key host checking, editing the /etc/ansible/ansible.cfg file and uncomment this line
```
host_key_checking = False
```
2. Or you add the host's fingerprint to the known_hosts file by SSHing into the master and workers from the ansible host, this prompts you to save the host's fingerprint to your known_hosts file:
```
promisepreston@ubuntu:~$ ssh myusername@192.168.43.240

The authenticity of host '192.168.43.240 (192.168.43.240)' can't be established.
ECDSA key fingerprint is SHA256:9Zib8lwSOHjA9khFkeEPk9MjOE67YN7qPC4mm/nuZNU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.43.240' (ECDSA) to the list of known hosts.

myusername@192.168.43.240's password:
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-53-generic x86_64)
```
### Dependencies playbook
That's the playbook we have to run to install all the dependencies. It will install Docker and the corresponding Kubernetes built-in tools on the different nodes. Besides, it will open the required ports in the firewall and apply some changes to the OS required to make Kubernetes work with no issues. Below, the complete list of tasks that will be performed in all the nodes:

- Install Docker

- Start and enable Docker service

- Disable SELinux

- Disable SELinux on reboot

- Disable SWAP

- Ensure net.bridge.bridge-nf-call-iptables is set to 1

- Disable firewalld (not recommended in Production environments)

- Add Kubernetes YUM repository

- Install kubelet

- Install kubeadm

- Start kubelet

- [ONLY IN MASTER] Install kubectl

### Running the playbook
From the Ansible Control Node, we have to run the following command
```
$ ansible-playbook -i hosts kube-dependencies.yml -k
-i to specify the location of the hosts file
```
The output should be something like this:
```
PLAY [all] ********************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************
ok: [worker1]
ok: [worker2]
ok: [master]

TASK [install Docker] *********************************************************************************************************************************************************************************
changed: [worker1]
changed: [worker2]
changed: [master]

TASK [Starting and Enabling Docker service] ***********************************************************************************************************************************************************
changed: [worker2]
changed: [master]
changed: [worker1]

TASK [disable SELinux] ********************************************************************************************************************************************************************************
changed: [worker2]
changed: [worker1]
changed: [master]

TASK [disable SELinux on reboot] **********************************************************************************************************************************************************************
[WARNING]: SELinux state change will take effect next reboot
changed: [worker2]
changed: [worker1]
changed: [master]

TASK [Disable SWAP since kubernetes can't work with swap enabled (1/2)] *******************************************************************************************************************************
changed: [worker2]
changed: [worker1]
changed: [master]

TASK [Disable SWAP in fstab since kubernetes can't work with swap enabled (2/2)] **********************************************************************************************************************
changed: [worker1]
changed: [worker2]
changed: [master]

TASK [ensure net.bridge.bridge-nf-call-ip6tables is set to 1] *****************************************************************************************************************************************
[WARNING]: The value 1 (type int) in a string field was converted to u'1' (type string). If this does not look like what you expect, quote the entire value to ensure it does not change.
changed: [worker2]
changed: [worker1]
changed: [master]

TASK [ensure net.bridge.bridge-nf-call-iptables is set to 1] ******************************************************************************************************************************************
changed: [worker1]
changed: [worker2]
changed: [master]

TASK [disable os firewall - firewalld] ****************************************************************************************************************************************************************
changed: [worker1]
changed: [worker2]
changed: [master]

TASK [add Kubernetes' YUM repository] *****************************************************************************************************************************************************************
changed: [worker2]
changed: [worker1]
changed: [master]

TASK [install kubelet] ********************************************************************************************************************************************************************************
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [install kubeadm] ********************************************************************************************************************************************************************************
changed: [worker1]
changed: [master]
changed: [worker2]

TASK [start kubelet] **********************************************************************************************************************************************************************************
changed: [master]
changed: [worker1]
changed: [worker2]

PLAY [master] *****************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************
ok: [master]

TASK [install kubectl] ********************************************************************************************************************************************************************************
changed: [master]

PLAY RECAP ********************************************************************************************************************************************************************************************
master                     : ok=22   changed=20   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker1                    : ok=20   changed=19   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker2                    : ok=20   changed=19   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
``` 
### Master playbook
This playbook is the third to run. It will perform some changes in the master node and initialize the cluster. The complete list of tasks is as follows:

- Reset kubeadm

- Initialise the cluster

- Create .kube directory

- Copy admin.conf to user’s kube config

- Install Pod network

#### Running the playbook
From the Ansible Control Node, we run the following command:
```
$ ansible-playbook -i hosts master.yml -k
i to specify the location of the hosts file
```
The output should be similar to this:
```
PLAY [master] *************************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [master]

TASK [Resetting kubeadm] **************************************************************************************************************************************************
changed: [master]

TASK [initialize the cluster] *********************************************************************************************************************************************
changed: [master]

TASK [Copying required files] *********************************************************************************************************************************************
[WARNING]: Consider using the file module with state=directory rather than running 'mkdir'.  If you need to use command because file is insufficient you can add 'warn:
false' to this command task or set 'command_warnings=False' in ansible.cfg to get rid of this message.
changed: [master]

TASK [install Pod network] ************************************************************************************************************************************************
changed: [master]

PLAY RECAP ****************************************************************************************************************************************************************
master                     : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
At this point, we can already check if the cluster has been initialised correctly. From the master’s node, we can run the following command:
```
$ sudo kubectl get nodes
The output would be like this:

NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   2m58s   v1.14.0
```
#### In case there is an error when initializing the cluster
In case of an error, after fixing the problem and before running again the master.yml playbook, you need to perform the following actions:
1. Reset the kubernetes cluster. From the master node, run this command
```
$ kubeadm reset -f
```
2. Remove all the data from all below locations
```
$ sudo rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/*
```
3. Delete the --cluster_initialized.txt-- file:
```
¢ sudo rm -f /root/cluster_initialized.txt
```
### Workers playbook
Fourth and last playbook to run. It will create a token in the master node, to be reused in the worker nodes when joining the cluster. The complete list of tasks is as follows:

- [ONLY IN MASTER] Create the token

- [ONLY IN MASTER] Set the token in a fact

- [ONLY IN WORKERS] Join the cluster

#### Running the playbook
From the Ansible Control Node, we have to run the following command:
```
$ ansible-playbook -i hosts workers.yml -k
i to specify the location of the hosts file

k to prompt the SSH password
```
The output would be similar to this:
```
PLAY [master] *************************************************************************************************************************************************************

TASK [get join command] ***************************************************************************************************************************************************
changed: [master]

TASK [set join command] ***************************************************************************************************************************************************
ok: [master]

PLAY [workers] ************************************************************************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************************************
ok: [worker2]
ok: [worker1]

TASK [join cluster] *******************************************************************************************************************************************************
changed: [worker2]
changed: [worker1]

PLAY RECAP ****************************************************************************************************************************************************************
master                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker1                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
worker2                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```
### Check the cluster
At this point, the cluster it’s been created and is up and running. To check this out, let’s run the following command from the master node:
```
$ kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master    Ready    master   16m     v1.19.0
worker1   Ready    <none>   4m49s   v1.19.0
worker2   Ready    <none>   4m52s   v1.19.0
```
### Deploy your first pod
We can deploy our first pod on the cluster. Let’s try something very simple, a Jira server instance. From the master node, we can run the following commands:
```
$ kubectl create deployment jira --image=atlassian/jira-software:8.5
deployment.apps/jira created

$ kubectl get pods
NAME                    READY   STATUS              RESTARTS   AGE
jira-77cdcdb4bd-pz8p2   0/1     ContainerCreating   0          2m

$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
jira   0/1     1            0           69s
It takes some minutes to get the pod ready

$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
jira-77cdcdb4bd-pz8p2   1/1     Running   0          6m35s

$ kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
jira   1/1     1            1           6m27s
```
To make accesible the pod, a service has to be created exposing the corresponding port. From the master node, run the following commands:
```
$ kubectl create service nodeport jira --tcp=8080:8080
service/jira created

$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jira         NodePort    10.109.22.190   <none>        8080:32258/TCP   86s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          118m
```
We can get the Flannel IP of the pod and the name of the node where it is running on. From the master node, run the following command:
```
$ sudo kubectl get pods --selector="app=jira" --output=wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
jira-77cdcdb4bd-pz8p2   1/1     Running   1          20h   10.244.1.3   worker2   <none>           <none>
```
Now, we can open a browser and navigate to http://[WORKER 2 IP ADDRESS]:32258 and the Jira Software setup page should be loaded!

## Setup the Kubernetes Dashboard
You can follow the instructions included in Web UI (Dashboard) . In short:

Deploy the dashboard UI in the Kubernetes cluster
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
```
Follow the instructions to create an user and generate an access token

Start the kubectl proxy to accept connections from any server:
```
$ kubectl proxy --address 0.0.0.0 --accept-hosts='^*$'
```
Open the browser and type the following URL
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
To retrieve the token for login, get the secret for the admin user:
```
$ kubectl get secret --namespace kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
admin-user-token-xpbtw             kubernetes.io/service-account-token   3      6d4h
default-token-vm5vc                kubernetes.io/service-account-token   3      6d4h
kubernetes-dashboard-certs         Opaque                                0      6d4h
kubernetes-dashboard-csrf          Opaque                                1      6d4h
kubernetes-dashboard-key-holder    Opaque                                2      6d4h
kubernetes-dashboard-token-dpljw   kubernetes.io/service-account-token   3      6d4h
```
Run the following command and copy/paste the output:
```
$ kubectl get secret admin-user-token-xpbtw --namespace kubernetes-dashboard -o jsonpath={.data.token} | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IkFGQmp1blFp.....
```