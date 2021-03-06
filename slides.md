# Ansible Tricks 'n' Tips

## To keep you rocking!

---
# Who is this guy?

Lukas Sadzik, Software Developer @ Sensiolabs

Automatization fan (aka "lazy guy")

[@ko_libri](https://twitter.com/ko_libri)

lukas.sadzik@sensiolabs.de

------
# Install from git

```bash
# install requirements
$ sudo easy_install pip
$ sudo pip install paramiko PyYAML Jinja2 httplib2 six
```

```bash
# clone & source ansible
$ git clone git://github.com/ansible/ansible.git --recursive

# you may add this to your ~/.bashrc
$ source ./ansible/hacking/env-setup
```

```bash
# update
$ git pull --rebase
```

------
# Name your steps

Just do it!

*Info*:
- allows you to use the `--start-at-task` option of the `ansible-playbook` command which may save a lot of time.
- it is documentation of your scripts.
- will lead to better output and a possibility to search for a (failing) step name in your scripts.

------
# Set up your project

Keep your inventory in your SCM.

*Info*:
- but keep sensitive data out of the repository ;)

---
## Inventory: 

### Use IPs and group your hosts

```ini
[cid]
cid.ko ansible_host=192.168.178.23
cid.dev ansible_host=192.168.3.23

[aerith]
aerith.ko ansible_host=192.168.178.24

[cloud]
cloud.ko ansible_host=192.168.178.25

[arch:children]
cid
cloud

[arch:vars]
ansible_python_interpreter=/usr/bin/python2
```

*Info*:
- with this you can use `ansible_nodename` you have IP and hostname at one place. You can use `ansible_nodename` to set the hostname and `ansible_default_ipv4.address` for the ip. Think about using this in you local `/etc/hosts`.

------
# Directory layout

---
## Simple directory layout
### (from best practices)

```
.
├── ansible.cfg             <- ansible configuration file
├── group_vars              <- vars for host groups
│   └── all.yml             <- extra file for all hosts
├── host_vars               <- vars for each host
│   └── somehost.foo.yml
├── hosts                   <- the inventory file
├── roles                   <- roles dir
│   └── my_role
└── site.yml                <- your playbook
```

```ini
# ansible.cfg
[defaults]
inventory = ./hosts
```

```bash
$ ansible-playbook site.yml
```

*Info*:
- nice, when you only need one inventory file

---
## Advanced directory layout 
### (from best practices)

```
├── ansible.cfg
├── inventory
│   ├── staging
│   │   ├── group_vars
│   │   ├── host_vars
│   │   └── hosts           <- host file for "staging"
│   └── prod
│       ├── [...]
│       └── hosts           <- hosts file for "prod"
├── roles
│   └── my_role
└── site.yml
```

```bash
$ ansible-playbook -i inventory/staging/hosts site.yml
```

*Info*: 
- For groups of hosts
- [Directory layouts in best pratices guide](http://docs.ansible.com/ansible/playbooks_best_practices.html#directory-layout)

---
## The layer between
### (result of my experience)

```
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   ├── all.yml    <- Vars for all hosts
│   │   ├── aerith.yml
│   │   ├── cid.yml
│   │   ├── cloud.yml
│   │   └── prod.yml
│   ├── host_vars
│   │   └── aerith.ko.yml
│   ├── prod           <- inventory for "prod"
│   ├── dev            <- and for "dev"
│   └── test           <- you know what this is..
├── roles
│   └── my_role
└── site.yml
```

```bash
$ ansible-playbook -i inventory/dev site.yml
```

*Info*: 
- Combines seperate inventory files and grouped vars files
- May be unhandy, with all the group files in one directory. So, it does not scale to very large environments.

------
# The `ansible-playbook` command

---
## Configure ansible

- `-i INVENTORY`/`--inventory-file=INVENTORY`: specify the inventory file
- `-e EXTRA_VARS`/`--extra-vars=EXTRA_VARS`: allows you to specify/overwrite variables for the playbook run.

---
### `--extra-vars`/`-e` example
```yaml
# site.yml
- hosts: all
  tasks:
    - debug: vars=hostvars
      when: debug|default(false)
```

```bash
# this will print out hostvars 
ansible-playbook -e debug=true site.yml
# this not
ansible-playbook site.yml 
```

---
## Do not execute everything

- `-t TAGS`/`--tags=TAGS`: only run plays and tasks tagged with these values
- `--skip-tags=SKIP_TAGS`: skip the given tags
- `--start-at-task=START_AT_TASK`: starts at the task with the given name
- `-l SUBSET`/`--limit=SUBSET`: limit to a subset of hosts

---
## Set connection options

- `-k`/`--ask-pass`: ask for connection password
-  `-u REMOTE_USER`/`--user=REMOTE_USER`: connect as this user
- `--private-key=PRIVATE_KEY_FILE`/`--key-file=PRIVATE_KEY_FILE`: give the private key file
- `-K`/`--ask-become-pass`: ask for privilege escalation password. 

*Info*: 
- `-k` Option is very usefull for passwords, you only need once

---
## General useful options

- `--flush-cache`: clear the fact cache (KEEP THIS IN MIND!!)
- `--step`: run tasks step by step, each step must be confirmed
- `-C`/`--check`: makes a "dry-run" without changing anything.
- `--syntax-check`: linter

------
# Use "real" yaml syntax instead of inline style

---
```yaml
- name: ensure root privileges for user
  lineinfile:
    dest: /etc/sudoers
    state: present
    regexp: "^{{ user_name }} ALL"
    line: "{{ user_name }} ALL=(ALL) NOPASSWD:ALL"
    validate: "visudo -cf %s"
```

*Info*:
- Is more readable ;)
- Allows you to add/remove options by lines (git will thank you)
- Drawback: You have to add `"` around `{{ variables }}`
- Hint: Did you see the `validate`? ;)

------
# Getting started/Initialize a new host

---
- Manage your users with ansible.
- Describe the end-state of your system. 
- Pass inital related config (setup-user credentials) via commandline on first run

*Info*:
- Do not have a "init playbook"
- end-state: includes inventory file, etc.
- Have the script, that ensures you can run ansible as a part of the usual scripts.

---
Example:
```yaml
- name: ensure user
  user:
    name: myself
    generate_ssh_key: yes

- name: add authorized key of controll machine 
  authorized_key:
    user: myself
    key: "{{ lookup('file', '/home[...]id_rsa.pub') }}"
```

```bash
# first run
$ ansible-playbook -u setup_user -k site.yml
# all other runs
$ ansible-playbook site.yml
```

*Info*:
- `lookup` searches on the controll machine! 
- `-k` (lower "k" is shortcut for `ask-pass`, that will ask for connection password)

------
# `github_key` module
## A module to notice

---
```yaml
- name: ensure user and get information
  user:
    name: my_user
    generate_ssh_key: yes
  register: user_info

- name: place publickey at github
  github_key:
    name: "SSH Key for {{ ansible_nodename }} (added via ansible)"
    pubkey: "{{ user_info.ssh_public_key }}"
    token: "{{ github_access_token }}"
```

Create tokens at [github.com/settings/tokens](https://github.com/settings/tokens) (activate `write:public_key`)

*Info*:
- [`github_key` documentation](http://docs.ansible.com/ansible/github_key_module.html)

------
# Ansible Galaxy

---
```yaml
# requirements.yml
- galaxy.role_name
- https://github.com/role/from_github
  name: as_this_name
```

```ini
# ansible.cfg
[defaults]
roles_path = ./galaxy_roles:./roles
```

```bash
# install the roles
$ ansible-galaxy install \
  --role-file=requirements.yml \
  --roles-path=galaxy_roles

# does the same, but with shortcuts for options
$ ansible-galaxy install -r=requirements.yml -p=galaxy_roles
```

*Info*:
- Add galaxy role path to `.gitignore`.
- [galaxy cli docs](http://docs.ansible.com/ansible/galaxy.html#installing-multiple-roles-from-a-file)

---
## Create a new role with `ansible-galaxy`

```bash
$ ansible-galaxy init --init-path=roles/ --offline my_new_role
```

---
```
.
└── my_new_role
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

*Info*:
[`ansible-galaxy init` in the docs](http://docs.ansible.com/ansible/galaxy.html#create-roles)

------
# Variable naming schema

---

## This looks good, but...

```yaml
user:
  name: my_user
  home: /home/my_user

```
---

## ... this does the job better

```yaml
user_name: my_user
user_home: "/home/{{ user_name }}"
```

*Info*:
- Prefix your variable names with the role name.
- Do NOT use `hash_behavior=merge`, and "nested" variables, even if it looks sexy on the first time.
    + this allows you to reuse variables.

------
# Using variables from another role

---
```yaml
# roles/user/defaults/main.yml
user_home: /home/my_user
```

```yaml
# roles/zsh/defaults/main.yml
zsh_user_home: "{{ user_home }}"
```

*Info*:
- In your tasks, do not rely on variables from another role.
- "Import" the variable in you `defaults/main.yml`.

------
# `defaults` vs `vars`

---
## defaults

- Configuration, you expose to the user
- Some kind of "arguments", you can use for the role
- "public"
---

## vars

- Configuration inside the role
- Typically environment specific & loaded by facts
- "private"

---
## Vars example
```yaml
# roles/ssh/vars/Debian.yml
ssh_service_name: ssh
```

```yaml
# roles/ssh/vars/Archlinux.yml
ssh_service_name: sshd
```

```yaml
# roles/ssh/tasks/main.yml
- name: Include OS-Specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: enable ssh service
  service:
    name: "{{ ssh_service_name }}"
    enabled: yes
```

------
# `become: true`

*Info*:
There is a lot of hazzle about the `become` directive, most targeting its, in first case, uncommon naming.

And yes, it may look strange to say `become: true` to execute something with root privileges. 

---
## become another user?: Yes

---
## Configure `become`

- `become`: `true`/`false` (default: `false`), toggles `become`.

- `become_user`: Username to become (default: `root`).

- `become_method`: One of: `sudo`, `su`, `pbrun`, `pfexec`, `doas`, `dzdo`, `ksu` (default: `sudo`).

- `become_flags`: Additional flags, like `-s /bin/sh`.

------
# Places to configure `become`

---
## At task level

```yaml
- name: install some packages
  become: true
  become_user: root
  become_method: sudo
  package:
    name: aweseome
    state: latest
```

---
## At includes

```yaml
- include: install_packages.yml
  become: true
  become_user: root
  become_method: sudo
```

*Info*:
- Will set `become: true` for all tasks in the includes file.

---
## At Role includes in playbook

```yaml
- hosts: all
  roles:
    - role: myrole
      become: true
      become_user: root
      become_method: sudo
```

*Info*:
- Will set `become: true` for the entire role.
- But NOT for handlers. Seems to be a bug. Solution: Add `become: true` for handler tasks.

---
## At the command line

```bash
$ ansible-playbook --become \
    --become-user=root \
    --become-method=sudo  \
    --ask-become-pass \  # <-- Remember this option
    site.yml
```

*Info*:
You can shortcut the `--aks-become-pass` option with `-K` (upper "K")
With this, ansible will ask your for the password to become the other user (Which password may depend on the `become-method`.)
[`become` in the docs](http://docs.ansible.com/ansible/become.html)

------
# Role include parameters

```yaml
# playbook.yml
- hosts: all
  roles:
      # simple role include
    - myrole

      # set role defaults value
    - role: someotherrole
      myrole_var: foobar

      # and call the same role twice
    - role: someotherrole
      myrole_var: notfoo
```

*Info*:
- You can pass arguments, when including roles in your playbook
- Keep in mind, that you can design roles, that are included more than one time
- [Roles in the docs](http://docs.ansible.com/ansible/playbooks_roles.html#roles)

------
# Role dependencies

---

Define role dependencies in the `meta/main.yml` file of your role

---
```yaml
# roles/myrole/meta/main.yml
dependencies:
  # simple
  - somerole

  # with parameters
  - role: aur 
    pkg_name: thinkfan
```

*Info*:
- Same as role includes in playbook ;)

---
## Be careful with dependencies and role options

```yaml
# playbook.yml
- hosts: all
  roles:
    - role: myrole
      tags: [myrole]

    - role: myotherrole
      tags: [myotherrole]
```

```yaml
# roles/myotherrole/meta/main.yml
dependencies:
  - myrole
```

*Info*:
With this, `myrole` will be executed twice. First time from the playbook, with the tag, second time from the dependency from `myotherrole`, without the tag.
[Role dependencies in the docs](http://docs.ansible.com/ansible/playbooks_roles.html#role-dependencies)

------
# `include_role` module
## A module to notice

---
```yaml
- name: include a role at task level
  include_role:
    name: myrole
  vars:
    myrole_var: foobar
```

*Info*:
[`include_role` in the docs](http://docs.ansible.com/ansible/include_role_module.html)

---
## You don't need to use the `main.yml`

```yaml
- name: include a role at task level
  include_role:
    name: myrole
    tasks_from: deploy.yml
```

*Info*:
[`include_role` in the docs](http://docs.ansible.com/ansible/include_role_module.html)

------
# Loops

*Info*:
[Loops in the docs](http://docs.ansible.com/ansible/playbooks_loops.html)

---
## `when` in loops

```yaml
- name: install some packages
  package:
    name: "{{ item.name }}"
    state: "{{ item.state }}"
  with_items:
    - { name: sudo,    state: present }
    - { name: ansible, state: latest }
    - { name: apache2, state: absent }
  when: item.state != "absent"
```

---
## Loops with `template` module

```yaml 
- name: write some files
  template:
    dest: "{{ item.filename }}"
    src: template.j2
  with_items:
    - { filename: foo, content: foobaz }
    - { filename: bar, content: barbaz }
```

```jinja
{# template.j2 #}
{{ item.content }}
```

*Info*: 
`item` is available in templates.

---
## Looping dictionaries in Jinja2

```jinja
{ % for key, item in my_dict.iteritems() %}
  {{ key }}{{ item }}
{ % endfor %}
```

------
# A word about `block`

Don't use them. Use `include`s instead.

*Info*:
- the idea of block is, to group tasks and apply some directives at a single point.
- Blocks can't be looped, what's sad, really sad.
- Blocks introduce strange, not intuitive conventions, intendention, etc.
- [Block in the docs](http://docs.ansible.com/ansible/playbooks_blocks.html)

------
# Delegating tasks

Use `delegate_to` to execute the task on another machine than the current.

---
```yaml
- name: add host to hostsfile
  lineinfile:
    dest: /etc/hosts
    line: "{{ ansible_default_ipv4.address }} {{ inventory_hostname }}"
  delegate_to: 127.0.0.1
```

*Info*:
[Delegation in the docs](http://docs.ansible.com/ansible/playbooks_delegation.html#delegation)

------
# `add_host` module
## A module to notice

---
```yaml
- name: ensure user and get information
  user:
    name: my_user
    generate_ssh_key: yes
  register: user_info

- name: add another hosts
  add_host:
    name: new_host.foo
    ansible_host: 192.168.42.31

- name: add authorized key to added host
  authorized_key:
    user: my_user
    key: "{{ user_info.ssh_public_key }}"
  delegate_to: new_host.foo
```

---
## `add_host` will survive `hosts`-sections in playbooks
```yaml
- hosts: localhost
  tasks:
    - ec2: instance_type=t2.small image=ami123456
      register: ec2

    - add_host:
        name: "{{ item.public_ip }}"
        group: my_ec2   # <- notice the groupname
      with_items: "{{ ec2.tagged_instances }}"

    - wait_for: host="{{ item.public_dns_name }}" port=22
      with_items: "{{ ec2.tagged_instances }}"

- hosts: my_ec2   # <- we use the groupname from above
  roles:
    - role: some_role
```

*Info*:
- [`add_host` in the docs](http://docs.ansible.com/ansible/add_host_module.html)

------
# `register`

---
```yaml
- name: get file infos
  stat:
    path: /opt/project/index.php
  register: file_info

- debug: var=file_info
```

```json
{
    "file_info": {
        "changed": false,
        "stat": {
            "atime": 1485083552.0,
            "attr_flags": "",
            "attributes": [],
            "birthtime": 1462644637.0,
            "block_size": 4096,
            "blocks": 8,
            "charset": "unknown",
            "checksum": "e739cd3bdf5395fd8bf7cd6b1b71b4b5dc1ffa58",
            "ctime": 1482073380.0,
            "dev": 16777217,
            "device_type": 0,
            "executable": false,
            "exists": true,
            "flags": 0,
            "generation": 0,
            "gid": 20,
            "gr_name": "staff",
            "inode": 6835479,
            "isblk": false,
            "ischr": false,
            "isdir": false,
            "isfifo": false,
            "isgid": false,
            "islnk": false,
            "isreg": true,
            "issock": false,
            "isuid": false,
            "md5": "6953afbf691e7a58f4bdb481f92d18fa",
            "mimetype": "unknown",
            "mode": "0644",
            "mtime": 1482073380.0,
            "nlink": 1,
            "path": "/Users/ko/.zshrc",
            "pw_name": "ko",
            "readable": true,
            "rgrp": true,
            "roth": true,
            "rusr": true,
            "size": 379,
            "uid": 501,
            "version": null,
            "wgrp": false,
            "woth": false,
            "writeable": true,
            "wusr": true,
            "xgrp": false,
            "xoth": false,
            "xusr": false
        }
    }
```


---
```yaml
- name: ensure user 
  user:
    name: myself
    generate_ssh_key: true
  register: user_info

- debug: var=user_info
```

```json
{
    "user_info": {
        "append": false,
        "changed": false,
        "comment": "",
        "group": 100,
        "home": "/home/myself",
        "move_home": false,
        "name": "myself",
        "shell": "/bin/zsh",
        "ssh_fingerprint": "2048 SHA256:G[...]A ansible-generated (RSA)",
        "ssh_key_file": "/home/myself/.ssh/id_rsa",
        "ssh_public_key": "ssh-rsa AA[...]bF ansible-generated",
        "state": "present",
        "uid": 1000
    }
}
```

*Info*:
- A LOT modules support the `register` directive. Just try it!
- Output may depend on what you've set up in the registering task
- [`register` in the docs](http://docs.ansible.com/ansible/playbooks_variables.html#registered-variables)

------
# `set_fact` module

```yaml
- name: set some facts
  set_fact:
    some_fact: foobar
    a_fact: bazbar

- debug: var=some_fact
- debug: var=a_fact
```

```json
{
    "some_fact": "foobar"
}
{
    "a_fact": "bazbar"
}
```

*Info*:
[`set_fact` module in the docs](http://docs.ansible.com/ansible/set_fact_module.html)

---
## Read input from JSON/YAML
```yaml
- shell: cat file.json
  register: result

- set_fact: myvar="{{ result.stdout | from_json }}"
```

```yaml
- shell: cat file.yaml
  register: result

- set_fact: myvar="{{ result.stdout | from_yaml }}"
```

*Info*: 
[`from_yaml` & `from_json` in the docs](http://docs.ansible.com/ansible/playbooks_filters.html#filters-for-formatting-data)

------
# `deploy_helper` module
## A module to notice

---
### We want this
```
.
├── current -> /opt/project/releases/20161114144418
├── releases
│   ├── 20161020130552
│   ├── 20161020134521
│   ├── 20161020134651
│   ├── [ ... ]
│   ├── 20161114143449
│   └── 20161114144418
└── shared
```

---

```yaml
- name: Initialize deployment helper
  deploy_helper:
    path: /opt/project
    state: present
```

Will produce this:

```
/opt/project
├── releases
└── shared
```

```json
{
    "deploy_helper": {
        "current_path": "/opt/project/current",
        "new_release": "20170121141232",
        "new_release_path": "/opt/project/releases/20170121141232",
        "previous_release": null,
        "previous_release_path": null,
        "project_path": "/opt/project",
        "releases_path": "/opt/project/releases",
        "shared_path": "/opt/project/shared",
        "unfinished_filename": "DEPLOY_UNFINISHED"
    }
}
```

*Info*:
- This call of the deploy helper module will create the project directory, it's shared and the release directory.
- Also, we get a lot of variables provides in the `deploy_helper` variable.

---
## Do the deployment

```yaml
- name: create new relase path
  file:
    path: "{{ deploy_helper.new_release_path }}"
    state: directory

- name: extract tarball to new release directory
  unarchive:
    src: my_software.tar.gz
    dest: "{{ deploy_helper.new_release_path }}"

- name: Add unfinished file
  file:
    path: "{{ deploy_helper.new_release_path }}/{{ deploy_helper.unfinished_filename }}"
    state: touch
```

*Info*: 
- As you see, no `deploy_helper` module call, only the variables provided.
- DO not forget, to add the `unfinished_file`,so ansible can detect succesful deployments.

---
### Result of previous slide

```
/opt/project
├── current
├── releases
│   └── 20161114144418
│       ├── DEPLOY_UNFINISHED
│       └── index.php
└── shared
```

---
```yaml
- name: Finalize the deployment
  deploy_helper: 
    path: /opt/project
    release: "{{ deploy_helper.new_release }}"
    state: finalize
```

will result in this:

```
/opt/project
├── current -> /opt/project/releases/20161114144418
├── releases
│   └── 20161114144418
│       └── index.php
└── shared
```

*Info*:
Note, that the `UNFINISHEDFILE` is absent!
[`deploy_helper` module in the docs](http://docs.ansible.com/ansible/deploy_helper_module.html)


------
# `run_once` directive

```yaml
- name: update database schema
  make:
    chdir: /opt/project
    target: database-migrate
  run_once: true
```

*Info*: 
- `run_once`, obviously run every playbook run ;)
- [`run_once` in the docs](http://docs.ansible.com/ansible/playbooks_delegation.html#run-once)

------
# Tip about services and handlers

- In tasks: Configure service, enable them, etc., but do not restart them!
- In Handlers: just only restart services.

*Info*:
- This makes your configuration more flexible
- This delegates responsibilites to where there belong

---
```yaml
# role/tasks/main.yml
- name: enable some service
  service:
    name: someservice
    enabled: true
  notify: restart someservice

# role/handlers/main.yml
- name: restart someservice
  service: someservice
    state: restarted
```

*Info*: 
- Do not forget the `notify`
- [handlers in the docs](http://docs.ansible.com/ansible/playbooks_intro.html#handlers-running-operations-on-change)
