# Deployment of 3-tier web app using ansible

In the project we deploy 3-tier web app with help of ansible-playbook tool. Web app consists of the following components:
* nginx
* uSWGI + Python (Flask)
* Postgres

## Prerequisites

We need Linux Ubuntu server 18.04 LTS with running sshd service. The server plays *Ansible Host* role.

### Setup *Ansible Host*

Let's implement *Ansible Host* as a Docker container.
1. Build a Docker image.

```
[webapp_ansible_deploy]% docker build -t ubuntu-sshd:18.04 .
```
<details>
  <summary>There is a Dockerfile for this:</summary>

```dockerfile
# Base image
FROM ubuntu:18.04

# Install packets
RUN apt-get update && apt-get install -y python openssh-server

# To set up SSHd service in a container
RUN mkdir /var/run/sshd
RUN echo 'root:secret' | chpasswd
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

ENV NOTVISIBLE "in users profile"
RUN echo "export VISIBLE=now" >> /etc/profile

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

</details>

2. Create and start a container

**Actually, this is an optional stage which can be implemented by ansible task `Start the container 'ubuntu_18.04' if it's not up`**.

```
[webapp_ansible_deploy]$ docker run -d -p 127.0.0.1:22:22 --name ubuntu_18.04 ubuntu-sshd:18.04
```

Check if the container is up:
```
[webapp_ansible_deploy]$ docker ps
CONTAINER ID        IMAGE               COMMAND               CREATED             STATUS              PORTS                  NAMES
9381672c0d12        ubuntu-sshd:18.04   "/usr/sbin/sshd -D"   5 seconds ago       Up 4 seconds        127.0.0.1:22->22/tcp   ubuntu_18.04
```
We have to be able to connect to the container's SSHd service via 127.0.0.1:22 host socket.

## Deployment

### 1. Setup use of 10.0.0.2/16 static IP address, Netmask 255.255.0.0, gateway 10.0.0.1/16
Create ansible role `container-net-setup`. roles/container-net-setup/tasks/main.yml:
```yaml
---
- name: Create the network 10.0.0.0/16
  docker_network:
    name: network-10.0.0.0_16
    ipam_config:
      - subnet: 10.0.0.0/16
        gateway: 10.0.0.1

- name: Start the container 'ubuntu_18.04' if it's not up
  docker_container:
    # The container 'ubuntu_18.04' should be existed and started.
    # If it doesn't exist then create it using ubuntu-sshd:18.04 image and start.
    name: ubuntu_18.04
    image: ubuntu-sshd:18.04
    published_ports: 127.0.0.1:22:22
    state: started

- name: Assign the ip 10.0.0.2 and connect the container to the network
  docker_container:
    name: ubuntu_18.04
    networks:
      - name: network-10.0.0.0_16
        ipv4_address: "10.0.0.2"
    state: started
    # Suppress DEPRECATION WARNING
    networks_cli_compatible: yes
```



