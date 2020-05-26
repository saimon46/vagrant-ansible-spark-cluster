# Vagrant Ansible Spark Standalone Cluster
Vagrant - Ansible setup of a Spark standalone cluster with Firewall configured and Livy proxy server.

## Setup on my machine
Before starting to work at the project I downloaded all the most important tools to work and to deploy the virtual enviroment. Many of them are tools or softwares that I already use to develop Java or Scala code or to work daily.
- **Visual studio code**: Used mainly work on the Project. All the code, all the yml files for example, is already validated and prettified in the editor. I use this tool because it has a good interaction with the Git repository and it's very good to work with different types of files, thanks to different add-ons or plugin that I can install. In this case I used the *Ansible* and the *Vagrant* plugins.
- **Intellij IDEA**: Used to code the `Hello World` Spark job deployed to test the cluster.

I also installed Vagrant and Ansible in my machine with the latest versions:
```console
$ vagrant --version
Vagrant 2.2.9
```
```console
$ ansible --version
ansible 2.9.9
```

## Spark
To start I looked into the Spark documentation and the Apache Spark website to see how to deploy a cluster with 2 main components: **Master** and **Slaves**.\
I then started to look also to all the possible ways to submit a Spark job to the cluster and I wrote on paper the different alternatives that I will explain in the Livy paragraph. 

## Livy
I also had a look to the Livy server that I finally decided to use as the way to submit the Spark jobs from outside. For security reasons is better have a single proxy server machine that is specifically in charge to talk with the Spark cluster internally and isolate the cluster from outside and avoid the direct connection from the user to the cluster.\
Other than this on Livy it's possible to do user authentication and on the proxy side and user impersonation to avoid possible privilege esclation. Many times is better add a new component that is in charge of handle the cluster service than putting at risk the entire cluster.\
Unfortunately I didn't manage to have the service up and running and working with the cluster but I did most of the work to have it in place.\
On the role livy paragraph I explain the reasons.

## Job submit
To test that the Spark is correctly working I created a small `HelloWorld` job that I saved as Jar in the the project.\
To submit it I used the Livy machine as driver that is used to submit the job. Here the command to run the playbook:
```console
$ ansible-playbook -i inventory.yml spark-submit-application.yml
```
The playbook uses the `spark-submit` command with the class definition and the jar file passed from the host machine.

## Vagrant
Using a Vagrant environment I started to setup the `Vagrantfile` in order to create different machines for every single component of the cluster, with different memory and CPU setup.\
I used a **Centos 7 base** Vagrant box to work with.\
The Vagrant setup creates instances under the same IP subnet `192.168.33.0/24`. They are splitted in this way:
| Type | CPU | Memory | # |
|---|---|---|---|
| Master | 1 | 1536 | 1 |
| Slave | 1 | 1024 | 2 |
| Livy | 1 | 768 | 1 |

Of course, is possible to change the number of Slaves and CPU and Memory of every machine.\
The variable `$num_instances` takes in consideration only che cluster elements and not the Livy proxy instance.\
In the Vagrant setup I also called the Ansible provisioner in order to setup all the machines created with the playbook `spark.yml` and the inventory `inventory.yml` that I created accordingly.\
To startup the process I use the command:
```console
$ vagrant up
```

## Ansible
First of all, Ansible is a tool that I never used but, as I told you, it's not a problem to learn new things that is the key of every job.\
I started to look the website and the documentation to familiarize with it, thing that I did many times to resolve many problems that I found during the definition of the Spark Cluster setup.\
Since I setup the provisioner in the Vagrant file, when Vagrant creates the machines it start automatically the provisioning made with Ansible.\
Of course to rerun the provisioning I used the command:
```console
$ vagrant provision
```
that force Vagrant to run only the provisioning, but can be used the ansible command as well:
```console
$ ansible-playbook -i inventory.yml spark.yml
```

## Setup of the inventory
In the inventory I specified all the machines necessary to be configured by Ansible. This file can be easily modified to be compatible with a real cluster not virtual.

## Setup of the config file
In the project, in the `var` folder there is the `config.yml` used for confiuguration purposes. There I put all the configurations for all the components of the project.\
This configuration file is then used in many tasks importing it in this way:
```console
  vars_files:
    - ./vars/config.yml
```

## Setup of the playbooks
The Ansible configuration and setup is made structuring all the different roles of the different components of the environment.\
Every role can be used on a different set of machines and to do so I created different roles for every single configuration category necessary.
- The important thing to mention is that all the package installations are driven by the presence of the service already installed in the machine. This means that all the time that the playbook runs, only if the application is not installed the installation is done.\
Of course this installation method doesn't work well with different versions of the frameworks or in case it's necessary to update the software. In this case would be necessary to check if that version of the package is installed to proceed with it. But it's even better create a package for the software and use a package manager, `yum` in our case, to do the installation of the version desired.\
For this specific project I preferred to go in this direction.
- Another important thing to mention is also the service management for the different processes running in the machines. You will see later, that the startup, and the relative restart, of the Spark processes (master or slave) and the Livy server is done with an handler that is triggered all the time a change is made on the configuration. This is a good approch yes, but it's better to create a service managed for example by systemctl, that takes control of the process and it can give you its status all the time. In this case that process running is not forgotten and left alone but, it can be managed by systemctl and Ansible consequently.

### Role: `users`
This is the role that define the users in the machines. It has a single task to create the service account `spark` **not sudoer** used by all the services running around the Spark cluster.\
Having a separate user for the spark service is better for different reasons. In case there is bug in the Spark service, a possible intruder cannot do a lot since the `spark` user has only the permissions to access to the Spark environment. If the Spark application run instead as `root` the possible intruder would have take control of the entire machine and all the cluster soon.

### Role: `spark`
This is the role that configure the entire Spark cluster. It has different tasks that I splitted on purpose in different files to have them better organized.\
There is one part for the init of the service, one for configuration and another to install.\
The installation and work directories related to the spark service have the `spark` user ownership.\
There are different templates where I specified different configurations such as the list of the slaves, the environment variables used by the service and of course the `log4j.properties` file where I specified the log rotation and deletion. The logs are saved in the folder `/var/log/spark`.\
In the `config.yml` file I spedified different important ports that I later on open internally or externally with the firewall rule definition. 
| Type | Port |
|---|---|
| Master port | 7077 |
| Worker port | 1024-65535 |
| Master UI port | 8080 |
| Worker UI port | 8085 |

### Role: `livy`
As for the spark application I did the same to organize all the different task to be done on the Livy machine. In this case the logs are saved in `/var/log/livy`.\
Unfortunately, for some reason I have some problem starting-up the Livy server from the playbook in Ansible. I have some errors maybe because some environment variables not correctly setted. It's not a problem of users because Ansible uses the correct `spark` user and all the folders used by Livy are correctly owned by the user `spark`. Very strange because I didn't manage to reproduce the same error manually in the machine since, the server starts and works fine.\
To do so I used the command entering in the machine with `vagrant`:
```console
$ vagrant ssh livy
vagrant@livy$ sudo runuser -l spark -c "/opt/livy/bin/livy-server start"
```
Here it's necessary to use the `spark` user again.\
To finish the work on this component I have to finish to setup the connection with the Spark cluster and fix the problem at the startup. After that it's fine to test it with the proper job.

### Role: `python`
I used this role to install python3 since it's not present by default in a Centos 7 machine. I used Python to test the manual job submit through Python API.\
I disabled it because because I decided to use the Livy server to submit Spark Jobs.

### Role: `passwordless`
This role is used to allow passwordless ssh connection between all the Cluster nodes (master and slaves). The node outside, for example the Livy server, doesn't have any passwordless configuration with the cluster.
This module runs only if the user change. It's skipped if the spark user remains the same.

### Role: `java`
This role is used to install java. It's used by all the machines.

## Firewall
I configured different roles for the different firewall configurations of the different component of the environment.\
What is really important is that the cluster has to be totally isolated from outside so no traffic is allowed going in except the UI dashboard of the Spark cluster.\
In order to do so I defined a zone `spark` with all the cluster components, Livy is included.\
The intra-cluster traffic is tollerated following the list of ports that I defined.\
The extra-cluster traffic is tollerated only for the UI http traffic in all the nodes as the table below.\
I didn't enable the `ssh` traffic on port `22` on the `default` zone because by default the port on a Centos 7 machine with sshd installed is already open.\
In case of the Livy server I had to open all the ports from `1024` to `65535` to the `spark` zone in order to allow the cluster to talk to the driver that submits the job. All those ports because I didn't specify a fixed port. I did so because this allows to submit multiple jobs in parallel without using a single driver port.

| Type | Port | Zone |
|---|---|---|
| Master port | 7077/tcp | spark |
| Worker port | 1024-65535/tcp | spark |
| Master UI port | 8080/tcp | default |
| Worker UI port | 8085/tcp | default |
| Livy driver port | 1024-65535/tcp | spark |
| Livy UI port | 8998/tcp | default |

To test the configuration I used the `nc` command to see if the ports were open or close from inside or outside the cluster (to test from outside I run it from my machine).\
To verify the connection ESTABLISHED I used `netstat -putan | grep ESTABLISHED` or `ss | grep ESTAB`.
```console
tcp    ESTAB      0      0      [::ffff:192.168.33.10]:7077                 [::ffff:192.168.33.30]:53144
tcp    ESTAB      0      0      [::ffff:192.168.33.10]:7077                 [::ffff:192.168.33.20]:42548
```
These are the ESTABLISHED connection from the master to the slaves.

### Role: `firewalld`
Role defined to enable the firewalld service and create the `spark` zone.

### Role: `firewalld_master`
Role defined for the specific Spark master traffic.

### Role: `firewalld_slaves`
Role defined for the specific Spark slaves traffic.

### Role: `firewalld_livy`
Role defined for the specific Livy traffic.
