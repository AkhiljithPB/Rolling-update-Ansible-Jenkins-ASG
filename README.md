# Rolling update::AutoScalingGroup::Ansible::Jenkins
## Description
###### Here, we will look into my small project to configure continues integration for a website on an existing Autoscaling group under an Elastic Load Balancer using ansible playbook  by cloning Git repository and manages the rolling update through Jenkins. I have configured the Webhook feature to alert the ansible for initiating the update.  

## Features

- Rolling update using Jenkins for Amazon Auto Scaling Group through playbook.
- Dynamic hosts inventory using ansible module "add_host"
- Seperate file for LoadBalancer health check.
- Customizable variables for IAM accesss&secrete keys, region,domain-name, document-root etc.


## Prerequisites
- Ansible.
- IAM user with required previleges.
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
- name: "Instance INformation"
  hosts: localhost

  vars:
    access: ""
    secret: ""
    region: "ap-south-1"
    ansible_python_interpreter: /usr/bin/python3
  tasks:

    - name: "ASG ip's"
      ec2_instance_info:
        aws_access_key: "{{ access }}"
        aws_secret_key: "{{ secret }}"
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": ansi-asg
          instance-state-name: [ "running"]
      register: asg_inst

    - name: "Dynamic inventory"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "ansi-asg"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: /root/admin/ansible.pem
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ asg_inst.instances }}"

- name: "Deploying techcafe.com from git"
  become: true
  hosts: ansi-asg
  gather_facts: true
  vars:
    git_repo: "https://github.com/AkhiljithPB/techcafe-web-dev.git"
    clone_path: "/usr/local/src/git/"
    doc_root: "/var/www/html/"
    health_time: 20
  tasks:

    - name: "Task-01:Installing webserver"
      yum:
        name:
          - httpd
          - git
        state: present
    - name: "Task-02:Creating clone directory"
      file:
        path: "{{ clone_path }}"
        state: directory

    - name: "Task-03:Cloning repo to clone_directory"
      git:
        repo: "{{ git_repo }}"
        dest: "{{ clone_path }}"
        key_file: /root/.ssh/github-ansi
      register: clonestat
    - debug: var=clonestat

    - name: "Task-04:Health-check disable"
      when: clonestat.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0000

    - name: "Task-05:Removing instance from ELB"
      when: clonestat.changed
      pause:
        seconds: "{{ health_check_time }}"

    - name: "Task-06:Copying files to document root"
      when: clonestat.changed
      copy:
        src: "{{ clone_path }}"
        dest: "{{ doc_root }}"
        remote_src: true

    - name: "Task-07:Restarting apache"
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Task-08:Health-check enable"
      when: clonestat.changed
      file:
        path: "/var/www/html/health.txt"
        state: file
        mode: 0644

    - name: "Task-09:Adding instance from ELB"
      when: clonestat.changed
      pause:
        seconds: "{{ health_check_time }}"
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
