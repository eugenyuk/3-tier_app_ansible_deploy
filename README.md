# Deployment of 3-tier web app using ansible

In the project we deploy 3-tier web app with help of ansible-playbook tool. Web app consists of the following components:
* nginx
* uSWGI + Python (Flask)
* Postgres

## Prerequisites

We need Linux Ubuntu server 18.04 LTS with running sshd service. The server plays *Ansible Host* role.

### Setup *Ansible Host*

