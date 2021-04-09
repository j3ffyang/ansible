# Ansible Deployment Workflow

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Pre-requisite](#pre-requisite)
- [Architecture Overview **](#architecture-overview)
- [Deployment Workflow](#deployment-workflow)
  - [Prototype Environment **](#prototype-environment)
  - [System Hardening](#system-hardening)
    - [`sshd`](#sshd)
    - [`firewalld`](#firewalld)
    - [Firewall Rules for `kubeadm`](#firewall-rules-for-kubeadm)
  - [Ansible Env Setup](#ansible-env-setup)
  - [Operating System](#operating-system)
    - [Delete Unused Packages - `role: os_pkg_rm`](#delete-unused-packages-role-os_pkg_rm)
    - [Set `/etc/hosts` - `role: os_hosts_mod`](#set-etchosts-role-os_hosts_mod)
    - [Set `hostname` to All Nodes](#set-hostname-to-all-nodes)
    - [Create a `nonroot` user on all nodes, including `control` node](#create-a-nonroot-user-on-all-nodes-including-control-node)
  - [Docker](#docker)
    - [Install docker ** (containerd.io)](#install-docker-containerdio)
    - [Point to `quay.io` for docker image, instead of `dockerhub.com`](#point-to-quayio-for-docker-image-instead-of-dockerhubcom)
    - [Private Docker Registry (optional)](#private-docker-registry-optional)
  - [Kubernetes (specific version)](#kubernetes-specific-version)
    - [Disable `swap`](#disable-swap)
    - [Install Kubernetes](#install-kubernetes)
    - [Init the cluster on `master` only](#init-the-cluster-on-master-only)
    - [Grant Permission to `ubuntu` to Manage Kubernetes](#grant-permission-to-ubuntu-to-manage-kubernetes)
    - [Install `flannel` Network Plugin (master only)](#install-flannel-network-plugin-master-only)
    - [Create `kubeadm join` Command](#create-kubeadm-join-command)
    - [Join `master` in Kubernetes cluster](#join-master-in-kubernetes-cluster)
    - [Components on Kubernetes](#components-on-kubernetes)
    - [Nginx and Ingress Network Controller on Kubernetes](#nginx-and-ingress-network-controller-on-kubernetes)
    - [Create persistentVolume?](#create-persistentvolume)
    - [Kubernetes upgrade for an existing cluster](#kubernetes-upgrade-for-an-existing-cluster)
    - [Airgap Docker and Kubernetes Install](#airgap-docker-and-kubernetes-install)
    - [Configure Kubernetes HA](#configure-kubernetes-ha)
  - [Install `helm3`](#install-helm3)
  - [Upfront Nginx Web Server on VM(s)](#upfront-nginx-web-server-on-vms)
    - [Reference](#reference)
    - [Beyond this point, VANTIQ deployment can start from now](#beyond-this-point-vantiq-deployment-can-start-from-now)
- [Appendix](#appendix)
    - [Full HA of Nginx](#full-ha-of-nginx)
    - [Comparison between `terraform` and `ansible`](#comparison-between-terraform-and-ansible)
    - [Sample code](#sample-code)

<!-- /code_chunk_output -->

## Objective

- This is an environment for production and it is not an airgap scenario
- Simplify deployment of a Kubernetes platform and a bunch of common applications, such as Nginx, PostgreSQL, Prometheus, etc
- Deploy bastion-host (DMZ) and upfront Nginx webServer, manage system hardening, configure firewall
- Define a kind of standardized environment where VANTIQ product can be deployed, as well as operational and manage-able, other than conforming cloud service provider

## Pre-requisite

- An existing virtual_machine platform. Doesn't matter they're from OpenStack, or VMware, or AWS/ Azure (deploy-able by Ansible or Terraform)
- The virtual_machines are networked functionally. `bastion-host` and `vantiqSystem`, in the next figure, can be in separate sub-network (eg, `10.0.10.0/24` and `10.0.20.0/24`) or the same. If in separate networks, they must be accessible to each other
- Need several block-disks for MongoDB and others
- A DNS that can resolve service.domain.com, or a local `/etc/hosts` must be modified as well as `nginx-ingress` accordingly

## Architecture Overview **

```plantuml

cloud freeWorld

package bastion-host {
  package webServer {
    collections nginx
    collections lb
    collections ha

    nginx -[hidden]> lb
    lb -[hidden]> ha
  }

  package vpn {
    collections wireGuard
  }

  webServer -[hidden]> vpn
}


package kubernetes {
  package vantiqSystem {

  component nginx_ingress
  component VANTIQ

  nginx_ingress -[hidden]> VANTIQ
  }

  package operation {
    component prometheus
  }
}

freeWorld -- webServer
webServer --> nginx_ingress
```

## Deployment Workflow

### Prototype Environment **

name | ip | spec
-- | -- | --
master0 | 10.39.64.10 | 2c4g
worker0 | 10.39.64.20 | 2c8g
worker1 | 10.39.64.21 | 2c8g

### System Hardening

This is specifically setup for bastion-host. __Only__ bastion-host can be accessed from internet

#### `sshd`
- `/etc/ssh/sshd_config`
```sh
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
```

#### `firewalld`
```sh
  ubuntu@master0:~/ansible$ sudo firewall-cmd --list-all
  public
    target: default
    icmp-block-inversion: no
    interfaces:
    sources:
    services: ssh dhcpv6-client http https
    ports: 12345/udp
    protocols:
    masquerade: yes
    forward-ports:
    source-ports:
    icmp-blocks:
    rich rules:
```

#### Firewall Rules for `kubeadm`

```sh
ubuntu@master0:~$ firewall-cmd --get-active-zones
docker
  interfaces: docker0

ubuntu@master0:~$ firewall-cmd --get-default-zone
public

ubuntu@master0:~$ sudo firewall-cmd --permanent --zone=internal --add-interface=eth0
ubuntu@master0:~$ sudo firewall-cmd --reload
success

ubuntu@master0:~$ firewall-cmd --get-default-zone
public
ubuntu@master0:~$ firewall-cmd --get-active-zones
docker
  interfaces: docker0
internal
  interfaces: eth0

ubuntu@master0:~$ sudo firewall-cmd --permanent --zone=internal --set-target=ACCEPT
success
ubuntu@master0:~$ sudo firewall-cmd --reload
success
```

Allow all traffic for `internal` network on `eth0` interface only


- Kernel tuning and enable `ip_forward` for `masquerade`


- Port open (VM level. Still need configuration in security group from cloud service provider):
- TCP: `80`, `443`, `22`
- UDP: `12345` for wireGuard





### Ansible Env Setup

- Pre-requisite: `ansible`, `python3`, and `pip3`

```sh
sudo apt-add-repository ppa:ansible/ansible
sudo apt update
sudo apt install ansible
sudo apt install -y python3-pip
```

```sh
ubuntu@master0:~/ansible$ python3 -V
Python 3.6.9
```

- Setup accounts and access

Follow the instruction to setup a `master0` and two `worker0` and `worker1` in the environment. And copy sshKey from `master0` into two workers by `ssh-copy-id`, so you can access worker by `root`

```sh
ubuntu@master0:~$ ssh root@worker1
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 5.4.0-1041-azure x86_64)
...
*** System restart required ***
Last login: Thu Mar 25 11:02:14 2021 from 10.39.64.10
root@worker1:~#
```

- Directory structure

tl;dr

```sh
ubuntu@master0:~/ansible$ tree
.
├── ansible.cfg
└── inventory
    └── hosts

1 directory, 2 files
```
- Important Configurations **

  - `ansible.cfg`
  ```sh
  ubuntu@master0:~/ansible$ cat ansible.cfg
  [defaults]

  inventory	= 	inventory/hosts
  ```

  - `inventory/hosts` - all nodes and associated IPs are here under Ansible **

  ```sh
  ubuntu@master0:~/ansible$ cat inventory/hosts
  [worker]
  worker0	ansible_host=10.39.64.20
  worker1 ansible_host=10.39.64.21

  [controller]
  master0 ansible_host=10.39.64.10

  [all:vars]
  ansible_python_interpreter=/usr/bin/python3
  ```

  - `global.yaml` - global variable

  ```sh
  ubuntu@master0:~/ansible$ cat global.yaml

  uusername: nonroot
  vaulted_passwd: secret
  ```

- Test Connections

```sh
ubuntu@master0:~/ansible$ ansible all -m ping -u root
worker0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
worker1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
master0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

> Reference > https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-18-04

- Initialize `roles`

`roles` naming pattern: `segment_target_exec`, eg. `os_host_mod`. All `roles` go to `~/ansible/roles`

Create a `role` with `task`

```sh
mkdir -p ~/ansible/roles
cd ~/ansible/roles

ansible-galaxy init os_pkg_rm
- Role roles was created successfully

ubuntu@master0:~/ansible/roles/os_pkg_rm$ tree
.
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

> Reference > https://www.decodingdevops.com/ansible-apt-module-examples/

- `main.yaml` - main playbook pattern

```sh
ubuntu@master0:~/ansible$ cat main.yaml

- hosts: all
  become: true

  roles:
    # - { role: os_hostname_display, when: "inventory_hostname in groups['all']" }
    # - { role: os_pkg_rm, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hosts_mod, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hosts_cp, when: "inventory_hostname in groups['worker']" }
    # - { role: os_hostname_set, when: "inventory_hostname in groups['worker']" }
    # - { role: os_ssh_auth, when: "inventory_hostname in groups['worker']" }
    # - { role: docker_install, when: "inventory_hostname in groups['all']" }
    # - { role: os_usr_create, when: "inventory_hostname in groups['worker']" }
    # - { role: os_swap_disable, when: "inventory_hostname in groups['all']" }
    # - { role: k8s_install, when: "inventory_hostname in groups['all']" }
    # - { role: k8s_init, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_usr_grant, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_flannel, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_join_cmd, when: "inventory_hostname in groups['master']" }
    - { role: k8s_join_node, when: "inventory_hostname in groups['worker']" }
```

- Standard playbook execution command

__Make sure comment out `roles` in `main.yaml` for particular action you want to perform__. Uncomment out all to execute all actions in one click. The `role` value in `main.yaml` must be identical to the one created by `ansible-galaxy init`

```sh
ansible-playbook --extra-vars @global.yaml main.yaml
```

> Will explain `global.yaml` later in this document

### Operating System

#### Delete Unused Packages - `role: os_pkg_rm`

  ```sh
  cat roles/os_pkg_rm/tasks/main.yml

  ---
  # tasks file for os_pkg_rm
  - name: Remove "snapd" package
    apt:
      name: snapd
      state: absent

  - name: Remove "ufw" package
    apt:
      name: ufw
      state: absent

  - name: Remove dependencies that are no longer required
    apt:
      autoremove: yes
  ```

This task won't trigger the actions of

  ```sh
  systemctl stop ufw
  systemctl disable ufw
  ```

But it doesn't matter

#### Set `/etc/hosts` - `role: os_hosts_mod`

- `ansible_host`

This file manages all hosts and their IPs

```sh
ubuntu@master0:~/ansible$ cat inventory/hosts

[worker]
worker0	ansible_host=10.39.64.20
worker1 ansible_host=10.39.64.21

[controller]
master0 ansible_host=10.39.64.10

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

- `templates` in `jinja2` format **

```sh
ubuntu@master0:~/ansible$ cat roles/os_hosts_mod/templates/hosts.j2

{% for item in groups["all"] %}
{{hostvars[item]['ansible_host']}} {{hostvars[item]['inventory_hostname']}}
{% endfor %}
```

The variable of `ansible_host` requires `inventory/hosts` contains `ansible_host` values

- `tasks`

```sh
ubuntu@master0:~/ansible$ cat roles/os_hosts_mod/tasks/main.yml
---
# tasks file for os_hosts_mod

- name: Copy hosts
  template:
    src: hosts.j2
    dest: /etc/hosts
```

> Reference > https://www.howtoforge.com/ansible-guide-manage-files-using-ansible/

#### Set `hostname` to All Nodes

According to `/etc/hosts` on `control node`

  ```sh
  ubuntu@master0:~/ansible$ cat roles/os_hostname_set/tasks/main.yml
  ---
  # tasks file for os_hostname_set

  - name: hostnamectl set-hostname
    hostname:
      name: "{{inventory_hostname}}"
  ```

#### Create a `nonroot` user on all nodes, including `control` node

  This supposes to create a `nonroot` user, to manage Docker and Kubernetes, in case a specific user has been occupied already in customer's environment. For example, `ubuntu` user is not allowed to be used.

  ```sh
  ubuntu@master0:~/ansible$ cat global.yaml

  uusername: nonroot
  vaulted_passwd: secret
  ```


  ```sh
  ubuntu@master0:~/ansible$ cat roles/os_usr_create/tasks/main.yml
  ---
  # tasks file for os_usr_create

  - name: Create a nonroot user
    user:
      name: "{{ uusername }}"
      password: "{{ vaulted_passwd | password_hash('sha512') }}"
      shell: /bin/bash
      groups: docker
      append: yes
      generate_ssh_key: yes
      ssh_key_bits: 2048
      ssh_key_file: .ssh/id_rsa
  ```

  ```sh
  ansible-playbook --extra-vars @global.yaml main.yaml
  ```


- ~~Setup `sshd` and `ssh-copy-id` to all worker-node~~

This action has been done before setting up Ansible as pre-requisite, unless otherwise you want to have another userId

- Kernel tuning: `inode`, `ulimit`, etc

### Docker
#### Install docker ** (containerd.io)

```sh
ubuntu@master0:~/ansible$ cat roles/docker_install/tasks/main.yml
---
# tasks file for docker_install

- name: Install apt packages
  apt:
    update_cache: yes
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common
    state: present

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker apt repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release | lower }} stable
    state: present

- name: Install docker
  apt:
    update_cache: yes
    name: "{{ packages }}"
    state: present
    update _cache: yes
  vars:
    packages:
    - docker-ce
    - docker-ce-cli
    - containerd.io
  notify:
    - docker status
```

> Reference >
- https://horrell.ca/2020/06/18/installing-docker-on-ubuntu-with-ansible/
- https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/ # updated 20210405 for `containerd`

#### Point to `quay.io` for docker image, instead of `dockerhub.com`

#### Private Docker Registry (optional)

This step is to prevent too many images from being downloaded over internet

---

### Kubernetes (specific version)

#### Disable `swap`

```sh
- name: Remove swapfile from /etc/fstab
  mount:
    name: "{{ item }}"
    fstype: swap
    state: absent
  with_items:
    - swap
    - none

- name: Disable swap
  command: swapoff -a
  when: ansible_swaptotal_mb > 0
```

> Reference > https://kubernetes.io/blog/2019/03/15/kubernetes-setup-using-ansible-and-vagrant/

#### Install Kubernetes

```sh
- name: Add an apt signing key for Kubernetes
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Adding apt repository for Kubernetes
  apt_repository:
    repo: deb https://apt.kubernetes.io/ kubernetes-xenial main
    state: present
    filename: kubernetes.list

- name: Install Kubernetes binaries
  apt:
    name: "{{ packages }}"
    state: present
    update_cache: yes
  vars:
    packages:
      - kubelet
      - kubeadm
      - kubectl

- name: Configure node ip
  lineinfile:
    path: /etc/default/kubelet
    line: KUBELET_EXTRA_ARGS=--node-ip={{ node_ip }}

- name: Restart kubelet
  service:
    name: kubelet
    daemon_reload: yes
    state: restarted
```

#### Init the cluster on `master` only

```sh
- name: Initialize the Kubernetes cluster using kubeadm
  command: kubeadm init --apiserver-advertise-address="192.168.50.10" --apiserver-cert-extra-sans="192.168.50.10"  --node-name k8s-master --pod-network-cidr=192.168.0.0/16
```

#### Grant Permission to `ubuntu` to Manage Kubernetes

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_usr_grant/tasks/main.yml
---
# tasks file for k8s_usr_grant
- name: Setup kubeconfig for ubuntu user
  command: "{{ item }}"
  with_items:
   - mkdir -p /home/ubuntu/.kube
   - cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config
   - chown ubuntu:ubuntu /home/ubuntu/.kube/config
```

You have to logout then login again to pick up the change!

#### Install `flannel` Network Plugin (master only)

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_flannel/tasks/main.yml
---
# tasks file for k8s_flannel
- name: Install flannel pod network
  become: false
  command: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> Reference > https://github.com/flannel-io/flannel

#### Create `kubeadm join` Command

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_join_cmd/tasks/main.yml
---
# tasks file for k8s_join_cmd
- name: Generate join command
  command: kubeadm token create --print-join-command
  register: join_command

- name: Copy join command to local file
  local_action: copy content="{{ join_command.stdout_lines[0] }}" dest="./join-command"
```

#### Join `master` in Kubernetes cluster

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_join_node/tasks/main.yml
---
# tasks file for k8s_join_node
- name: Copy the join command to server location
  copy: src=join-command dest=/tmp/join-command.sh mode=0777

- name: Join the node to cluster
  command: sh /tmp/join-command.sh
```

#### Components on Kubernetes
- `prometheus` for monitoring
- `postgreSQL` for Keycloak
- `cert-manager` for automated certificate issuing

#### Nginx and Ingress Network Controller on Kubernetes
- Apply custom SSL
- ~~Apply VANTIQ license key~~

#### Create persistentVolume?

#### Kubernetes upgrade for an existing cluster

#### Airgap Docker and Kubernetes Install

#### Configure Kubernetes HA

### Install `helm3`

Follow the instruction at https://helm.sh/docs/intro/install/

```sh
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### Upfront Nginx Web Server on VM(s)
- HA. Refer to Full HA of Nginx in Appendix
- LB
  - Get all `worker_node` from Kubernetes
  - Get `nodePort` from `service` in Kubernetes
  - Put the above into `/etc/nginx/nginx.conf` in upfront Nginx webServer(s)
- Perf tuning, caching

#### Reference

- Copy a file

  ```sh
  cat roles/os_hosts_cp/tasks/main.yml
  ---
  # tasks file for os_mod_hosts
  - name: Insert multiple lines and Backup
    blockinfile:
      path: /etc/hosts
      backup: yes
      block: |
        10.39.64.10	master0
        10.39.64.20	worker0
        10.39.64.21	worker1
  ```

#### Beyond this point, VANTIQ deployment can start from now
---



## Appendix

#### Full HA of Nginx

If there are 2 virtualIPs, bind them to both Nginx web server to maximize the utilization of 2 Nginx web servers simultaneously on production

```plantuml

cloud freeWorld

rectangle "dnsQuery: pis.truedigital.com" as dns

rectangle "publicVIP: 159.138.238.56" as pubvip

rectangle "priVIP: 10.0.20.13" as privip
rectangle "priVIP2: 10.0.20.14" as privip2

package nginx-web1 {
  rectangle "pubIP1: 159.138.254.213" as pubip1
  rectangle "internalIP1: 10.0.20.11" as intip1

  pubip1 -[hidden]- intip1
}

package nginx-web2 {
  rectangle "pubIP2: 159.138.235.98" as pubip2
  rectangle "internalIP2: 10.0.20.12" as intip2

  pubip2 -[hidden]- intip2
}

freeWorld --> dns
dns --> pubvip

pubvip --> privip
pubvip --> privip2

privip --> intip1
privip ..> intip2

privip2 ..> intip1
privip2 --> intip2
```

#### Comparison between `terraform` and `ansible`

Clarification: there is no intention to have them compete each other. Actually they're not competitors either. It's not that when one is preferred and another one would be completely wrong. The reason of picking one is that better fits scenario for both technical and business perspectives.

Even the decision made might not be proper for any reason for longer term, such as tool deprecated, community no longer supported, we would be able to manage the switching effort at an acceptable and reasonable cost, to avoid disruptive change.

Whenever picking one is not intentional to replace another. Both Ansible and Terraform tools do a lot of things pretty well.

> And my personal preference is to **use Terraform for orchestration/ provisioning and Ansible for configuration management**.

**There is still some overlapped functions**

> When it comes to orchestration you can use the orchestration tools to not only provision servers, but also databases, caches, load balancers, queues, monitoring, subnet configuration, firewall settings, routing rules, SSL certificates, and almost every other aspect of your infrastructure, mainly public cloud infrastructure.

I can't agree 100%
> Ansible uses procedural style where you write the code that specifies, step-by-step tasks in order to achieve desired end state.

> Ref > https://linuxhandbook.com/terraform-vs-ansible/

Because

> Ansible allows you to write is a declarative code. Even though it executes the tasks in a serial order, which is why a lot of people think its procedural in a way. Here are the reasons why I believe Ansible is declarative,

> - Ansible allows you to write tasks where you focus on WHAT you want rather than HOW to achieve it. These tasks are then mapped to some underlying code (typically python however you could create modules in any language), which is a procedural code and is platform specific.
> - Ansible uses YAML as the language to create the playbooks, its way to define infrastructure as a code. Inherently YAML is a declarative language. All the tools that use YAML are essentially creating a declarative interface for their users e.g. kubernetes, docker compose etc.



> Ref > https://www.quora.com/Is-Ansible-procedural-or-declarative


#### Sample code

```sh
ubuntu@master0:~/ansible$ ansible all -a "df -h" -u root
master0 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
tmpfs           394M  712K  393M   1% /run
/dev/sdb1       124G  2.2G  122G   2% /
tmpfs           2.0G  124K  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/sdb15      105M  6.1M   99M   6% /boot/efi
/dev/sda1        16G   45M   15G   1% /mnt
tmpfs           394M     0  394M   0% /run/user/1000
tmpfs           394M     0  394M   0% /run/user/0
worker0 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           797M  684K  796M   1% /run
/dev/sda1       124G  1.7G  123G   2% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs           797M     0  797M   0% /run/user/0
worker1 | CHANGED | rc=0 >>
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
tmpfs           797M  684K  796M   1% /run
/dev/sda1       124G  1.7G  123G   2% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda15      105M  6.1M   99M   6% /boot/efi
tmpfs           797M     0  797M   0% /run/user/0
```
