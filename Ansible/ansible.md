# Ansible
- IT automation tool
- system 설정, software 배포, cd, zero downtime rolling update를 지원하는 도구
- 주요 목표는 simplicity & ease-of-use
## playbook
- 간단히 config 관리, 다수의 머신에 대한 배포 시스템의 기본적인 단위
```yaml
---
# This playbook deploys the whole application stack in this site.

# Apply common configuration to all hosts
- hosts: all

  roles:
  - common

# Configure and deploy database servers.
- hosts: dbservers

  roles:
  - db

# Configure and deploy the web servers. Note that we include two roles
# here, the 'base-apache' role which simply sets up Apache, and 'web'
# which includes our example web application.

- hosts: webservers

  roles:
  - base-apache
  - web

# Configure and deploy the load balancer(s).
- hosts: lbservers

  roles:
  - haproxy

# Configure and deploy the Nagios monitoring node(s).
- hosts: monitoring

  roles:
  - base-apache
  - nagios
```