---
author: "李昌"
title: "vagrant+ansible安装k8s集群"
date: "2021-09-06"
tags: ["kubernetes", "install", "vagrant"]
categories: [""]
ShowToc: true
TocOpen: true
---

**已过时，不可用**

> 部署环境： ubuntu20.04， 8G+4核
> kubernete版本： 1.22.1

## 1. 安装vagrant和ansible
按官网教程即可

## 2. Vagrantfile
建立如下目录
```tree
k8s-cluster
├── kubernetes-setup
│   ├── master-playbook.yml
│   └── node-playbook.yml
└── Vagrantfile
```

其中，Vagrantfile内容如下：
```
IMAGE_NAME = "bento/ubuntu-16.04"
N = 2

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 2048
        v.cpus = 2
    end

    config.vm.define "k8s-master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.hostname = "k8s-master"
        master.vm.provision "ansible" do |ansible|
            ansible.playbook = "kubernetes-setup/master-playbook.yml"
            ansible.extra_vars = {
                node_ip: "192.168.50.10",
            }
        end
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
            node.vm.provision "ansible" do |ansible|
                ansible.playbook = "kubernetes-setup/node-playbook.yml"
                ansible.extra_vars = {
                    node_ip: "192.168.50.#{i + 10}",
                }
            end
        end
    end
end
```

master-playbook.ym内容如下：
```yaml
---
- hosts: all
  become: true
  tasks:
  - name: Install packages that allow apt to be used over HTTPS
    apt:
      name: "{{ packages }}"
      state: present
      update_cache: yes
    vars:
      packages:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common

  - name: Add an apt signing key for Docker
    apt_key:
      url: http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg
      state: present

  - name: Add apt repository for stable version
    apt_repository:
      repo: deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial stable
      state: present

  - name: Install docker and its dependecies
    apt:
      name: "{{ packages }}"
      state: present
      update_cache: yes
    vars:
      packages:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    notify:
      - docker status

  - name: Add vagrant user to docker group
    user:
      name: vagrant
      group: docker

  - name: Change Cgroup Driver from cgroupfs to systemd
    shell: |
      echo '{
         "exec-opts": ["native.cgroupdriver=systemd"]
      }' >> /etc/docker/daemon.json

  - name: Restart docker
    shell: |
       systemctl daemon-reload
       systemctl restart docker

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

  - name: Add an apt signing key for Kubernetes
    apt_key:
      url: https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
      state: present

  - name: Adding apt repository for Kubernetes
    apt_repository:
      repo: deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
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
      path: /usr/bin/kubelet
      line: KUBELET_EXTRA_ARGS=--node-ip={{ node_ip }}

  - name: Restart kubelet
    service:
      name: kubelet
      daemon_reload: yes
      state: restarted

  - name: Pull CoreDns:v1.8.4
    shell: |
        docker pull registry.aliyuncs.com/google_containers/coredns:latest
        docker tag registry.aliyuncs.com/google_containers/coredns:latest registry.aliyuncs.com/google_containers/coredns:v1.8.4
        docker rmi registry.aliyuncs.com/google_containers/coredns:latest

  - name: Initialize the Kubernetes cluster using kubeadm
    command: kubeadm init --apiserver-advertise-address="192.168.50.10" --apiserver-cert-extra-sans="192.168.50.10"  --node-name k8s-master --pod-network-cidr=192.168.0.0/16 --image-repository=registry.aliyuncs.com/google_containers

  - name: Setup kubeconfig for vagrant user
    command: "{{ item }}"
    with_items:
     - mkdir -p /home/vagrant/.kube
     - cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
     - chown vagrant:vagrant /home/vagrant/.kube/config

  - name: Install calico pod network
    become: false
    command: kubectl create -f https://docs.projectcalico.org/manifests/calico.yaml

  - name: Generate join command
    command: kubeadm token create --print-join-command
    register: join_command

  - name: Copy join command to local file
    shell: |
     echo hello | tee /vagrant/hello
     kubeadm token create --print-join-command | tee /vagrant/join-command

    # local_action: copy content="{{ join_command.stdout_lines[0] }}" dest="{{ playbook_dir }}/join-command"

  handlers:
    - name: docker status
      service: name=docker state=started
```

node-playbook.yml内容如下：
```yaml
---
- hosts: all
  become: true
  tasks:
  - name: Install packages that allow apt to be used over HTTPS
    apt:
      name: "{{ packages }}"
      state: present
      update_cache: yes
    vars:
      packages:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg-agent
      - software-properties-common

  - name: Add an apt signing key for Docker
    apt_key:
      url: http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg
      state: present

  - name: Add apt repository for stable version
    apt_repository:
      repo: deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial stable
      state: present

  - name: Install docker and its dependecies
    apt:
      name: "{{ packages }}"
      state: present
      update_cache: yes
    vars:
      packages:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    notify:
      - docker status

  - name: Add vagrant user to docker group
    user:
      name: vagrant
      group: docker

  - name: Change Cgroup Driver from cgroupfs to systemd
    shell: |
      echo '{
         "exec-opts": ["native.cgroupdriver=systemd"]
      }' >> /etc/docker/daemon.json

  - name: Restart docker
    shell: |
       systemctl daemon-reload
       systemctl restart docker

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

  - name: Add an apt signing key for Kubernetes
    apt_key:
      url: https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
      state: present

  - name: Adding apt repository for Kubernetes
    apt_repository:
      repo: deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
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
      path: /usr/bin/kubelet
      line: KUBELET_EXTRA_ARGS=--node-ip={{ node_ip }}

  - name: Restart kubelet
    service:
      name: kubelet
      daemon_reload: yes
      state: restarted

  - name: Copy the join command to server location
    shell: |
        cat /vagrant/join-command >> /tmp/join-command.sh
        chmod +x /tmp/join-command.sh

  - name: Join the node to cluster
    command: sh /tmp/join-command.sh

  handlers:
    - name: docker status
      service: name=docker state=started
```

## 3. 启动集群
```sh
cd /path/to/k8s-cluster/

vagrant up
```

耐心等待一段时间，三个节点的k8s集群就启动成功了。
