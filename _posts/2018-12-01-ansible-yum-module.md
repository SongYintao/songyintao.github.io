---
title: Ansible 
subtitle: Ansible yum 模块学习
layout: post
tag: [ansible,yum]
---

ansible使用过程记录。

```shell
Usage: ansible <host-pattern> [options]

Define and run a single task 'playbook' against a set of hosts

Options:
  -a MODULE_ARGS, --args=MODULE_ARGS
                        module arguments
  --ask-vault-pass      ask for vault password
  -B SECONDS, --background=SECONDS
                        run asynchronously, failing after X seconds
                        (default=N/A)
  -C, --check           don't make any changes; instead, try to predict some
                        of the changes that may occur
  -D, --diff            when changing (small) files and templates, show the
                        differences in those files; works great with --check
  -e EXTRA_VARS, --extra-vars=EXTRA_VARS
                        set additional variables as key=value or YAML/JSON, if
                        filename prepend with @
  -f FORKS, --forks=FORKS
                        specify number of parallel processes to use
                        (default=5)
  -h, --help            show this help message and exit
  -i INVENTORY, --inventory=INVENTORY, --inventory-file=INVENTORY
                        specify inventory host path or comma separated host
                        list. --inventory-file is deprecated
  -l SUBSET, --limit=SUBSET
                        further limit selected hosts to an additional pattern
  --list-hosts          outputs a list of matching hosts; does not execute
                        anything else
  -m MODULE_NAME, --module-name=MODULE_NAME
                        module name to execute (default=command)
  -M MODULE_PATH, --module-path=MODULE_PATH
                        prepend colon-separated path(s) to module library
                        (default=['/Users/nali/.ansible/plugins/modules',
                        '/usr/share/ansible/plugins/modules'])
  -o, --one-line        condense output
  --playbook-dir=BASEDIR
                        Since this tool does not use playbooks, use this as a
                        subsitute playbook directory.This sets the relative
                        path for many features including roles/ group_vars/
                        etc.
  -P POLL_INTERVAL, --poll=POLL_INTERVAL
                        set the poll interval if using -B (default=15)
  --syntax-check        perform a syntax check on the playbook, but do not
                        execute it
  -t TREE, --tree=TREE  log output to this directory
  --vault-id=VAULT_IDS  the vault identity to use
  --vault-password-file=VAULT_PASSWORD_FILES
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number and exit

  Connection Options:
    control as whom and how to connect to hosts

    -k, --ask-pass      ask for connection password
    --private-key=PRIVATE_KEY_FILE, --key-file=PRIVATE_KEY_FILE
                        use this file to authenticate the connection
    -u REMOTE_USER, --user=REMOTE_USER
                        connect as this user (default=None)
    -c CONNECTION, --connection=CONNECTION
                        connection type to use (default=smart)
    -T TIMEOUT, --timeout=TIMEOUT
                        override the connection timeout in seconds
                        (default=10)
    --ssh-common-args=SSH_COMMON_ARGS
                        specify common arguments to pass to sftp/scp/ssh (e.g.
                        ProxyCommand)
    --sftp-extra-args=SFTP_EXTRA_ARGS
                        specify extra arguments to pass to sftp only (e.g. -f,
                        -l)
    --scp-extra-args=SCP_EXTRA_ARGS
                        specify extra arguments to pass to scp only (e.g. -l)
    --ssh-extra-args=SSH_EXTRA_ARGS
                        specify extra arguments to pass to ssh only (e.g. -R)

  Privilege Escalation Options:
    control how and which user you become as on target hosts

    -s, --sudo          run operations with sudo (nopasswd) (deprecated, use
                        become)
    -U SUDO_USER, --sudo-user=SUDO_USER
                        desired sudo user (default=root) (deprecated, use
                        become)
    -S, --su            run operations with su (deprecated, use become)
    -R SU_USER, --su-user=SU_USER
                        run operations with su as this user (default=None)
                        (deprecated, use become)
    -b, --become        run operations with become (does not imply password
                        prompting)
    --become-method=BECOME_METHOD
                        privilege escalation method to use (default=sudo),
                        valid choices: [ sudo | su | pbrun | pfexec | doas |
                        dzdo | ksu | runas | pmrun | enable | machinectl ]
    --become-user=BECOME_USER
                        run operations as this user (default=root)
    --ask-sudo-pass     ask for sudo password (deprecated, use become)
    --ask-su-pass       ask for su password (deprecated, use become)
    -K, --ask-become-pass
                        ask for privilege escalation password

Some modules do not make sense in Ad-Hoc (include, meta, etc)
```



#### 1. yum module

​	`yum module`可以提供的`status`： `latest `，`present`,`installed`,`removed`, `absent`，前3个代表安装，后面2个是卸载。

例：在指定节点上安装tree服务

```
[root@master ~]#ansible all -m yum -a "state=present name=tree"
```

例：在指定节点上安装httpd服务

```
[root@master ~]#ansible all -m yum -a "state=present name=httpd"
[root@client01 tmp]# rpm -qa httpd
httpd-2.2.15-54.el6.centos.x86_64
```

**例：在指定节点上安装tree服务**
检查服务是不存在的

```shell
[root@hdp-m ~]# ansible dr_exp1 -m shell -a "rpm -qa tree"
 [WARNING]: Consider using yum, dnf or zypper module rather than running rpm
192.168.1.142 | SUCCESS | rc=0 >>
127.0.0.1 | SUCCESS | rc=0 >>
192.168.1.137 | SUCCESS | rc=0 >>
```

执行命令批量安装：
ansible dr_exp1 -m yum -a "state=present name=tree"

执行命令批量删除

```shell
[root@hdp-m ~]# ansible dr_exp1 -m yum -a "state=removed name=tree"
[root@hdp-m ~]# ansible dr_exp1 -m yum -a "state=absent name=tree"
```



#### 2. rpm usage



```shell
rpm
RPM version 4.11.3
Copyright (C) 1998-2002 - Red Hat, Inc.
This program may be freely redistributed under the terms of the GNU GPL

Usage: rpm [-aKfgpqVcdLilsiv?] [-a|--all] [-f|--file] [-g|--group] [-p|--package] [--pkgid] [--hdrid] [--triggeredby]
        [--whatrequires] [--whatprovides] [--nomanifest] [-c|--configfiles] [-d|--docfiles] [-L|--licensefiles] [--dump]
        [-l|--list] [--queryformat=QUERYFORMAT] [-s|--state] [--nofiledigest] [--nofiles] [--nodeps] [--noscript] [--allfiles]
        [--allmatches] [--badreloc] [-e|--erase <package>+] [--excludedocs] [--excludepath=<path>] [--force]
        [-F|--freshen <packagefile>+] [-h|--hash] [--ignorearch] [--ignoreos] [--ignoresize] [-i|--install] [--justdb] [--nodeps]
        [--nofiledigest] [--nocontexts] [--noorder] [--noscripts] [--notriggers] [--nocollections] [--oldpackage] [--percent]
        [--prefix=<dir>] [--relocate=<old>=<new>] [--replacefiles] [--replacepkgs] [--test] [-U|--upgrade <packagefile>+]
        [--reinstall=<packagefile>+] [-D|--define 'MACRO EXPR'] [--undefine=MACRO] [-E|--eval 'EXPR'] [--macros=<FILE:...>]
        [--noplugins] [--nodigest] [--nosignature] [--rcfile=<FILE:...>] [-r|--root ROOT] [--dbpath=DIRECTORY] [--querytags]
        [--showrc] [--quiet] [-v|--verbose] [--version] [-?|--help] [--usage] [--scripts] [--setperms] [--setugids]
        [--conflicts] [--obsoletes] [--provides] [--requires] [--info] [--changelog] [--xml] [--triggers] [--last] [--dupes]
        [--filesbypkg] [--fileclass] [--filecolor] [--fscontext] [--fileprovide] [--filerequire] [--filecaps]
[root@docker1 ~
```



