# Ansible Workstation Init Playbook

This ... creates the default linux desktop environment. Execute this after the installation of popos (or debian/ubuntu) and all the necessary tools will be installed like vscode, github, ... All Configuration will be done.
Sync with seafile client and github is prepared... So the workstation is ready to use...

## Prerequisites

- Basic installation of Popos with hard drive encryption.
- Install ansible with `apt install ansible`.

## How to use

Get the latest version of this repository from github and then execute this command:

```bash
ansible-playbook local.yml
```

By default this will install all basic tools and does the basic configuration. Some tasks are only executed, if you provide additional tags like this:

```bash
ansible-playbook local.yml --tags all
ansible-playbook local.yml --tags steam
...
```

### Available Tags

...

