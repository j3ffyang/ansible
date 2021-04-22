# Ansible Deployment Workflow

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Pre-requisite](#pre-requisite)
- [Architecture Overview](#architecture-overview)
- [Deployment Workflow](#deployment-workflow)
  - [Prototype Environment](#prototype-environment)
  - [System Hardening](#system-hardening)
    - [`sshd`](#sshd)
    - [`firewalld`](#firewalld)
    - [Firewall Rules for `kubeadm`](#firewall-rules-for-kubeadm)
  - [Ansible Env Setup](#ansible-env-setup)
  - [Operating System](#operating-system)
    - [Delete Unused Packages - `role: os_pkg_rm`](#delete-unused-packages-role-os_pkg_rm)
    - [Set `/etc/hosts` - `role: os_hosts_mod`](#set-etchosts-role-os_hosts_mod)
    - [Set `hostname` to All Nodes - `role: os_hostname_set`](#set-hostname-to-all-nodes-role-os_hostname_set)
    - [Create a `nonroot` user on all nodes, including `control` node - `role: os_usr_create`](#create-a-nonroot-user-on-all-nodes-including-control-node-role-os_usr_create)
  - [Docker](#docker)
    - [Install docker (containerd.io) - `role: docker_install`](#install-docker-containerdio-role-docker_install)
    - [Point to `quay.io` for docker image, instead of `dockerhub.com`](#point-to-quayio-for-docker-image-instead-of-dockerhubcom)
    - [Private Docker Registry (optional)](#private-docker-registry-optional)
  - [Kubernetes (specific version) - `1.18.18` **](#kubernetes-specific-version-11818)
    - [Disable `swap` - `role: os_swap_disable`](#disable-swap-role-os_swap_disable)
    - [Install Kubernetes - `role: k8s_install`](#install-kubernetes-role-k8s_install)
    - [Init the cluster on `master` only - `role: k8s_init`](#init-the-cluster-on-master-only-role-k8s_init)
    - [Grant Permission to `ubuntu` to Manage Kubernetes - `role: k8s_kubeconfig`](#grant-permission-to-ubuntu-to-manage-kubernetes-role-k8s_kubeconfig)
    - [Install `flannel` Network Plugin (master only) - `role: k8s_flannel`](#install-flannel-network-plugin-master-only-role-k8s_flannel)
    - [Create `kubeadm join` Command - `role: k8s_join_cmd`](#create-kubeadm-join-command-role-k8s_join_cmd)
    - [Join `master` in Kubernetes cluster - `role: k8s_join_node`](#join-master-in-kubernetes-cluster-role-k8s_join_node)
    - [Destroy Entire Kubernetes Cluster](#destroy-entire-kubernetes-cluster)
    - [(Optional) AutoComplete and Alias for `kubectl` and `kubeadm` - `role: k8s_autocompletion`](#optional-autocomplete-and-alias-for-kubectl-and-kubeadm-role-k8s_autocompletion)
    - [Create Custom persistentVolume?](#create-custom-persistentvolume)
    - [Kubernetes upgrade for an existing cluster](#kubernetes-upgrade-for-an-existing-cluster)
    - [Airgap Docker and Kubernetes Install](#airgap-docker-and-kubernetes-install)
    - [Configure Kubernetes HA](#configure-kubernetes-ha)
  - [Helm3](#helm3)
    - [Install `helm3` - `role: helm3_install` **](#install-helm3-role-helm3_install)
    - [`ingress-nginx` - `role: k8s_ingress_nginx`](#ingress-nginx-role-k8s_ingress_nginx)
    - [~~`nginx_ingress` - `role: nginx_ingress`~~](#~~nginx_ingress-role-nginx_ingress~~)
    - [`cert-manager` - `role: cert-manager`](#cert-manager-role-cert-manager)
    - [`sealedSecrets` by `kubeseal`](#sealedsecrets-by-kubeseal)
    - [Prometheus](#prometheus)
    - [Components on Kubernetes](#components-on-kubernetes)
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

## Architecture Overview

The following figure is drawn by [PlantUML](https://plantuml.com). If you use VScode or Atom.io as markdown editor, you need a plugin `markdown-preview-enhanced`(https://shd101wyy.github.io/markdown-preview-enhanced) to view it.

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

### Prototype Environment

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
- Basic Configurations

  - `ansible.cfg`

  ```sh
  [defaults]

  inventory               = inventory/hosts
  strategy                = debug   # display debug message while running
  # enable_task_debugger  = True
  ```

  - `inventory/hosts` - all nodes and associated IPs are here under Ansible

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
    #
    # - { role: docker_install, when: "inventory_hostname in groups['all']" }
    # - { role: os_usr_create, when: "inventory_hostname in groups['all']" }
    #
    # - { role: os_swap_disable, when: "inventory_hostname in groups['all']" }
    # - { role: k8s_install, when: "inventory_hostname in groups['all']" }
    # - { role: k8s_init, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_kubeconfig, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_flannel, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_join_cmd, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_join_node, when: "inventory_hostname in groups['worker']" }
    # - { role: k8s_autocompletion, when: "inventory_hostname in groups['controller']" }
    #
    # - { role: helm3_install, when: "inventory_hostname in groups['master']" }
    # - { role: k8s_ingress_nginx, when: "inventory_hostname in groups['master']" }
    - { role: cert-manager, when: "inventory_hostname in groups['master']" }
    # - { role: prometheus, when: "inventory_hostname in groups['master']" }
    # - { role: kubeseal, when: "inventory_hostname in groups['master']" }
ubuntu@master0:~/ansible$
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

- `templates` in `jinja2` format

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

#### Set `hostname` to All Nodes - `role: os_hostname_set`

According to `/etc/hosts` on `control node`

  ```sh
  ubuntu@master0:~/ansible$ cat roles/os_hostname_set/tasks/main.yml
  ---
  # tasks file for os_hostname_set

  - name: hostnamectl set-hostname
    hostname:
      name: "{{inventory_hostname}}"
  ```

#### Create a `nonroot` user on all nodes, including `control` node - `role: os_usr_create`

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
#### Install docker (containerd.io) - `role: docker_install`

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

### Kubernetes (specific version) - `1.18.18` **

#### Disable `swap` - `role: os_swap_disable`

```sh
ubuntu@master0:~/ansible$ cat roles/os_swap_disable/tasks/main.yml
---
# tasks file for os_swap_disable
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

#### Install Kubernetes - `role: k8s_install`

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_install/tasks/main.yml
---
# tasks file for k8s_install
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
      - kubelet=1.18.18-00
      - kubeadm=1.18.18-00
      - kubectl=1.18.18-00

        # - name: Configure node ip
        #   lineinfile:
        #     path: /etc/default/kubelet
        #     line: KUBELET_EXTRA_ARGS=--node-ip={{ node_ip }}
        #   vars:
        #     node_ip:
        #       - 10.39.64.10

- name: Restart kubelet
  service:
    name: kubelet
    daemon_reload: yes
    state: restarted
```

> Reference > https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04

#### Init the cluster on `master` only - `role: k8s_init`

`--apiserver-advertise-address` is `master0` IP address

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_init/tasks/main.yml
---
# tasks file for k8s_init
- name: Initialize the Kubernetes cluster using kubeadm
  command: kubeadm init --apiserver-advertise-address="10.39.64.10" --apiserver-cert-extra-sans="10.39.64.10" --node-name k8s-master --pod-network-cidr=10.244.0.0/16
```

As long as `--pod-network-cidr=10.244.0.0/16` gets set, `cni0` device IP won't be able to change until otherwise `cni0` device is deleted.

#### Grant Permission to `ubuntu` to Manage Kubernetes - `role: k8s_kubeconfig`

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_kubeconfig/tasks/main.yml
---
# tasks file for k8s_kubeconfig
# - name: Setup kubeconfig for ubuntu user
#   command: "{{ item }}"
#   with_items:
#    - mkdir -p /home/ubuntu/.kube
#    - cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config
#    - chown -R ubuntu:ubuntu /home/ubuntu/.kube

- name: create .kube directory
  become: yes
  become_user: ubuntu
  file:
    path: $HOME/.kube
    state: directory
    mode: 0755

- name: copy admin.conf to user's kube config
  copy:
    src: /etc/kubernetes/admin.conf
    dest: /home/ubuntu/.kube/config
    remote_src: yes
    owner: ubuntu
    mode: 0600
```

You have to logout then login again to pick up the change!

If `/home/ubuntu/.kube` isn't owned by `ubuntu.ubuntu`, but owned by `root.root` instead, the following error would appear

> Reference > https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04

```sh
ubuntu@master0:~/.kube$ kubectl get all --all-namespaces
I0412 08:13:35.339148   11404 request.go:668] Waited for 1.152966258s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/policy/v1?timeout=32s
I0412 08:13:45.339837   11404 request.go:668] Waited for 6.198896723s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/apiextensions.k8s.io/v1beta1?timeout=32s
I0412 08:13:56.739889   11404 request.go:668] Waited for 1.19751473s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/events.k8s.io/v1?timeout=32s
I0412 08:14:06.939134   11404 request.go:668] Waited for 3.191095759s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/admissionregistration.k8s.io/v1?timeout=32s
I0412 08:14:16.939176   11404 request.go:668] Waited for 4.997458384s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/scheduling.k8s.io/v1?timeout=32s
I0412 08:14:27.139494   11404 request.go:668] Waited for 6.99703677s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/networking.k8s.io/v1beta1?timeout=32s
I0412 08:14:37.739217   11404 request.go:668] Waited for 1.197000394s due to client-side throttling, not priority and fairness, request: GET:https://10.39.64.10:6443/apis/authentication.k8s.io/v1?timeout=32s
```


#### Install `flannel` Network Plugin (master only) - `role: k8s_flannel`

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_flannel/tasks/main.yml
---
# tasks file for k8s_flannel
- name: Install flannel pod network
  become: false
  command: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> Reference > https://github.com/flannel-io/flannel

#### Create `kubeadm join` Command - `role: k8s_join_cmd`

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

#### Join `master` in Kubernetes cluster - `role: k8s_join_node`

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_join_node/tasks/main.yml
---
# tasks file for k8s_join_node
- name: Copy the join command to server location
  copy: src=join-command dest=/tmp/join-command.sh mode=0777

- name: Join node to cluster
  command: sh /tmp/join-command.sh
```

#### Destroy Entire Kubernetes Cluster

Run the following on __each__ of nodes

```sh
sudo kubeadm reset

sudo rm -fr /etc/cni/net.d
rm -fr ~/.kube/

# if you want to reinstall or install the very latest ones
sudo apt purge kubectl kubeadm kubelet kubernetes-cni -y
sudo apt autoremove

# kubeadm reset should remove the following ones. But double check
sudo rm -fr /etc/kubernetes/; sudo rm -fr /var/lib/etcd; sudo rm -rf /var/lib/cni/

# unnecessary in most cases
sudo systemctl daemon-reload

# run this if firewalld and/ or ufw are running
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X

# remove all running docker containers
docker rm -f `docker ps -a | grep "k8s_" | awk '{print $1}'`
```

Output of the above

```sh
ubuntu@master0:~/ansible$ sudo kubeadm reset
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
[reset] Removing info for node "k8s-master" from the ConfigMap "kubeadm-config" in the "kube-system" Namespace
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```

**Remove both `cni0` and `flannel.1` devices by restarting VM**

Consult with this reference, if you want to remove `docker` as well

> Reference > https://hiberstack.com/10677/how-to-uninstall-docker-and-kubernetes/


#### (Optional) AutoComplete and Alias for `kubectl` and `kubeadm` - `role: k8s_autocompletion`

- Ansible code - run this on Ansible `controller` for `ubuntu` user only

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_autocompletion/tasks/main.yml
---
# tasks file for k8s_autocompletion
- name: update ~/.bashrc for kubectl & kubeadm autocompletion for ubuntu user
  blockinfile:
    dest: /home/ubuntu/.bashrc
    block: |
      source <(kubectl completion bash)
      source <(kubeadm completion bash)
      source <(helm completion bash)
      alias k=kubectl
      complete -F __start_kubectl k
    backup: yes
```

- Manual step - add the following into `~/.bashrc` on Ansible `controller` for `ubuntu`

```bash{.line-numbers}
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```bash
# alias and auto-completion
alias k=kubectl
complete -F __start_kubectl k
```

#### Create Custom persistentVolume?

- `lsblk`
- `fdisk` > `n` to create partition > `w` to write configuration
- `mkfs.xfs` for `mongoDB` and `mkfs.ext4` for others
- `mount` to `/mnt/disks-by-id/diskX`
- Update `/etc/fstab`
- `storageClass`

#### Kubernetes upgrade for an existing cluster

#### Airgap Docker and Kubernetes Install

#### Configure Kubernetes HA

---

### Helm3

#### Install `helm3` - `role: helm3_install` **

Up to the date that I write this document, `helm` version = `v3.5.3`

- Install `community.kubernetes` plugin
```sh
ansible-galaxy collection install community.kubernetes
```

- Ansible code

Notice that we want to use `ubuntu` user to control `helm`, there are `become` and `become_user` privilege escalation defined, as `~/ansible/main.yaml` has a global `become: true` as `root`

```sh
ubuntu@master0:~/ansible$ cat roles/helm3_install/tasks/main.yml

---
# tasks file for helm3_install
#- name: add gnupg key for codership repo
#  apt-key: keyserver=keyserver.ubuntu.com id=BC19DDBA
#
#- name: add repo
#  apt_repository: repo='deb http://ubuntu-cloud.archive.canonical.com/{{ ansible_distribution | lower }} {{ ansible_distribution_release }}-updates/liberty main' state=present

# - name: add apt-key
#   shell:
#   - /usr/bin/curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
#   - /usr/bin/sudo apt-get install apt-transport-https --yes
#
# - name: add repo
#   apt_repository: repo='deb https://baltocdn.com/helm/stable/debian/ all main' state=present
#
# - name: install helm3
#   shell: sudo apt-get update; sudo apt-get install helm
#

- name: Retrieve helm bin
  unarchive:
    src: https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz
    dest: /tmp
    creates: /usr/local/bin/helm
    remote_src: yes

- name: Move helm binary into place
  command: cp /tmp/linux-amd64/helm /usr/local/bin/helm
  args:
    creates: /usr/local/bin/helm

- name: Add a repository
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: stable
    repo_url: https://charts.helm.sh/stable
```

> Reference >
https://github.com/geerlingguy/ansible-for-devops/blob/master/kubernetes/examples/helm.yml
https://www.ansible.com/blog/automating-helm-using-ansible

#### `ingress-nginx` - `role: k8s_ingress_nginx`

```sh
ubuntu@master0:~/ansible$ cat roles/k8s_ingress_nginx/tasks/main.yml
---
# tasks file for k8s_ingress_nginx

- name: Add ingress-nginx chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: ingress-nginx
    repo_url: "https://kubernetes.github.io/ingress-nginx"

- name: Deploy latest version of ingress-nginx
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: ingress-nginx
    chart_ref: ingress-nginx/ingress-nginx
    release_namespace: ingress-nginx
    create_namespace: yes
    values:
      replicas: 1
```

#### ~~`nginx_ingress` - `role: nginx_ingress`~~

This project has been deprecated and replaced with https://kubernetes.github.io/ingress-nginx/

```sh
ubuntu@master0:~/ansible$ cat roles/nginx_ingress/tasks/main.yml
---
# tasks file for nginx-ingress

- name: Add nginx-ingress chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: nginx-ingress
    repo_url: "https://helm.nginx.com/stable"

- name: Deploy latest version of ingress-nginx
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: nginx-ingress
    chart_ref: nginx-stable/nginx-ingress
    release_namespace: nginx-ingress
    create_namespace: yes
    values:
      replicas: 1
```

#### `cert-manager` - `role: cert-manager`

Install with `crd` enabled

```sh
ubuntu@master0:~/ansible$ cat roles/cert-manager/tasks/main.yml
---
# tasks file for cert-manager
- name: Add cert-manager chart repo
  become: yes
  become_user: ubuntu
  kubernetes.core.helm_repository:
    name: jetstack
    repo_url: "https://charts.jetstack.io"

- name: Deploy latest version of cert-manager
  become: yes
  become_user: ubuntu
  kubernetes.core.helm:
    name: cert-manager
    chart_ref: jetstack/cert-manager
    release_namespace: cert-manager
    create_namespace: yes
    release_values:
      installCRDs: true
```

> Reference > https://cert-manager.io/docs/installation/kubernetes/

#### `sealedSecrets` by `kubeseal`

- Install `kubernetes.core` module, equivalent to `community.kubernetes`

```sh
ansible-galaxy collection install kubernetes.core
```

- Ansible code

```sh
ubuntu@master0:~/ansible$ cat roles/kubeseal/tasks/main.yml
---
# tasks file for kubeseal

- name: Add helm repo
  become: yes
  become_user: ubuntu
  vars:
    chart_repo_url: "https://bitnami-labs.github.io/sealed-secrets"
  kubernetes.core.helm_repository:
    name: bitnami
    repo_url: "{{ chart_repo_url }}"
```

> Reference > https://github.com/bitnami-labs/sealed-secrets


#### Prometheus

```sh
ubuntu@master0:~/ansible$ cat roles/prometheus/tasks/main.yml
---
# tasks file for prometheus
- name: Deploy latest version of Prometheus chart inside monitoring namespace (and create it)
  community.kubernetes.helm:
    name: prom
    chart_ref: stable/prometheus
    release_namespace: monitoring
    create_namespace: true
```

#### Components on Kubernetes
- `prometheus` for monitoring
- `postgreSQL` for Keycloak
- `cert-manager` for automated certificate issuing

#### Upfront Nginx Web Server on VM(s)
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

rectangle "dnsQuery: pis.malata.com" as dns

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
