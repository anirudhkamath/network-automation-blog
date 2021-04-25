This post describes an easy way to build and automate network compliance reporting using Ansible.

# Compliance in the network

As a network automation engineer, it is vital to understand the role of compliance standards in terms of network design. For automation to have its best effects, the network must have standards in place throughout the network for multiple parts of it, such as:

- Device host naming
- Device configuration block layouts
- Site design layouts
- Cabling coventions

to name a few.

When the size of the network scales, it becomes extremely difficult for network engineers to perform reliable compliance checks when required. Hence, this is an apt job for a network engineer to automate - as **the one thing code does better than humans is stupidly well defined steps at large scale**.

# Defining standardization parameters in code

To define a network standard to code is an art in itself. There are multiple ways to do this (obviously), but when boiled down to the basics, the code needs to know a couple of things:

- What data is standardized?
- How is that data standardized?

For instance, take an example standard for DNS that is supposed to be in effect across the network:

> All network devices must have the DNS servers 192.168.100.1 and 192.168.100.2 configured.

On a Cisco router (for instance), this means that the configuration needs to have these 2 lines present:

```
ip name-server 192.168.100.1
ip name-server 192.168.100.2
```

So in this case, the answers to the questions posed above are:

- What data is standardized? **The DNS servers 192.168.100.1 and 192.168.100.2**.
- How is it standardized? **For Cisco IOS based devices, the lines `ip name-server 192.168.100.1` and `ip name-server 192.168.100.2` must be present in the configuration. For other platforms, the lines will differ, but you catch my drift.

## How could this be conveyed in Ansible?

As this post is concerned about have compliance reporting done using Ansible, let us implement a way to check DNS configuration compliance (example shown above) in a plug-n-play manner using Ansible. To know how to use the basics of Ansible, please refer to [my post on Ansible](ansible-hostname-compliance.md).

First off, we need to tell Ansible about the standard DNS server IP addresses. As this is a standard across all devices in the network, the scope of devices that should have these servers is **all** devices. In your project folder's `group_vars` folder, under the `all` group declare a variable named `dns_servers`. This variable is a list of DNS servers that must be configured on all devices.

In a file called `all.yml` in the `group_vars` folder declare the variable:

```yaml
---
dns_servers:
  - 192.168.100.1
  - 192.168.100.2
```

# The compliance engine

Compliance checks should not be one-off job that just runs once and waits for the next "scheduled" run to get a report. In reality, automation should almost always involve compliance checks when performing actual changes on devices. An ideal way to make automated changes can be something like this:

1. Check compliance with regard to section of configuration in scope of change
    - Fail in rare edge cases so it can be reviewed separately
2. If not compliant, start actual changes to get desired end state.

So, it seems like this compliance check could be a set of tasks that need to be used repeatedly in multiple workflows. This is where [Ansible roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) come in handy! Roles allow you to declare and load re-usabale tasks, variables, and much more in a dynamic plug-n-play manner.

In the root of your project file structure, declare a folder named `roles`. Within the `roles` folder, declare a `dns-compliance` folder- this sub folder will hold tasks, vars, templates, libraries, etc. which are required for the role to work effectively in any playbook that may require use of the role.

```
# root of project
roles/
    dns-compliance/
        tasks/
        templates/
```

In this role, we require a `tasks/` and `templates/` sub-folder. The `tasks` folder holds the tasks that the compliance role will run when it is called, and the `templates` folder holds template files written in a format such as Jinja2, TextFSM, etc.

Using a [TextFSM parsing](textfsm.md) template, we can capture what servers are configured on device if any. In the `templates` folder, create a template file named `dns_ios.textfsm`. The content of this template could be:

```
Value List DNS_SERVERS ((\d{1,3}\.){3}\d{1,3})

Start
  ^\s*ip\s+name-server\s+${DNS_SERVERS}\s*$$
```

As for the tasks, in the `tasks` folder, create a file named `dns_compliance_ios.yml`. An example set of tasks to check DNS compliance could do the following things:

1. Collect running configuration
2. Collect presently configured DNS servers from running config
3. Check if it is compliant

This can be done as such:

```yaml
---

- name: "COLLECT RUNNING CONFIG"
  ios_command:
    commands: show running-config
  register: show_run

- name: "GET LIST OF DNS SERVERS CONFIGURED"
  set_fact:
    dns_servers_configured: "{ { show_run.stdout[0] | parse_cli_textfsm(role_path ~ '/templates/dns_ios.textfsm') } }"
  when:
    - show_run is defined

- name: "CHECK IF DNS IS COMPLIANT"
  set_fact:
    dns_compliant: "true"
  when:
    - dns_servers_configured == dns_servers

- name: "CHECK IF DNS IS COMPLIANT"
  set_fact:
    dns_compliant: "false"
  when:
    - dns_servers_configured != dns_servers
```

> Please remove the spaces in between the curly brace characters, GitHub pages does not display it properly. Follow [Jinja2 variable syntax](https://ttl255.com/jinja2-tutorial-part-1-introduction-and-variable-substitution/)

Here, `role_path` is an Ansible defined variable that can be used in a role to provide the correct path to the root of the roles folder.

> **NOTE:** The above tasks file works specifically with Ansible version 2.9. **For Ansible 2.10 or 3**, please refer to [this link](https://docs.ansible.com/ansible/latest/network/user_guide/cli_parsing.html#parsing-with-textfsm) for parsing TextFSM.

# Using the compliance check in a playbook

In the root of your folder, create your playbook `test_dns_compliance.yml`. Here is where the DNS compliance role is called to test DNS configuration compliance for the devices the playbook runs against. The playbook would look like this:

```yaml
---

- name: "TESTING DNS COMPLIANCE FOR IOS"
  hosts: ios # run against hosts in ios group defined in groupvars
  become: yes # go to enable mode
  gather_facts: no

  tasks:
    - include_role:
        name: "dns-compliance"
        tasks_from: "dns_compliance_ios.yml"

    - debug:
        var: dns_compliant # print if compliant or not
```
