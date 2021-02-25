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
## with_items
```yaml
command: echo {{ item }}
with_items: "{{ list }}"

---
command: echo {{ item }}
with_items:
  - a
  - b
  - c
```

## templating (Jinja2)
- dynamic expression이 가능해짐
- variable에 접근이 가능해짐
- templating은 task가 target machine으로 보내지기 전에 Ansible Controller에서 발생한다.

### ansible.builtin.template
- file을 templating해서 target host로 보낸다.
- Jinja2 templating language support
```yaml
- name: Template a file to /etc/file.conf
  template:
    src: tpl.conf.j2
    dest: /etc/file.conf
    owner: bin
    group: wheel
    mode: '0664
```

### Jinja Syntax
- {% ... %} : Statement
- {{ ... }} : Expressions
- {# ... #} : Comments
- `# ... ##  : Line statements`

#### Loop Exp
```jinja2
{% for row in rows %}
  <li class="{{ loop.cycle('odd', 'even') }}"> {{ row }}}</li>
{% endfor %}
# loop.cycle이 special variable
```
| Variable | Description |
|----------|-------------|
|loop.index| cur iter(1 indexed)|
|loop.indexo| cur iter(0 indexed)|
|loop.revindex| 뒤에서부터 iter (1 indexed)|
|loop.revindexo | 뒤에서부터 iter (0 indexed) |
| loop.first | true if first iter|
| loop.last | true if last iter|
| loop.length | item 수 |
| loop.cycle | 