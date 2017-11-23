# Introduction

Express Containers with Docker Enterprise Edition (EE) is a complete solution from Hewlett Packard Enterprise that includes all the hardware, software, professional services, and support you need to deploy a containers-as-a-service (CaaS) platform, allowing you to get up and running quickly and efficiently. The solution takes the HPE hyperconverged infrastructure and combines it with Docker’s enterprise-grade container platform, popular open source tools, along with deployment and advisory services from HPE Pointnext.

Express Containers with Docker EE is ideal for customers migrating legacy applications to containers, transitioning to a container DevOps development model or needing a hybrid environment to support container and non-containerized applications on a common VM platform.  Express Containers with Docker EE provides a solution for both IT development and IT operations, and comes in two versions.  The version for IT developers (Express Containers with Docker EE: Dev Edition) addresses the need to provide a cloud-like container development environment with built-in container tooling.  The version for IT operations (Express Containers with Docker EE: Ops Edition) addresses the need to have a production ready environment that is very easy to deploy and manage.  

This document describes the best practices for deploying and operating the Express Containers with Docker EE: Dev Edition. It describes how to automate the provisioning of the environment using a set of Ansible playbooks. It also outlines a set of manual steps to harden, secure and audit the overall status of the system. A corresponding document focused on setting up Express Containers with Docker: Ops Edition is also available. 

**Note**
- The Ansible playbooks described in this document are only intended for Day 0 deployment automation of Docker EE on SimpliVity
- The Ansible playbooks described in this document are not directly supported by HPE and are intended as an example of deploying Docker EE on HPE SimpliVity.  We welcome input from the user community via Github to help us prioritize all future bug fixes and feature enhancements

## About Ansible

Ansible is an open-source automation engine that automates software provisioning, configuration management and application deployment.

As with most configuration management software, Ansible has two types of server: the controlling machine and the nodes. A single controlling machine orchestrates the nodes by deploying modules to the nodes over SSH. The modules are temporarily stored on the nodes and communicate with the controlling machine through a JSON protocol over the standard output. When Ansible is not managing nodes, it does not consume resources because no daemons or programs are executing for Ansible in the background. Ansible uses one or more inventory files to manage the configuration of the multiple nodes in the system.

In contrast with other popular configuration management software such as Chef, Puppet, and CFEngine, Ansible uses an agentless architecture. With an agent-based architecture, nodes must have a locally installed daemon that communicates with a controlling machine. With an agentless architecture, nodes are not required to install and run background daemons to connect with a controlling machine. This type of architecture reduces the overhead on the network by preventing the nodes from polling the controlling machine.

More information about Ansible can be found here: [http://docs.ansible.com](http://docs.ansible.com)


## About Docker Enterprise Edition

Docker Enterprise Edition (EE) is the leading enterprise containers-as-a-service (CaaS) software platform for IT that manages and secures diverse applications across disparate infrastructure, both on-premises and in the cloud. Docker EE provides integrated container management and security from development to production. Enterprise-ready capabilities like multi-architecture orchestration and secure software supply chain give IT teams the ability to manage and secure containers without breaking the developer experience.

Docker EE provides

- Integrated management of all application resources from a single web admin UI.
- Frictionless deployment of applications and Compose files to production in a few clicks.
- Multi-tenant system with granular role-based access control (RBAC) and LDAP/AD integration.
- Self-healing application deployment with the ability to apply rolling application updates.
- End-to-end security model with secrets management, image signing and image security scanning.


More information about Docker Enterprise Edition can be found here: [https://www.docker.com/enterprise-edition](https://www.docker.com/enterprise-edition)



## About Simplivity

Simplivity is an enterprise-grade hyper-converged platform uniting best-in-class data services with the world's best-selling server.

Rapid proliferation of applications and the increasing cost of maintaining legacy infrastructure causes significant IT challenges for many organizations. With HPE SimpliVity, you can streamline and enable IT operations at a fraction of the cost of traditional and public cloud solutions by combining your IT infrastructure and advanced data services into a single, integrated solution. HPE SimpliVity is a powerful, simple, and efficient hyperconverged platform that joins best-in-class data services with the world’s best-selling server and offers the industry’s most complete guarantee.

More information about Simplivity can be found here: [https://www.hpe.com/us/en/integrated-systems/simplivity.html](https://www.hpe.com/us/en/integrated-systems/simplivity.html)

**Target Audience:** This document is primarily aimed at technical individuals working in the development side of the pipeline, such as technical architects, developers and system administrators, but anybody with an interest in automating the provisioning of virtual servers and containers may find this document useful.

**Assumptions:** This document assumes a minimum understanding of concepts like virtualization, containerization and some knowledge around Linux and VMWare technologies.

## Required Versions

The following software versions were used to implement the playbooks that are described in later sections. Other version may work but have not been tested.

- Ansible 2.2 and 2.3. Please note that the playbooks will not work with Ansible 2.4 due to an open defect https://github.com/ansible/ansible/issues/32000. Do not use Ansible 2.4 until this defect is fixed.
- Docker EE 17.06 (tested with UCP 2.2.3 and 2.2.4 and DTR 2.4.0)
- Red Hat Enterprise Linux 7.3 and 7.4
- VMWare ESXi 6.5.0 and vCenter 6.5.0
- HPE SimpliVity OmniStack 3.7.1.60


# Architecture

The development environment is comprised of two HPE SimpliVity 380 Gen10 servers. HPE recommends dual socket SimpliVity systems with at least 14 CPU cores per socket (28 total cores per system) for optimal performance and support during HA failover scenario. Please refer to Appendix A for a sample BOM.  Since the SimpliVity technology relies on VMware virtualization, the servers are managed using vCenter. The load among the two hosts will be shared as per Figure 1.

Uptime is paramount for any users implementing Docker containers in business critical environments. Express Containers offers various levels of high availability (HA) to support continuous availability. All containers including the Docker system containers are protected by Docker’s swarm mode. Swarm mode can protect against individual hardware, network, and container failures based on the user’s declarative model. 
Express Containers with Docker EE also deploys load balancers in the system to help with container traffic management. There are three load balancer VMs – UCP load balancer, DTR load balancer, and Docker worker node load balancer. Since these load balancers exist in VMs, they have some degree of HA but may incur some downtime during the restoration of these VMs due to a planned or unplanned outage. For optimal HA configuration, the user should consider implementing a HA load balancer architecture using the Virtual Router Redundancy Protocol (VRRP). For more information see http://www.haproxy.com/solutions/high-availability/. 


![Solution Architecture][dev-architecture]
**Figure 1.** Solution Architecture

The Ansible playbooks can be modified to fit your environment and your high availability (HA) needs. By default, the Ansible Playbooks will set up a 2 node environment. HPE and Docker recommend a minimal starter configuration of 2 physical nodes for running Docker in a development environment.

The distribution of the Docker and non-Docker modules over the 2 physical nodes via virtual machines (VMs) is as follows:

- 1 Docker Universal Control Plane (UCP) VM node
- 1 Docker Trusted Registry (DTR) VM node
- 2 Docker Swarm worker VM nodes for container workloads
- 1 Docker UCP load balancer VM to ensure access to UCP in the event of a node failure
- 1 Docker DTR load balancer VM to ensure access to DTR in the event of a node failure
- 1 Docker Swarm Worker node VM load balancer
- 1 Logging server VM for central logging 
- 1 NFS server VM for storage Docker DTR images

In addition to the above, the playbooks also set up:

- Docker persistent storage driver from VMware
- Elasticsearch, Logstash, and Kibana (ELK) stack
- Play with Docker (PWD)
- CloudBees Jenkins
- SimpliVity backup policy for data volumes and for the NFS storage used by DTR for storing Docker images

These nodes can live in any of the hosts and they are not redundant. The vSphere Docker volume plug-in stores data in a shared datastore that can be accessed from any machine in the cluster.

![Solution stack][solutionstack]  
**Figure 2.** Solution stack


# Sizing considerations

A node is a machine in the cluster (virtual or physical) with Docker Engine running on it. When provisioning each node, assign it a role: UCP Controller, DTR, or worker node so that they are protected from running application workloads.

To decide what size the node should be in terms of CPU, RAM, and storage resources, consider the following:

1. All nodes should at least fulfil the minimal requirements, for UCP 2.0 2GB of RAM and 3GB of storage. More detailed requirements are in the UCP documentation.
2. UCP Controller nodes should be provided with more than the minimal requirements, but won’t need much more if nothing else runs on them.
3. Ideally, Worker node size will vary based on your workloads so it is impossible to define a universal standard size.
4. Other considerations like target density (average number of containers per node), whether one standard node type or several are preferred, and other operational considerations might also influence sizing.

If possible, node size should be determined by experimentation and testing actual workloads, and they should be refined iteratively. A good starting point is to select a standard or default machine type in your environment and use this size only. If your standard machine type provides more resources than the UCP Controllers need, it makes sense to have a smaller node size for these. Whatever the starting choice, it is important to monitor resource usage and cost to improve the model.

For Express Containers with Docker EE: Dev Edition, the following section describes sizing configurations. The vCPU allocations are described in Table 1 while the memory allocation is described in Table 2. 



**Table 1.** vCPU

| vCPUs | simply01 | simply02 | 
|:------|:--------:|:--------:|
| ucp1  |	4      |          |
| dtr1  | 2		   |          |
| worker1 |4	   |          |
| worker2| 	       | 	4	  | 
| ucb_lb| 	2      | 		  | 
| dtr_lb| 	2       | 		  | 
| worker_lb	| 2	   |          |
| nfs	| 	       |    2      |
| logger	| 	   |  2	      | 
| **Total vCPU per node**|	**16** |**8**|



**Note:**  In the case of one ESX host failure, the remaining node can accommodate the amount of vCPU required.




**Table 2.** Memory allocation

| RAM | simply01 | simply02 |
|:------|:--------:|:--------:|
| ucp1  |	8      |          |
| dtr1  | 16	   |          |
| worker1 |16	   |          |
| worker2| 	       | 	16 | 
| ucb_lb| 	2      | 		  | 
| dtr_lb| 	2       | 		  | 
| worker_lb	|2 	   |          |
| nfs	| 	       |  2        |
| logger	| 	   |  2	      | 
| **Total RAM required** (per node)| 	**46**	| **20**| 
| **Total RAM required**| 	| 	**66**	
| Available RAM	| 384	| 384| 	


**Note:** In the case of one ESX host failure, the remaining host can accommodate the amount of RAM required for all VMs.



# Provisioning the operations environment

This section describes in detail how to provision the environment described previously in the architecture section. This section describes in detail how to provision the environment described previously in the architecture section. The high level steps this guide will take are shown in Figure 3.

![Provisioning steps][provisioning]
**Figure 3.** Provisioning steps


## Verify Prerequisites

You must assemble the information required to assign values to each and every variable used by the playbooks, before you start deployment. The variables are fully documented in the following sections “Editing the group variables” and “Editing the vault”. A summary of the information required is presented in Table 3.

**Table 3.** Summary of information required

|Component|	Details|
|---------|-----------|
|Virtual Infrastructure|	The FQDN of your vCenter server and the name of the Datacenter which contains the SimpliVity cluster. You will also need administrator credentials in order to create templates, and spin up virtual machines.
|SimpliVity Cluster	|The name of the SimpliVity cluster and the names of the member of this cluster as they appear in vCenter. You will also need to know the name of the SimpliVity datastore where you want to land the various virtual machines. You may have to create this datastore if you just installed your SimpliVity cluster. In addition, you will need the IP addresses of the OmniStack virtual machines. Finally you will need credentials with admin capabilities for using the OmniStack API. These credentials are typically the same as you vCenter admin credentials
|L3 Network requirements|	You will need one IP address for each and every VM configured in the Ansible inventory (see the section “Editing the inventory”). At the time of writing, the example inventory configures 14 virtual machines so you would need to allocate 14 IP addresses to use this example inventory. Note that the Ansible playbooks do not support DHCP so you need static IP addresses.   All the IPs should be in the same subnet. You will also have to specify the size of the subnet (for example /22 or /24) and the L3 gateway for this subnet.
|DNS|	You will need to know the IP addresses of your DNS server. In addition, all the VMs you configure in the inventory should have their names registered in DNS. In addition, you will need the domain name to use for configuring the virtual machines (such as example.com)
|NTP Services|	You need time services configured in your environment. The solution being deployed (including Docker) uses certificates and certificates are time sensitive. You will need the IP addresses of your time servers (NTP).
|RHEL Subscription	|A RHEL subscription is required to pull extra packages that are not on the DVD.|
|Docker Prerequisites|	You will need a URL for the official Docker EE software download and a license file.  Refer to the Docker documentation to learn more about this URL and the licensing requirements here: https://docs.docker.com/engine/installation/linux/docker-ee/rhel/ in the section entitled "Docker EE repository URL"|
|Proxy	|The playbooks pull the Docker packages from the Internet. If you environment accesses the Internet through a proxy, you will need the details of the proxy including the fully qualified domain name and the port number.|



## Enable vSphere High Availability
You must enable vSphere High Availability (HA) to support virtual machine failover during an HA event such as a host failure. Sufficient CPU and memory resources must be reserved across the system so that all VMs on the affected host(s) can fall over to remaining available hosts in the system. You configure an Admission Control Policy (ACP) to specify the percentage CPU and memory to reserve on all the hosts in the cluster to support HA functionality. 

More information on enabling vSphere HA and configuring Admission Control Policy is available in the HPE SimpliVity documentation. Log in to the HPE Support Center at https://www.hpe.com/us/en/support.html and search for “HPE SimpliVity 380”. The administration guide is listed when you select the User document type.

**Note:** You should not use the default Admission Control Policy. Instead, you should calculate the memory and CPU requirements that are specific to your environment.


## Install vSphere Docker Volume Service driver on all ESXi hosts

vSphere Docker Volume Service technology enables stateful containers to access the storage volumes provided by SimpliVity. This is a one-off manual step. In order to be able to use Docker volumes using the vSphere driver, you must first install the latest release of the vSphere Docker Volume Service (vDVS) driver, which is available as a vSphere Installation Bundle (VIB). To perform this operation, login to each of the ESXi hosts in turn and then download and install the latest release of vDVS driver.

```
# esxcli software vib install -v /tmp/vmware-esx-vmdkops-<version>.vib --no-sig-check
```

More information on how to download and install the driver can be found on the Docker Store at https://store.docker.com/plugins/vsphere-docker-volume-service







## Create a VM template

The first step of the automated solution is the creation of a VM Template that you will use as the base for all your nodes. In order to create a VM Template you will first create a Virtual Machine with the OS installed and then convert the Virtual Machine to a VM Template. Since the goal of automation is to remove as many repetitive tasks as possible, the VM Template is created as lean as possible, with any additional software installs and/or system configuration performed subsequently using Ansible.

It would be possible to automate the creation of the template. However, as this is a one-off task, it is appropriate to do it manually. The steps to create a VM template manually are described below.


1. Log in to vCenter and create a new Virtual Machine. In the dialog box, shown in Figure 4, select `Typical` and press `Next`.  
![Create New Virtual Machine][createnewvm]  
**Figure 4.** Create New Virtual Machine  
  
2. Specify the name and location for your template, as shown in Figure 5.  
![Specify name and location for the virtual machine][vmnamelocation]  
**Figure 5.** Specify name and location for the virtual machine  
  
3. Choose the host/cluster on which you want to run this virtual machine, as shown in Figure 6.
![Choose host / cluster][choosehost]  
**Figure 6.** Choose host / cluster 


4. Choose a datastore where the template files will be stored, as shown in Figure 7.
![Select storage for template files][selectstorage]  
**Figure 7.** Select storage for template files


5. Choose the OS as shown in Figure 8, in this case Linux, RHEL7 64bit.
![Choose operating system][chooseos]  
**Figure 8.** Choose operating system

6. Pick the network to attach to your template as shown in Figure 9. In this example there is only one NIC but depending on how you plan to architect your environment you might want to add more than one.
![Create network connections][createnetwork]  
**Figure 9.** Create network connections


7. Create a primary disk as shown in Figure 10. The chosen size in this case is 50GB but 20GB should be typically enough.
![Create primary disk][createprimarydisk]  
**Figure 10.** Create primary disk


8. Confirm that the settings are right and press Finish as shown in Figure 11.
![Confirm settings][confirmsettings]  
**Figure 11.** Confirm settings


9. The next step is to virtually insert the RHEL7 DVD, using the Settings of the newly created VM as shown in Figure 12. Select your ISO file in the Datastore ISO File Device Type and make sure that the “Connect at power on” checkbox is checked.  
![Virtual machine properties][vmprops]  
**Figure 12.** Virtual machine properties



10. Finally, you can optionally remove the Floppy Disk, as shown in Figure 13, as this is not required for the VM.
![Remove Floppy drive][removefloppy]  
**Figure 13.** Remove Floppy drive

11. Power on the server and open the console to install the OS. On the welcome screen, as shown in Figure 14, pick your language and press `Continue`.
![Welcome screen][welcomescreen]  
**Figure 14.** Welcome screen

12. The installation summary screen will appear, as shown in Figure 15.
![Installation summary][installsummary]  
**Figure 15.** Installation summary

13. Scroll down and click on Installation Destination, as shown in Figure 16.
![Installation destination][installdestination]  
**Figure 16.** Installation destination

14. Select your installation drive, as shown in Figure 17, and click Done.
![Select installation drive][installdrive]  
**Figure 17.** Select installation drive


15. Click Begin Installation, using all the other default settings, and wait for the configuration of user settings dialog, shown in Figure 18.
![Configure user settings][configuser]  
**Figure 18.** Configure user settings


16. Select a root password, as shown in Figure 19.
![Set root password][setrootpwd]  
**Figure 19.** Set root password



17. Click Done and wait for the install to finish. Reboot and log in into the system using the VM console.

18.	The Red Hat packages required during the deployment of the solution come from two repositories: `rhel-7-server-rpms` and `rhel 7-server-extras-rpms`. The first repository can be found on the Red Hat DVD but the second cannot. There are two options, with both options requiring a Red Hat Network account.


  - **Option 1:** Use Red Hat subscription manager to register your system. This is the easiest way and will automatically give you access to the official Red Hat repositories. Use the `subscription-manager register` command as follows.
```
# subscription-manager register --auto-attach
```
If you are behind a proxy, you must configure this before running the above command to register.
```
# subscription-manager config --server.proxy_hostname=<proxy IP> --server.proxy_port=<proxy port>
```
If you use this option, the playbooks will automatically enable the `extras` repository on the VMs that need it. If you encounter proxy errors related to `cdn.redhat.com`, you should run the subscription-manager refresh command to recreate your certificate.

  - **Option 2:** Use an internal repository. Instead of pulling the packages from Red Hat, you can create copies of the required repositories on a dedicated node. You can then configure the package manager to pull the packages from the dedicated node. Your `/etc/yum.repos.d/redhat.repo` could look as follows.
```
[RHEL7-Server]
name=Red Hat Enterprise Linux $releasever - $basearch
baseurl=http://yourserver.example.com/rhel-7-server-rpms/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[RHEL7-Server-extras]
name=Red Hat Enterprise Linux Extra pkg $releasever - $basearch
baseurl=http://yourserver.example.com/rhel-7-server-extras-rpms/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```
The following articles explain how you can create a local mirror of the Red Hat repositories and how to share them.
https://access.redhat.com/solutions/23016  
https://access.redhat.com/solutions/7227

Before converting the VM to a template, you will need to setup access for the Ansible host to configure the individual VMs. This is explained in the next section. 


## Create the Ansible node

In addition to the VM Template, you need another Virtual Machine where Ansible will be installed. This node will act as the driver to automate the provisioning of the environment and it is essential that it is properly installed. The steps are as follows:

1. Create a Virtual Machine and install your preferred OS (in this example, and for the sake of simplicity, RHEL7 will be used). The rest of the instructions assume that, if you use a different OS, you understand the possible differences in syntax for the provided commands. If you use RHEL 7, select **Infrastructure Server** as the base environment and the **Guests Agents** add-on during the installation.
2. Log in to the root account and create an SSH key pair. Do not protect the key with a passphrase (unless you want to use ssh-agent).  
```
# ssh-keygen
```
3. Configure the following yum repositories, rhel-7-server-rpms and rhel-7-server-extras-rpms as explained in the previous section.
4. Configure the EPEL repository. See here for more information: http://fedoraproject.org/wiki/EPEL. Note that `yum-config-manager` comes with the Infrastructure Server base environment, if you did not select this environment you will have to install the `yum-utils` package.
```
# rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# yum-config-manager --enable rhel-7-server-extras-rpms	
```
5. Install Ansible. The playbooks were tested with Ansible 2.2 and 2.3. Please note that the playbooks will not work with Ansible 2.4 due to an open defect https://github.com/ansible/ansible/issues/32000. Do not use Ansible 2.4 until this defect is fixed. To install the Ansible 2.3 use the following command:
```
# yum install ansible-2.3.2.0-2.el7
```
6. Make a list of all the hostnames and IPs that will be in your system and update your `/etc/hosts` file accordingly. This includes your UCP nodes, DTR nodes, worker nodes, NFS server, logger server and load balancers.
7. Install the following packages which are a mandatory requirement for the playbooks to function as expected. (Update pip if requested).
```
# yum install python-pyvmomi python-netaddr python2-jmespath python-pip gcc python-devel openssl-devel git
# pip install --upgrade pip
# pip install cryptography
# pip install pysphere
```
8. Copy your SSH public key to the VM Template so that, in the future, your Ansible node can SSH without the need of a password to all the Virtual Machines created from the VM Template.
```
# ssh-copy-id root@<VM_Template>
```

Please note that in both the Ansible node and the VM Template you might need to configure the network so one node can reach the other. Instructions for this step have been omitted since it is a basic step and could vary depending on the user’s environment.




## Finalize the template

Now that the VM Template has the public key of the Ansible node, we’re ready to convert this VM to a VM Template. Perform the following steps in the VM Template to finalize its creation:

1. Clean up the template by running the following commands:

```
# rm /etc/ssh/ssh_host_*
# history -c
```

2. Shut down the VM

```
# shutdown –h now
```

3. Once the Virtual Machine is ready and turned off, convert it to a template as shown in Figure 20.
![Convert to template][converttotemplate]  
**Figure 20** Convert to template  

This completes the creation of the VM Template.







## Prepare your Ansible configuration

On the Ansible node, retrieve the latest version of the playbooks using git.
```
# git clone https://github.com/HewlettPackard/Docker-SimpliVity
```
Change to the directory which you just cloned:
```# cd ~/Docker-SimplVity```

Change to the `dev` directory
```# cd dev```


**Note:** File names are relative to the `dev` directory. For example `vm_hosts` is located in `~/Docker-SimpliVity/dev` and `group_vars/vars` relates to `~/Docker-SimpliVity/dev/groups_vars/vars`.


You now need to prepare the configuration to match your own environment, prior to deploying Docker EE and the rest of the nodes. To do so, you will need to edit and modify three different files:



- `vm_hosts` (the inventory file)
- `group_vars/vars` (the group variables file)
- `group_vars/vault` (the encrypted group variable file)

You should work from the root account for the configuration steps and later when you run the playbooks.







## Editing the inventory

Change to the directory that you previously cloned using git and edit the vm\_hosts file.

The nodes inside the inventory are organized in groups. The groups are defined by brackets and the group names are static so they must not be changed. All other inforamtion, including hostnames, specifications, IP addresses can be edited to match your needs. The groups are as follows:

- [ucp\_main]: A group containing one single node which will be the main UCP node and swarm leader. Do not add more than one node under this group.
- [ucp]: A group containing all the UCP nodes, including the main UCP node. Typically you should have either 3 or 5 nodes under this group. For dev were only specifying 1 node.
- [dtr\_main]: A group containing one single node which will be the first DTR node to be installed. Do not add more than one node under this group.
- [dtr]: A group containing all the DTR nodes, including the main DTR node. Typically you should have either 3 or 5 nodes under this group. For dev were only specifying 1 node.
- [worker]: A group containing all the UCP nodes, including the main UCP node. Typically you should have either 3 or 5 nodes under this group.
- [ucp\_lb]: A group containing one single node which will be the load balancer for the UCP nodes. Do not add more than one node under this group. 
- [dtr\_lb]: A group containing one single node which will be the load balancer for the DTR nodes. Do not add more than one node under this group.
- [worker\_lb]: A group containing one single node which will be the load balancer for the worker nodes. Do not add more than one node under this group.
- [lbs]: A group containing all the load balances. This group will have 3 nodes, also defined in the three groups above.
- [nfs]: A group containing one single node which will be the NFS node. Do not add more than one node under this group.
- [logger]: A group containing one single node which will be the logger node. Do not add more than one node under this group.
- [local]: A group containing the local Ansible host. It contains an entry that should not be modified.

There are also a few special groups:

- [docker:children]: A group of groups including all the nodes where Docker will be installed.
- [vms:children]: A group of groups including all the Virtual Machines involved apart from the local host.

Finally, you will find some variables defined for each group:

- [ucp:vars]: A set of variables defined for all nodes in the [ucp] group.
- [dtr:vars]: A set of variables defined for all nodes in the [dtr] group.
- [worker:vars]: A set of variables defined for all nodes in the [worker] group.
- [lbs:vars]: A set of variables defined for all nodes in the [lbs] group.
- [nfs:vars]: A set of variables defined for all nodes in the [nfs] group.
- [logger:vars]: A set of variables defined for all nodes in the [logger] group.
- [elk:vars]:  A set of variables defined for all nodes in the [elk] group.
 
If you wish to configure your nodes with different specifications rather than the ones defined by the group, it is possible to declare the same variables at the node level, overriding the group value. For instance, you could have one of your workers with higher specifications by doing:

```
[worker]
worker01 ip_addr='10.0.0.10/16' esxi_host='esxi1.domain.local'
worker02 ip_addr='10.0.0.11/16' esxi_host='esxi1.domain.local'
worker03 ip_addr='10.0.0.12/16' esxi_host='esxi1.domain.local' cpus='16' ram'32768'
[worker:vars]
cpus='4'
ram='16384'
disk2_size='200'
node_policy='bronze'
```

In  the example above, the worker03 node would have 4 times more CPU and double RAM than the rest of worker nodes.

The different variables you can use are as described in the table below. They are all mandatory unless if specified otherwise:

| Variable | Scope | Description |
| --- | --- | --- |
| ip\_addr | Node | IP address in CIDR format to be given to a node |
| esxi\_host | Node | ESXi host where the node will be deployed. Please note that if the cluster is configured with DRS, this option will be overriden |
| cpus | Node/Group | Number of CPUs to assign to a VM or a group of VMs |
| RAM | Node/Group | Amount of RAM in MB to assign to a VM or a group of VMs |
| disk2\_usage | Node/Group | Size of the second disk in GB to attach to a VM or a group of VMs. This variable is only mandatory on Docker nodes (UCP, DTR, worker) and NFS node. It is not required for the logger node or the load balancers. |
| node\_policy | Node/Group | Simplivity backup policy to assign to a VM or a group of VMs. The name has to match one of the backup policies defined in the group\_vars/vars file described in the next section |

## Editing the group variables

Once our inventory is ready, the next step is to modify the group variables to match our environment. To do so, we need to edit the file group\_vars/vars under the cloned directory containing the playbooks. The variables here can be defined in any order but for the sake of clarity they have been divided into sections.

### VMware configuration

All VMware-related variables should be here. All of them are mandatory and described in the Table 2 below.

| Variable | Description |
| --- | --- |
| vcenter\_hostname | IP or hostname of the vCenter appliance |
| vcenter\_username | Username to log in to the vCenter appliance. It might include a domain i.e. '[administrator@vsphere.local](mailto:administrator@vsphere.local)' |
| datacenter | Name of the datacenter where the environment will be provisioned |
| vm\_username | Username to log into the VMs. It needs to match the one from the VM Template, so unless you have created an user, you must use 'root' |
| vm\_template | Name of the VM Template to be used. Note that this is the name from a vCenter perspective, not the hostname |
| folder\_name | vCenter folder to deploy the VMs. If you do not wish to deploy in a particular folder, the value should be '/' |
| datastores | List of datastores to be used, in list format, i.e. ['Datastore1','Datastore2'...]. Please note that from a Simplivity perspective it's best practice to use just one Datastore. Using more than one will not provide any advantages in terms of reliability and will add additional complexity. |
| disk2 | UNIX name of the second disk for the Docker VMs. Typically '/dev/sdb' |
| disk2\_part | UNIX name of the partition of the second disk for the Docker VMs. Typically '/dev/sdb1' |
| vsphere\_plugin\_version | Version of the vSphere plugin for Docker. The default is 'latest' but you could pick a specific version, i.e. '0.12' |

### Simplivity configuration

All Simplivity-related variables should be here. All of them are mandatory and described in the Table 3 below.

| Variable | Description |
| --- | --- |
| simplivity\_username | Username to log in to the Simplivity Omnistack appliances. It might include a domain i.e. ' [administrator@vsphere.local](mailto:administrator@vsphere.local)' |
| omnistack\_ovc | List of Omnistack hosts to be used, in list format, i.e. ['omni1.local','onmi2.local'...] |
| backup\_policies | List of dictionaries containing the different backup policies to be used along with the scheduling information. Any number of backup policies can be created and they need to match the node\_policy variables defined in the inventory. The format is as follows:backup\_policies: - name: daily'   days: 'All'   start\_time: '11:30'   frequency: '1440'   retention: '10080' - name: 'hourly'   days: 'All'   start\_time: '00:00'   frequency: '60'   retention: '2880' |
| dummy\_vm\_prefix | In order to be able to backup the Docker volumes, a number of "dummy" VMs need to be spin up. This variable will set a recognizable prefix for them |
| docker\_volumes\_policy | Backup policy to use for the Docker Volumes |

### Networking configuration

All network-related variables should be here. All of them are mandatory and described in the Table 4 below.

| Variable | Description |
| --- | --- |
| nic\_name | Name of the device, for RHEL this is typically ens192 and it is recommended to leave it as is |
| gateway | IP address of the gateway to be used |
| dns | List of DNS servers to be used, in list format, i.e. ['8.8.8.8','4.4.4.4'...] |
| domain\_name | Domain name for your Virtual Machines |
| ntp\_server | List of NTP servers to be used, in list format, i.e. ['1.2.3.4','0.us.pool.net.org'...] |

### Docker configuration

All Docker-related variables should be here. All of them are mandatory and described in the Table 5 below.

| Variable | Description |
| --- | --- |
| docker\_ee\_url | URL to your Docker EE packages |
| rhel\_version | Version of your RHEL OS, i.e: 7.3 |
| dtr\_version | Version of the Docker DTR you wish to install. You can use a numeric version or latest for the most recent one |
| ucp\_version | Version of the Docker UCP you wish to install. You can use a numeric version or latest for the most recent one |
| images\_folder | Directory in the NTP server that will be mounted in the DTR nodes and that will host your Docker images |
| license\_file | Full path to your Docker license file (it should be stored in your Ansible host) |
| ucp\_username | Username of the administrator user for UCP and DTR, typically admin. |

### Environment configuration

All Environment-related variables should be here. All of them are described in the Table 7 below.

| Variable | Description |
| --- | --- |
| env | Dictionary containing all environment variables. It contains three entries described below. Please leave empty the proxy related settings if not required: <ul><li>http\_proxy: HTTP proxy URL, i.e. 'http://15.184.4.2:8080'. This variable defines the HTTP proxy url if your environment is behind a proxy.</li><li>https\_proxy: HTTP proxy URL, i.e. 'http://15.184.4.2:8080'. This variable defines the HTTPS proxy url if your environment is behind a proxy.</li><li>no\_proxy: List of hostnames or IPs that don't require proxy, i.e. 'localhost,127.0.0.1,.cloudra.local,10.10.174.'</li></ul>|

## Editing the vault

Once our group variables file is ready, the next step is to create a vault file to match our environment. The vault file is essentially the same thing than the group variables but it will contain all sensitive variables and will be encrypted.

To create a vault we'll create a new file group\_vars/vault and we'll add the following entries:

```
---
vcenter_password: 'xxx'
vm_password: 'xxx'
simplivity_password: 'xxx'
ucp_password: 'xxx'
```

To encrypt the vault you need to run the following command:

```# ansible-vault encrypt group_vars/vault```

You will be prompted for a password that will decrypt the vault when required.

Edit your vault anytime by running:

```# ansible-vault edit group_vars/vault```

The password you set on creation will be requested.

In order for ansible to be able to read the vault we'll need to specify a file where the password is stored, for instance in a file called .vault\_pass. Once the file is created, take the following precautions to avoid illegitimate access to this file:

1. Change the permissions so only root can read it

```# chmod 600 .vault_pass```

1. Add the file to your .gitignore file if you're pushing the set of playbooks to a git repository

# Running the playbooks

So at this point the system is ready to be deployed. Go to the root folder and run the following command:

```# ansible-playbook -i vm_hosts site.yml --vault-password-file .vault_pass```

The playbooks should run for 25-35 minutes depending on your server specifications and in the size of your environment.

## Scaling out your environment

The playbooks are idempotent, which means that they can be run over and over but only the changes that are found from the previous run will be applied. In a second or subsequent run you might see some errors in the output but these are normal and can be safely ignored.

The reasoning behind providing idempotency is that you would be able to scale out your system if you wished to do so. If you had deployed an environment with 3 workers, 3 DTRs and 3 UCPs and you wanted to have 5 of each, you would just need to amend your inventory (vm\_hosts file) and run the above command again.

# Deep dive into the playbooks

This section will go more in detail about how the playbooks work and what functionalities are being provided.

## site.yml

The site.yml playbook is the main playbook and will contain a list of playbooks to be run sequentially. The list of playbooks is described in the following sections in running order:

## playbooks/create\_vms.yml

This playbook will create all the necessary Virtual Machines for the environment from the VM Template defined in the vm\_template variable.

It is composed of the following sequential tasks:

- Create all VMs: Creates all the required VMs from the specified templates in the specified folder. Each VM will be hosted by the specified esxi\_host defined in the inventory, unless if VMWare DRS technology is used, in which case the setting will be ignored and DRS will decide how to spread the load. The Virtual Machines will be powered on upon creation.
- Add secondary disks: Adds a second disk on the VMs having the disk2\_size variable defined (usually the Docker nodes and the NFS server, but not the load balancers or the logger node). The datastore will be chosen at random from the list provided in the group variables.
- Wait for VMs to boot up: Waits for 3 minutes for the VMs to be completely booted up.

The first two tasks make use of the vmware\_guest Ansible module. More information about the module can be found here: [http://docs.ansible.com/ansible/latest/vmware\_guest\_module.html](http://docs.ansible.com/ansible/latest/vmware_guest_module.html)

## playbooks/config\_networking.yml

This playbook will configure the network settings in all Virtual Machines.

It is composed of the following sequential tasks:

- Change hostname: Updates the /etc/hostname file with the hostname and domain name provided in the group variables.
- Update hostname: Makes the new hostname current by using the hostnamectl tool
- Add new connection: Adds a new network connection using the nmcli tool along with the network information provided in the group variables and inventory (IP address, gateway, interface name, etc.)
- Bring connection up: Brings up the newly created connection.
- Enable connection autoconnect: Makes sure the connection is brought up after a reboot.
- Update /etc/hosts: Updates the /etc/hosts file using a template. The template file is called j2 and is located in the templates folder. The template will loop over all the nodes defined in the inventory to have a complete and up-to-date hosts file. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)
- Update DNS settings: Updates the /etc/resolv.conf file using a template. The template file is called conf.j2 and is located in the templates folder. The template will loop over all the DNS hosts defined in the group variables to have a complete conf file. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)

Since at the beginning of the playbook the Virtual Machines don't have yet network connectivity, in most tasks we will make use of the vmware\_vm\_shell Ansible module, that allows us to run shell commands directly on the Virtual Machines. More information about this module can be found here: [http://docs.ansible.com/ansible/latest/vmware\_vm\_shell\_module.html](http://docs.ansible.com/ansible/latest/vmware_vm_shell_module.html)

## playbooks/distribute\_keys.yml

This playbook is optional and will distribute all nodes' public keys around so all nodes can password-less login to one another. Since this could be seen as a security risk (there is technically no need for a worker node user to log into an UCP node, for instance), it is disabled by default but can be uncommented for the site.yml file if required.

It is composed of the following sequential tasks:

- Register key: Stores the default SSH private key (/root/.ssh/id\_rsa)
- Create keypairs: Creates SSH key pairs in the nodes where these didn't yet exist, using the ssh-keygen tool.
- Fetch all public ssh keys: Registers all public keys from all nodes.
- Deploy keys on all servers: Pushes all keys to all nodes using a nested loop.

## playbooks/install\_haproxy.yml

This playbook will install and configure the haproxy package in the load balancer nodes. haproxy is the chosen tool to implement load balancing between UCP nodes, DTR nodes and worker nodes.

It is composed of the following sequential tasks:

- Open http and https ports: Makes use of the firewalld command to open the required ports tcp/80 and tcp/443
- Reload firewalld configuration: Reloads the firewall to apply the new rules
- Install haproxy: Installs the latest version of haproxy using the Ansible yum module.
- Update haproxy.cfg on Worker Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called worker.j2 and is located in the templates folder. The template will loop over all the Worker nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.
- Update haproxy.cfg on UCP Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called ucp.j2 and is located in the templates folder. The template will loop over all the UCP nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.
- Update haproxy.cfg on DTR Load Balancer: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load balancer. The template file is called dtr.j2 and is located in the templates folder. The template will loop over all the DTR nodes defined in the inventory to have a complete list of hosts in the backend. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the haproxy service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Enable and restart haproxy service: Makes sure that the service is enabled and restarted.

## playbooks/install\_ntp.yml

This playbook will install and configure the ntp package in all Virtual Machines in order to have a synchronized clock all across the environment. It will use the server or servers specified in the ntp\_servers variable in the group variables file.

It is composed of the following sequential tasks:

- Install ntp: Updates the /etc/haproxy/haproxy.cfg file using a template on the Worker load
- Update ntp.conf: Updates the /etc/ntp.conf file using a template. The template file is called conf.j2 and is located in the templates folder. The template will loop over all the provided NTP servers defined in the ntp\_servers variable. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the ntp service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Enable and restart ntp service: Makes sure that the service is enabled and restarted.

## playbooks/install\_docker.yml

This playbook will install Docker along with all its dependencies.

It is composed of the following sequential tasks:

- Install dependencies: Uses yum to install all the packages that are required by Docker.
- Set Docker url: Updates the /etc/yum/vars/dockerurl file with the contents of the docker\_ee\_url variable.
- Set Docker version: Updates the /etc/yum/vars/dockerosversion file with the contents of the rhel\_version variable.
- Add Docker repository: Runs yum-config-manager in order to add the yum repository specified in the docker\_ee\_url variable.
- Install Docker: Uses yum to install the latest version of Docker Enterprise Edition.

## playbooks/install\_rsyslog.yml

This playbook will install and configure rsyslog in the logger node and in all Docker nodes. The logger node will be configured to receive all syslogs on port 514 and the Docker nodes will be configured to send all logs (including container logs) to the logger node.

It is composed of the following sequential tasks:

- Open required ports for rsyslog: Makes use of the firewalld command to open the required ports tcp/514 and ucp/514 on the logger node
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Install rsyslog: Install the latest version of rsyslog on all docker and logger nodes
- Configure logger server: Updates the /etc/rsyslog.conf configuration file on the logger node with an updated file that allows to receive logs on port 514 using both UCP and TCP protocols. To achieve this, an amended conf file, stored under the files folder, is copied over to the logger server. This task will notify the handler defined below in order to restart the rsyslog service if a change in the file has occurred.
- Allow docker nodes to send logs: Updates the /etc/rsyslog.conf file using a template to allow sending logs to the logger node. The template file is called conf.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the rsyslog service if a change in the file has occurred.

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Restart Rsyslog: Makes sure that the rsyslog service is enabled and restarted.

## playbooks/config\_docker\_lvs.yml

This playbook will perform a set of operations on the Docker nodes in order to create a partition on the second disk and carry out the LVM configuration, required for a sound Docker installation.

It is composed of the following sequential tasks:

- Create partition on second disk: Uses the Ansible parted module to create a LVM partition on the second disk. The partition will have a GPT label and will use the 100% of the drive. The directive ignore\_errors present in this task (and in some of the following ones) exists in order to allow for the playbook to be run more than once, for instance if we're running yml again to scale out our environment. Since the partition will be already created in this case scenario, the task will fail but will be safely ignored. Unfortunately, Ansible does not provide, as of today, with a more elegant method to allow for this module to be idempotent when using GPT labels.
- Create Docker VG: Creates a Volume Group called docker in the newly created partition.
- Create thinpool LV: Creates a Logical Volume called thinpool in the docker Volume Group.
- Create thinpoolmeta LV: Creates a Logical Volume called thinpoolmeta in the docker Volume Group.
- Convert LVs to thinpool and storage for metadata: Uses the lvconvert command to convert the Logical Volumes to a thin pool and storage location for metadata for the thin pool
- Config thinpool profile: Creates the /etc/lvm/profile/docker-thinpool.profile configuration file. To achieve this, an preconfigured docker-thinpool.profile file, stored under the files folder, is copied over to the Docker nodes
- Apply the LVM profile: Uses the command lvchange to apply the changes of the profile file that was just copied
- Enable monitoring for LVs: Uses the command lvs to enable monitoring on the Logical Volumes
- Create /etc/docker directory: Creates an /etc/docker folder, that will be used to push the json file in the next task
- Config Docker daemon: Creates the /etc/docker/daemon.json file, containing the configuration required to use the devicemapper driver. The file also contains some configuration options for the rsyslog logging, allowing the container logs to be centralized. The template file is called json.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html)
- Enable and restart docker service: Makes sure that the docker service is enabled and restarted

## playbooks/docker\_post\_config.yml

This playbook will run a variety of tasks to complete the installation of the Docker environment.

It is composed of the following sequential tasks:

- Create Docker service directory: Creates a folder /etc/systemd/system/docker.service.d on all Docker nodes
- Add proxy details: Updates the /etc/systemd/system/docker.service.d/http-proxy.conf file using a template to configure the proxy settings when these have been defined in the env variable. The template file is called http-proxy.conf.j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html). This task will notify the handler defined below in order to restart the docker service if a change in the file has occurred.
- Add insecure registry: Modifies the file /usr/lib/systemd/system/docker.service on all Docker nodes, modifying the line containing the directive ExecStart to allow using an insecure DTR.

At this point we have a meta task which goal is to run the handlers if required so the changes above are taken in account. More information about meta tasks can be found here: [http://docs.ansible.com/ansible/latest/meta\_module.html](http://docs.ansible.com/ansible/latest/meta_module.html)

- Check if vsphere plugin is installed: Queries the list of Docker plugins to find out if the vSphere plugin has already been installed. The task will record the output, which will be used as a conditional for the next step
- Install vsphere plugin: Installs the vSphere plugin if it's not already installed

This playbook also contains a handler, which is a task that will be run when notified by one or more of the specified tasks above. The goal of having a handler is typically to restart a service only when a configuration file has been changed.

- Restart Docker: Makes sure that the docker service is restarted.

## playbooks/install\_nfs\_server.yml

This playbook will install and configure an NFS server on the NFS node.

It is composed of the following sequential tasks:

- Open required ports: Makes use of the firewalld command to open the required ports for the NFS server
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Create partition on second disk: Uses the Ansible parted module to create a partition on the second disk. Please note that since the GPT label is not used here, the module is by default idempotent and the tasks can be run multiple times, regardless of whether the partition exists or not, without failing. Hence we don't need the directive ignore\_errors in this occasion.
- Create filesystem: Creates an xfs filesystem in the partition created above.
- Create images folder: Creates a directory in the filesystem above. The directory will be named after the variable images\_folder and will host the images stored in the DTR nodes
- Mount filesystem: Uses the Ansible mount module to mount the directory created above
- Install NFS server: Uses the Ansible yum module to install the latest versions of the packages nfs-utils and rpcbind.
- Enable and start NFS services on server: Makes sure that the following services are enabled and started: rpcbind, nfs-server, nfs-lock, nfs-idmap.
- Modify exports file on NFS server: Updates the /etc/exports file using a template with all the mount points to be exported by the NFS service. The template file is called j2 and is located in the templates folder. More information about Ansible templates can be found here: [http://docs.ansible.com/ansible/latest/playbooks\_templating.html](http://docs.ansible.com/ansible/latest/playbooks_templating.html).
- Refresh exportfs: Runs the exportfs command to make the shared folder available

## playbooks/install\_nfs\_clients.yml

This playbook will install the required packages on the DTR nodes to be able to mount an NFS share.

It is composed of one single task:

- Install NFS client: Uses the Ansible yum module to install the latest version of the package nfs-utils.

## playbooks/install\_ucp\_nodes.yml

This playbook will install and configure the Docker UCP nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for UCP: Makes use of the firewalld command to open the required ports for UCP
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install UCP on nodes that are already part of the swarm
- Copy the license: Uses the Ansible copy module to push the license file into the first UCP node, prior to starting the installation
- Install swarm leader and first UCP node: Installs UCP in the first node. The command will specify the UCP version to install, the host IP address, the user and password for the administrator, the license and all the san information. More details on installing UCP can be found here: [https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/](https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/)
- Get swarm manager token: Generates and registers the command to add a manager node to the swarm
- Get swarm worker token: Generates and registers the command to add a worker node to the swarm
- Save worker token: Creates (or rewrites) a file /tmp/worker\_token and copies the worker token generated earlier into the file. This file will be used later to add the worker and DTR nodes to the swarm
- Add additional UCP nodes to the swarm: Makes use of registered manager token two tasks above to join the rest of the UCP nodes to the swarm

## playbooks/install\_dtr\_nodes.yml

This playbook will install and configure the Docker DTR nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for DTR: Makes use of the firewalld command to open the required ports for the DTR
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Get worker token: Retrieves the file created in the previous playbook (/tmp/worker\_token) and registers the contents in a variable
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install DTR on nodes that are already part of the swarm
- Add DTR nodes to the swarm: Uses the previously registered command to join the swarm
- Install first DTR node: Installs DTR in the first node. The command will specify the DTR version to install, the NFS images folder location, the fully qualified hostname of the DTR node, the address of the DTR load balancer, the URL of the main UCP node and the username and password to access UCP. More details on installing the DTR can be found here: [https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/](https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/)
- Get replica ID: Uses the docker ps command to extract and register the replica ID. This is used to add additional DTR nodes to the first one
- Add DTR nodes: Adds additional nodes to the DTR cluster. The command will specify the DTR version to install, the fully qualified hostname of the DTR node, the URL of the main UCP node, the username and password to access UCP and the replica ID captured in the previous step.
- Enable image scanning: *Note: this step is now disabled since from DTR 2.2.0 the API has changed and the image scanning is enabled by default*. Makes use of the DTR REST API to enable the Image Scanning feature. This feature is only available when using a Docker EE Advanced license. More information about the Image Scanning feature can be found here: [https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/scan-images-for-vulnerabilities/#change-the-scanning-mode](https://docs.docker.com/datacenter/dtr/2.2/guides/user/manage-images/scan-images-for-vulnerabilities/#change-the-scanning-mode). A reference for the REST API can be found here: [https://docs.docker.com/datacenter/dtr/2.2/reference/api/](https://docs.docker.com/datacenter/dtr/2.2/reference/api/). Please note that the REST API is still under development and some of the described methods might not reflect the reality of the current available API.

## playbooks/install\_worker\_nodes.yml

This playbook will install and configure the Docker Worker nodes defined in the inventory.

It is composed of the following sequential tasks:

- Open required ports for Workers: Makes use of the firewalld command to open the required ports for the Worker nodes
- Reload firewalld configuration: Reloads the firewall on the logger node to apply the new rules
- Get worker token: Retrieves the file created in the previous playbook (/tmp/worker\_token) and registers the contents in a variable
- Check if node already belongs to the swarm: Uses the docker info command to verify if the node is already part of the swarm and records the output to be used in later tasks. This is to prevent trying to install Workers on nodes that are already part of the swarm
- Add Worker nodes to the swarm: Uses the previously registered command to join the swarm

## playbooks/install\_elk.yml

This playbook will install and configure ELK stack defined in the inventory.

It is composed of the following sequential tasks:

- Open port: Makes use of the firewalld command to open the required ports tcp/5000
- Reload firewalld configuration: Reloads the firewall to apply the new rules
- Copies configuration files from src: files/elk/* to elk virtual machine under root
- Installing elk stack: install docker\-compose from github.com/docker/compose
- Starts docker.service
- Docker build from file elk-Dockerfile usning docker.elastic.co/logstash/logstash:5.4.2_hpe
- Execute compose file: elk-docker-compose.yml
- Run logspout as a service which will send all container logs to logstash elk host which are visible through http://(your-elk-host}:5000

## playbooks/install\_cloudbees.yml

This playbook will install and configure Cloudbees Team Edition CI/CD Jekins pipe line.

- Cloudbees team edition software is pulled from docker hub
- Container is spun up from cloudbees\-jenkins\-team image
- exposing port 8080
- Jenkins application starts and provides you with instance ID for licensing
- Admin password will be displayed on start\-up or alternativley \/var\/jenkins\_home\/secrets/initailAdminPasssword


## playbooks/install\_playwithdocker.yml

This playbook will install and configure Play With Docker container defined in the inventory.

- Play with docker or PWD will provide a developer with a crash and burn environment to run up to 5 instances in docker swarm mode for a duration of 24hrs which will assist with knowledge take on and temp tested bed.

#ADD tasks here#

## playbooks/config\_dummy\_vms\_for\_docker\_volumes\_backup.yml

This playbook will make sure that we are able to backup Docker volumes that have been created using the vSphere plugin in Simplivity. There is not a straight forward way to do this, so we need to use a workaround. Since all Docker volumes are going to be stored in the dockvols folder in the datastore(s), we need to create a 'dummy' VM per datastore. The vmx, vmsd and vmkd files from this VMs will have to be inside the dockvols folder, so when these VMs are backed up, the volumes are backed up as well. Obviously these VMs don't need to take any resources and we can keep them powered off. Remember to create the dockvols folder under your specific datastores.

It is composed of the following sequential tasks:

- Create Dummy VMs: Creates a 'dummy' VM per datastore using extremely low specifications. The name of the VMs will be prefixed with the variable dummy\_vm\_prefix and suffixed with the datastore name where it lives.
- Generate powercli script: Generates a powerCLI script using a template that will take in account all defined datastores and that will be copied to the main UCP node in the location defined by the powercli\_script variable. The template file is called powercli\_script.j2 and is located in the templates folder. The script will perform the following tasks:
  - For each 'dummy' VM, the files \*.vmx, \*.vmsd and \*.vmdk will be copied over to the dockvols folder in its current datastore
  - Using the \*.vmx files, we'll registed new 'dummy' VMs that now live inside the dockvols folder. The name of these VMs will be composed of the dummy\_vm\_prefix, the string "-in-dockvols-" and the datastore name as the suffix.
  - The old dummy VMs will be removed
- Run powercli script on temporary docker container: Runs the generated script on a powerCLI container to perform the tasks described above. The container provides a powerCLI core environment and is provided by VMWare. More information about powerCLI core can be found here: [https://labs.vmware.com/flings/powercli-core](https://labs.vmware.com/flings/powercli-core). A good article on how to use the container can be found here: [http://www.virtuallyghetto.com/2016/10/5-different-ways-to-run-powercli-script-using-powercli-core-docker-container.html](http://www.virtuallyghetto.com/2016/10/5-different-ways-to-run-powercli-script-using-powercli-core-docker-container.html)
- Delete powercli script from docker host: Deletes the generated script from the UCP host since it contains sensitive information, like the vCenter credentials, in plain text.

## playbooks/config\_simplivity\_backups.yml

This playbook will configure the defined backup policies in the group variables file in Simplivity and will include all Docker nodes plus the 'dummy' VMs created before, so the existing Docker volumes are also taken in account. The playbook will mainly use the Simplivite REST API to perform these tasks. A reference to the REST API can be found here: [https://api.simplivity.com/rest-api\_getting-started\_overview/rest-api\_getting-started\_overview\_rest-api-overview.html](https://api.simplivity.com/rest-api_getting-started_overview/rest-api_getting-started_overview_rest-api-overview.html)

It is composed of the following sequential tasks:

- Get Simplivity token: Makes a REST call against the Simplivity API to authenticate and retrieve a token that will be used in the following tasks. More information about authenticating against the Simplivity API can be found here: [https://api.simplivity.com/rest-api\_getting-started\_getting-started/rest-api\_getting-started\_getting-started\_request-oauth-2-token.html](https://api.simplivity.com/rest-api_getting-started_getting-started/rest-api_getting-started_getting-started_request-oauth-2-token.html)
- Retrieve current backup policies: Makes a REST call against the Simplivity API to retrieve all the current backup policies. The result will be stored in a variable current\_policies.
- Extract existing policies names: Uses the Ansible directive set\_fact to create a variable current\_policies\_names that will store a list of the names of all the currently available backup policies, based on the result of a JSON query against the variable current\_policies, registered above. This task provides a subset of the output from the previous task, which will return not just the policies names, but all the related information to each policy.
- Extract policies names to be added: Uses the Ansible directive set\_fact to create a variable backup\_policies\_names that will store a list of the names of the backup policies defined by the user, based on the result of a JSON query against the variable backup\_policies, defined in the group variables.
- Set list of nonexistent policies to be added: Uses the Ansible directive set\_fact to create a variable new\_policies\_names that will store a list of the names of the backup policies to be created, based on the result of the difference between the existing policies and the newly defined ones.
- Create backup policies: Makes a REST call against the Simplivity API to create the backup policies listed in the new\_policies\_names variable.
- Get policy IDs: Makes a REST call against the Simplivity API to retrieve all the current backup policies. The result will be stored in a variable policy\_ids.
- Create backup rules: Makes a REST call against the Simplivity API to create the defined backup rules in the global variable backup\_policies. In order to avoid duplicated rules, the task will only run when the new\_policies\_names is not empty, meaning that the backup policies defined by the user had not been created yet before running this playbook.
- Get VMs information: Makes a REST call against the Simplivity API to retrieve all the VMs information.
- Assign backup policies to VMs: Makes a REST call against the Simplivity API to assign the previously created VMs to a backup policy in Simplivity. Each node or node group in the inventory has a node\_policy variable that will define which policy they should be on.
- Set dummy VM names in one string: Uses the Ansible directive set\_fact to create a variable dummy\_vms\_string that will store a string with the the names of the dummy VMs, separated by commas
- Convert to list: Uses the Ansible directive set\_fact to create a list dummy\_vms that will be a list based on the string dummy\_vms\_string from the previous task
- Assign backup policies to Docker volumes: Makes a REST call against the Simplivity API to assign the previously created dummy VMs to a backup policy in Simplivity. This will make sure that all the Docker volumes will be also backed up on a regular basis

# Accessing the UCP UI
Once the playbooks have run and completed successfully, the Docker UCP UI should be available by browsing to the UCP load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your credentials and the dashboard will be displayed. You should see all the nodes information in your Docker environment by clicking on Nodes. By looking into the services you should see the monitoring services that were installed during the playbooks execution:

# Accessing the DTR UI
The Docker DTR UI should be available by browsing to the DTR load balancer or any of the nodes via HTTPS. The authentication screen will appear. Enter your UCP credentials and you should see the empty list of repositories. If you navigate to `Settings > Security`, you should see the Image Scanning feature already enabled (note that you need an Advanced license to have access to this feature).

# Security considerations
In addition to having all logs centralized in an unique place and the image scanning feature enabled, there are another few guidelines that should be followed in order to keep your Docker environment as secure as possible.

## Securing the daemon socket
The default is to create a non-networked socket to communicate with the daemon, but you could alternatively communicate using TLS. This will require you to create a CA and server and client keys with OpenSSL. The whole procedure is explained in detail here: [https://docs.docker.com/engine/security/https/](https://docs.docker.com/engine/security/https/)

## Start with known images
Developers can pick base images from the Docker Hub to start with. Although this allows them a great deal of freedom, some caution is needed. Images from Docker Hub are uncurated and may contain vulnerabilities. Use store images where possible. These are curated images. Use small base images to reduce the surface area.

## Enable Docker Content Trust
Notary/Docker Content Trust is a tool for publishing and managing trusted collections of content. Publishers can digitally sign collections and consumers can verify integrity and origin of content. This ability is built on a straightforward key management and signing interface to create signed collections and configure trusted publishers. More information about Notary can be found here: [https://success.docker.com/Architecture/Docker\_Reference\_Architecture%3A\_Securing\_Docker\_EE\_and\_Security\_Best\_Practices#dtr-notary](https://success.docker.com/Architecture/Docker_Reference_Architecture%3A_Securing_Docker_EE_and_Security_Best_Practices#dtr-notary)

## Prevent tags from being overwritten
By default, users with access to push to a repository, can push the same tag multiple times to the same repository. As an example, a user pushes an image to library/wordpress:latest, and later another user can push the image with exactly the same name but different functionality. This might make it difficult to trace back the image to the build that generated it.

To prevent this from happening you can configure a repository to be immutable. Once you push a tag, DTR won't anyone else to push another tag with the same name.

More information about immutable tags can be found here: [https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/](https://beta.docs.docker.com/datacenter/dtr/2.3/guides/user/manage-images/prevent-tags-from-being-overwritten/)

## Use secrets
Use secrets to pass credentials to a container. Never pass as an environment variable which is clear

## Isolate swarm nodes to a specific team
With Docker EE Advanced, you can enable physical isolation of resources by organizing nodes into collections and granting Scheduler access for different users. To control access to nodes, move them to dedicated collections where you can grant access to specific users, teams, and organizations.

More information about this subject can be found here: [https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/](https://beta.docs.docker.com/datacenter/ucp/2.2/guides/admin/manage-users/isolate-nodes-between-teams/)

## Docker Bench for Security
The Docker Bench for Security is a script that checks for dozens of common best-practices around deploying Docker containers in production. The tests are all automated, and are inspired by the CIS Docker Community Edition Benchmark v1.1.0.

The Docker Bench for Security should be run on a regular basis to make sure that our system is as secure as we'd expect it to be.

More information about this tool plus the files to run it can be found in its Github repository: [https://github.com/docker/docker-bench-security](https://github.com/docker/docker-bench-security)

# Contact
Please get in touch via Github if you have any questions.

# Demo
A much briefer video with a quick demo can be found here: https://vimeo.com/229389079



[dev-architecture]: </dev/images/architecture.png> "Figure 1. Solution Architecture"
[solutionstack]: </dev/images/solutionstack.png> "Figure 2. Solution stack"
[provisioning]: </dev/images/provisioning.png> "Figure 3. Provisioning steps"

[createnewvm]: </dev/images/createnewvirtualmachine.png> "Figure 4. Create New Virtual Machine"
[vmnamelocation]: </dev/images/vmnamelocation.png> "Figure 5. Specify name and location for the virtual machine" 
[choosehost]: </dev/images/choosehost.png> "Figure 6. Choose host / cluster"
[selectstorage]: </dev/images/selectstorage.png> "Figure 7. Select storage for template files"
[chooseos]: </dev/images/chooseos.png> "Figure 8. Choose operating system"
[createnetwork]: </dev/images/createnetwork.png> "Figure 9. Create network connections"
[createprimarydisk]: </dev/images/createprimarydisk.png> "Figure 10. Create primary disk"
[confirmsettings]: </dev/images/confirmsettings.png> "Figure 11. Confirm settings"
[vmprops]: </dev/images/vmprops.png> "Figure 12. Virtual machine properties"
[removefloppy]: </dev/images/removefloppy.png> "Figure 13. Remove Floppy drive"
[welcomescreen]: </dev/images/welcomescreen.png> "Figure 14. Welcome screen"
[installsummary]: </dev/images/installsummary.png> "Figure 15. Installation summary"
[installdestination]: </dev/images/installdestination.png> "Figure 16. Installation destination"
[installdrive]: </dev/images/installdrive.png> "Figure 17. Select installation drive"
[configuser]: </dev/images/configuser.png> "Figure 18. Configure user settings"
[setrootpwd]: </dev/images/setrootpwd.png> "Figure 19. Set root password"

[converttotemplate]: </dev/images/converttotemplate.png> "Figure 20. Convert to template"
[ucpauth]: </dev/images/ucpauth.png> "Figure 21. UCP authentication screen"
[ucpdash]: </dev/images/ucpdash.png> "Figure 22. UCP dashboard"
[nodesinfo]: </dev/images/nodesinfo.png> "Figure 23. Nodes information"
[servicesinfo]: </dev/images/servicesinfo.png> "Figure 24. Services information"
[dtrauth]: </dev/images/dtrauth.png> "Figure 25. DTR authentication screen"
[dtrrepos]: </dev/images/dtrrepos.png> "Figure 26. DTR repositories"
[imagescanning]: </dev/images/imagescanning.png> "Figure 27. Image scanning in DTR"
[DTRstorage]: </dev/images/DTRstorage.png> "Figure 28. DTR storage settings"
