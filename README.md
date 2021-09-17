# Rolling updates for AWS AutoScaling Groups using Ansible Playbook and jenkins!

 ASG's let you declare a configuration, a minimum and maximum number of ec2 nodes, a desired number, and it keeps that number going for you. It even lets you set it up to scale up or down automatically based on cloudwatch events, effectively adjusting the desired number of nodes in the ASG, in response to load.

##### The challenge is: how do you update it?

If you change the launch configuration or launch template, it does not cause new nodes to be rolled in. Even if it did, you would want them to roll in sanely and slowly, one at a time, rather than all at once.

Here, [Anisble][1] does a decent job, with a little extra work,

1. Clone the latest files to a Git-Cloned directory.
1. Offloads the instances one by one.
2. Copies the updated files from Git-Cloned directory to document_root.
3. Activate the Offloaded instance after completing copy from Git-Cloned directory.

[1]: https://www.ansible.com/

In here will be looking into my small project to configure continues integration for a website on an existing Autoscaling group under an Elastic Load Balancer using ansible playbook  by cloning Git repository and manages the rolling update through Jenkins. I have configured the Webhook feature to alert the ansible for initiating the update.  

## Features

- Rolling update using Jenkins for Amazon Auto Scaling Group through playbook.
- Dynamic hosts inventory using ansible module "add_host"
- Seperate file for LoadBalancer health check.
- Customizable variables for IAM accesss&secrete keys, region,domain-name, document-root etc.


## Prerequisites
- Ansible.
- IAM user with required previleges.
      _NB: IAM role should be attached to the Ansible Master_
- Basic knowledge in Autoscaling group and Ansible-playbook.

To install ansible:
> https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

## Architecture
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/rolling.jpg "image Title")
## Required Modules in Ansible.
- yum
- pip
- ec2_instance_info
- add_host
- git
- copy
- service
- file
- pause

## 🗄️ Ansible Playbook:

Below is the main playbook "main.yml" I have written to complete this task. There is also two more files one consists of required variables "rolling.vars" and another one have the userdata which I have used to configure the launch configuration. 

```sh
---
- name: "ASG rolling update"
  hosts: localhost
  vars_files:
    - rolling.vars
  tasks:
  
    - name: "PIP Installation"
      yum:
        name: pip
        state: present
    
    - name: "boto3 Installation"
      pip:
        name: boto3
        state: present
    
    - name: "Gathering EC2 ASG Informations"
      ec2_instance_info:
        region: "{{ region }}"
        filters: 
          "tag:aws:autoscaling:groupName": "{{ tag_name }}"
          instance-state-name: ["running"]
      register: asg_instances
    
    - name: "Dynamic Inventory For ASG Instances"
      add_host: 
        name: "{{ item.public_ip_address }}"
        groups: "ansible-asg"
        ansible_host: "{{ item.public_ip_address }}" 
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ private_key }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ asg_instances.instances }}"
    
- name: "Website Deploying from Git-repo to ASG Instances"
  become: true
  hosts: ansible-asg
  gather_facts: false
  serial: 1
  vars_files:
    - rolling.vars
  tasks:
    - name: "httpd$git Installation"
      yum:
        name: 
          - httpd
          - git
        state: latest
    
    - name: "Creating the Clone Directory"
      file:
        path: "{{ clone_path }}"
        state: directory
    
    - name: "Cloning the Git Repository"
      git:
        repo: https://github.com/AkhiljithPB/techcafe-web-dev.git
        dest: "{{ clone_path }}"
      register: clonestat

    - name: "Health Check Disable"
      when: clonestat.changed
      file:
        path: /var/www/html/health.txt
        state: file
        mode: 000
    
    - name: "Ec2 Instance offload(OutofService)"
      when: clonestat.changed
      pause: 
        seconds: "{{ health_check_time }}"

    - name: "Copying Cloned contents to Document_root of instances"
      when: clonestat.changed
      copy:
        src: "{{ clone_path }}"
        dest: /var/www/html/
        remote_src: true
    
    - name: "httpd Restart/Enable"
      when: clonestat.changed
      service:
        name: httpd
        state: restarted
        enabled: true
    
    - name: "Health Check Enable"
      when: clonestat.changed
      file:
        path: /var/www/html/health.txt
        state: file
        mode: 0644

    - name: "Returning Ec2 instance to Inservice"
      when: clonestat.changed
      pause:
        seconds: "{{ health_check_time }}"
        prompt: "The instance will be back in a moment"
```
```sh
#vim rolling.vars
---
region: "ap-south-1"
tag_name: "asg-01"
private_key: "ansible.pem"
health_check_time: 15
gitRepository: "https://github.com/AkhiljithPB/techcafe-web-dev.git"
clone_path: /usr/local/src/git/
```
📃  Userdata for launch configuration

```sh
#!/bin/bash
yum install httpd php git -y
git clone https://github.com/AkhiljithPB/techcafe-web-dev.git /usr/local/src/
cp -apf /usr/local/src/* /var/www/html/
service httpd restart
chkconfig httpd on
```

Finally we have to setup Jenkins for configuring the continues integration.

```sh
#amazon-linux-extras install epel -y
#yum install java-1.8.0-openjdk-devel -y
#wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
#rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
#yum -y install jenkins
#systemctl start jenkins
#systemctl enable jenkins
```
The default port for jenkins is _8080_.
```sh
http://<public_ip>:8080
```
After login to the Jenkins, we have to install the required plugins.
Please follow below steps:
> Manage Jenkins --> Manage Plugins
- search for ansible plugin and install the same.

To configure the ansible for integration,

> Manage Jenkins --> Global Tool Configuration 
 
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/ans1.png "image Title")

Here, update the ansible with executable path.

Next create a free style project from _Dashboard --> New Item_ and update the Git repository url under Source Code Management and _SAVE_.
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/git-jen.png "image Title")
Then update build procedure from Build Environment to invoke the ansible-playbook.
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/build.png "image Title")

> Update the ansible palybook file location

![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/4.png "image Title")

> After configuring the anisble from jenkins like above then enable git webhooks to alert the playbook to trigger.
 
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/694ffefc85158210feb113ce870f752b137c8160/webhooks.png "image Title")

> Update the Jenkins configuration as such to trigger when GitHub pushes.
 
![image text](https://github.com/AkhiljithPB/Rolling-update-Ansible-Jenkins-ASG/blob/0975f1813832081bad99900d370fadf5873524fb/jenkins%20trigger.png "image Title")


### Setup now finished. 👼
