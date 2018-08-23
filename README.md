Ansible Role Sudo/Sudoers
========

[![Build Status](https://travis-ci.org/Turgon37/ansible-sudoers.svg?branch=master)](https://travis-ci.org/Turgon37/ansible-sudoers)
[![License](https://img.shields.io/badge/license-MIT%20License-brightgreen.svg)](https://opensource.org/licenses/MIT)
[![Ansible Role](https://img.shields.io/badge/ansible%20role-Turgon37.sudoers-blue.svg)](https://galaxy.ansible.com/Turgon37/sudoers/)

## Description

:grey_exclamation: Before using this role, please know that all my Ansible roles are fully written and accustomed to my IT infrastructure. So, even if they are as generic as possible they will not necessarily fill your needs, I advice you to carrefully analyse what they do and evaluate their capability to be installed securely on your servers.

This roles configures sudo

## Requirements

Require Ansible >= 2.4

### Dependencies

## OS Family

This role is available for Debian and CentOS

## Features

At this day the role can be used to :

  * install sudo
  * configure defaults and command rules
  * provide a pseudo "type" to allow other role to include sudo rules
  * [local facts](#facts)

## Configuration

### Role

All variables which can be overridden are stored in [defaults/main.yml](defaults/main.yml) file as well as in table below. To see default values please refer to this file.

| Name                                    | Types/Values              | Description                                                                |
| ----------------------------------------|---------------------------| -------------------------------------------------------------------------- |
| `sudoers__sss`                          | Boolean                   | Install packages necessary to use sudo with sss backend                    |
| `sudoers__defaults_(global/group/host)` | List of dict/string       | Declare defaults settings for sudoers at role level                        |
| `sudoers__purge`                        | Boolean                   | If true, all sudo rules not handled directly by this role will be removed  |
| `sudoers__ansible_managed_key`          | String                    | The string use to identify what sudo rules are managed by ansible          |
| `sudoers__rules_(global/group/host)`    | Dict of rules (see below) | The sudoers rules to apply at role level                                   |

#### Sudo rule

This role provide a pseudo "type" that you can use from another role.
This allow this another role to declare a set of sudo rules out of this sudoers role's scope and after it application in the playbook.

To use it just declare a task like this

```
- name: Configure sudoers rule for ROLE
  include_role:
    name: sudoers
    tasks_from: types/sudo_rule
  vars:
    sudoers__sudo_rule: {}
```

The whole rule configuration items must be under the variable named `sudoers__sudo_rule`.

| Name                     | Types/Values                 | Description                                                                                                                                                     |
| -------------------------|------------------------------| --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`                   | String                       | The name of the rule file (must not contains any space)                                                                                                         |
| `state`                  | Enum absent/present          | The state of the rule to remove if needed                                                                                                                       |
| `defaults`               | List of defaults (see below) | The list of sudoers 'defaults' directives to apply in this rule. Note that theses will be effective for whole sudo runtime configuration, not only to this rule |
| `users`                  | List of string               | List of users for which this rule will apply                                                                                                                    |
| `hosts`                  | List of string               | Optional list of hosts on which this rule will apply, default to ALL                                                                                            |
| `commands`               | List of commands (see below) | List of command definitions                                                                                                                                     |
| `comment`                | String                       | Optional comment to include in file                                                                                                                             |

##### "defaults" directive

Because sudo "defaults" directive can take optional value, ansible support two form for each defaults directive under defaults key:

```
Defaults   always_set_home
Defaults   listpw = always
```

* a simple string
* a mapping

The string version is the simplest to use because the string will be just put after the sudo keyword Defaults.

The mapping one allow more fine setup. First, be conscious that sudo Defaults directive support host, user, command and runas filtering. But you can only choose one filter condition for a default directive. So this ansible role will used the previous keys order as priority order if you set multiple filtering keys.

If the directive require a value you must set the directive name a key of the mapping and it's value associated as the value of the key.  
Else if the directive require only it's name (like requiretty) you have (it's a limitation) to put the static world "defaults" as key, and the name of the directive as the value.

For example to apply "requiretty" on user1 user set the following vars :

```
sudoers__sudo_rule:
  name: rule1
  defaults:
    - defaults: requiretty
      user: user1
```

And to apply "listpw" one set :

```
sudoers__sudo_rule:
  name: rule1
  defaults:
    - listpw: always
      user: user1
```

To summarize all theses dict keys are available inside a defaults specification :

| Name               | Usage                                              |
| -------------------|----------------------------------------------------|
| `defaults: NAME`   | For unvalued directives                            |
| `NAME: VALUE`      | For valued directive                               |
| `host: HOST`       | To restrict Defaults effect on specified host      |
| `user: USER`       | To restrict Defaults effect on specified user      |
| `command: COMMAND` | To restrict Defaults effect on specified command   |
| `runas: RUNAS_USER`| To restrict Defaults effect on specified runas user|


##### "commands" directive

Each command under commands key allow the user to run a system command with or without restrictions

A command bloc allow the following keys

| Name            | Types/Values             | Usage                                                                     |
| ----------------|--------------------------|---------------------------------------------------------------------------|
| `commands:`     | String or list of string | Command pattern(s), see man 5 sudoers for syntax specification            |
| `run_as_user:`  | String or list of string | This/theses commands (line above) must only be run as this/theses user(s) |
| `run_as_group:` | String or list of string | This/theses commands (line above) must only be run as this/theses group(s)|
| `tags`          | String or list of string | Tag or list of tags to apply on this command                              |

For example to allow user "user1" to run ls any where as root user without typing password :

```
sudoers__sudo_rule:
  name: rule1
  commands:
   - commands: /bin/ls
     run_as_user: user1
     run_as_group: root
     tags: NOPASSWD
```

## Facts

By default the local fact are installed and expose the following variables :

* ```ansible_local.sudoers.version_full```
* ```ansible_local.sudoers.version_major```


## Example

### Playbook

Use it in a playbook as follows:

```yaml
- hosts: all
  roles:
    - turgon37.sudoers
```

### Inventory

  * Declare your defaults settings

```
sudoers__defaults_global:
  - always_set_home
  - insults
  - listpw: always
  - mailsub: "[PRODUCTION][%h][SUDO SECURITY]"
  - mailto: admin@example.com
  - mail_no_user
  - mail_no_perms
  - mail_no_host
  - mail_badpass
  - passprompt_override
  - pwfeedback
  - secure_path: /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin
  - '!visiblepw'
```

  * Usage with sssd

```
# Required for sudo to retrieve rules from LDAP
sudoers__sss: true
```

  * Declare sudo rules from another role


```
- name: Configure sudoers rule for ROLE
  include_role:
    name: sudoers
    tasks_from: types/sudo_rule
  vars:
    sudoers__sudo_rule:
      name: role__autogenerated_rule_10
      remove_using_regexp:
        - role__autogenerated_rule_0[0-9]+
      force_remove_using_regexp: true
      users: '{{ role__user }}'
      hosts: ALL
      comment: Autogenerated rule for role
      commands:
        - commands: /bin/ls
          run_as_user: '{{ role__another_user }}'
          run_as_group: root
          tags: NOPASSWD
        - commands: /bin/cat /home/[a-zA-Z]*/.ssh/config
          run_as_user: ALL
          run_as_group: root
      defaults:
        - defaults: '!requiretty'
          user: '{{ role__user }}'
      state: present
```