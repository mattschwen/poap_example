---
layout: post
title: A Practical Implementation of Cisco POAP
subtitle: Power-On Auto Provisioning
bigimg: /img/highlands-in-winter.jpg
tags: [cisco, poap, python, ansible, nexus, nxos, dhcp, power-on auto provisioning, aci]
gh-repo: mattschwen/poap_example
gh-badge: [star, fork, follow]

# gh-repo: daattali/beautiful-jekyll
# gh-badge: [star, fork, follow]
# image: /img/hello_world.jpeg
# layout: post
# title: To be
# subtitle: ... or not to be?
# tags: [books, shakespeare, test]
# bigimg: /img/path.jpg
---
# Purpose

The purpose of this post is to show a real world implementation of `POAP` using 100% `Opensource` tools like `Linux, Python and Ansible`. It would be helpful to have a basic understand of `Ansible` and `Python` but is not required. I have provided a `GIT` repository that will contain all the necessary `Ansible` files.

[GitHub Page](https://github.com/mattschwen/poap_example)

# Understanding POAP

## Cisco Documentation
- [Official POAP Documention from Cisco](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/fundamentals/configuration/guide/b_Cisco_Nexus_9000_Series_NX-OS_Fundamentals_Configuration_Guide_7x/b_Cisco_Nexus_9000_Series_NX-OS_Fundamentals_Configuration_Guide_7x_chapter_0100.html)
- [POAP Script on Github](https://github.com/datacenter/nexus9000/blob/master/nx-os/poap/poap.py)

> Do not go out and buy [DCNM](https://www.cisco.com/c/en/us/products/cloud-systems-management/prime-data-center-network-manager/index.html) just for the **POAP** and **templating** features. `Ansible` can do it all for free, but it will require some steps for the initial setup.

It is important to first understand how `POAP` actually works. I will not get to in depth in this section since there is alreay a ton of [documentation](https://developer.cisco.com/docs/nx-os/#!poap/poap-poweron-auto-provisioning) out there to explain this.

## Requirements

    [x] DHCP Server
    [x] TFTP Server # can be same server as DHCP
    [x] SCP Server # can be same server as DHCP
    [x] POAP Script # link included above
    [x] NXOS Code
    [x] Nexus Switch

## Process

1. Boot a new `Nexus` switch or `write erase reload` an existing `Nexus` switch.
2. If no configuration exists, the `POAP` process (built-in Python scripts) will run as soon as the device loads.
3. `DHCP` discovery is done on the management port of the switch.
4. `DHCP` server options tells the switch its `IP` and `TFTP` server so it can download the `POAP` script. (identified by switch serial #)
5. The switch will now download the `POAP` script and execute it.
6. If the bootflash does not contain the code listed in the `POAP` script it will download it from the location listed in the `POAP` script.
7. The `POAP` script then downloads its configuration file.
8. The switch loads code (if necessary) and reboots.
9. The switch loads the configuration file and saves it to `NVRAM`.

## POAP Scirpt

This [script](https://github.com/datacenter/nexus9000/blob/master/nx-os/poap/poap.py) is huge, but you only need to pay attention to a few lines of code. We will convert the `POAP.py` script into a `Jinja2` template to be used for `Ansible` by renaming `POAP.py` to `POAP.py.j2`.

Below are the only options that need to be changed for the poap script. The `{% raw %}"{{  }}"{% endraw %}` brackets indicate that `Jinja2` will fill in those spaces with variables from your `Ansible` files (covered in the `Ansible` section).

> The `mode` option indicates how the `DHCP` server will identify the switch. I tested this with `MAC addresses`, but since `Serial numbers` are listed on the box it was much easier to use the `serial_number` option.

```python
"""
Increasing the script timeout to handle the legacy nxos
versions where upgrading will take time
"""

# script_timeout=1800
# --- Start of user editable settings ---
# Host name and user credentials
options = {
options = {
   "username": "UBUNTU_SERVER_USERNAME",
   "password": "UBUNTU_SERVER_PASSWORD",
   "hostname": {% raw %}"{{ scp_server }}"{% endraw %},
   "transfer_protocol": "scp",
   "mode": "serial_number",
   "target_system_image": {% raw %}"{{ image }}"{% endraw %},
}
```
<hr>
# Environment

## Server

    [x] Ubuntu 16.04.2 LTS
    [x] isc-dhcp-server
    [x] default tftp service

> Ensure the `Ubuntu` server has access to the `Management Interface` network of the switch and vice-versa.

I found it much easier to work with an `Opensource DHCP server`, for this example I will use [isc-dhcp-server](https://www.isc.org/downloads/dhcp/).

> Of courese, do the basic `Ubuntu` stuff like `apt update`, apt install [all packages required for python, isc-dchp-server, tftp], `disable` any `firewall` or enable ports for dhcp, tftp, ssh, etc.

## Ansible

> I am using `Ansible` v2.3 on my `Mac` and it works fine.

If you know nothing about `Ansible`, you may want to check out so basic `Ansible` training. However, this setup is pretty straight forward.

### Ansible Structure

> I feel it is important to understand the structure of `Ansible` prior to running a `playbook`. If you don't know what a `playbook` is, take some time to do some basic `Ansible` training. It is not hard and `Ansible` can change your life, do it!

Create the following structure (hopefully in a `GIT` repository of your own) but you can clone [poap_example](https://github.com/mattschwen/poap_example). Each document will be explained below.

```
ansible
├── environments
│   └── servers
│       ├── group_vars
│       │   └── poap.yml
│       └── inventory
├── poap-playbook.yml
└── roles
    └── poap
        ├── files
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── createmd5.py.j2
            ├── dhcp.conf.j2
            ├── switch.cfg.j2
            ├── poap.py.j2
            └── tftp.j2
```
<br>
### Ansible Files

**poap-playbook.yml**

```yaml
---
- name: setup poap server
  hosts: poap
  become: true
  roles:
    - poap
```

**environments/servers/group_vars/poap.yml**

> The `-` in the `YAML` file indicates a list, you can add as `items to the list` (switches) as you want by specifying another `-` with all other switch variables.

```yaml
---
dhcp_reservations:
  - name: new_switch_1 # hostname of the new switch
    sn: FOC232323232323 # Serial number of the switch
    fixed_address: '2.2.2.2' # Address you are giving the switch
    subnet: {% raw %}'{{ management_netmask }}'{% endraw %} # Subnet of switch fixed address
    default_gateway: {% raw %}'{{ management_gateway }}'{% endraw %} # default gateway
    tftp_server_address: {% raw %}'{{ tftp_server }}'{% endraw %} # scp, tftp, and dhcp are all the same server
    boot_filename: {% raw %}'{{ image }}'{% endraw %} # Image name specified below
  - name: new_switch_2
    sn: FOC242424242424
    fixed_address: '2.2.2.3'
    subnet: {% raw %}'{{ management_netmask }}'{% endraw %}
    default_gateway: {% raw %}'{{ management_gateway }}'{% endraw %}
    tftp_server_address: {% raw %}'{{ tftp_server }}'{% endraw %}
    boot_filename: {% raw %}'{{ image }}'{% endraw %}

# POAP
image: nxos.7.0.3.I4.8a.bin
tftp_server: 1.1.1.1
scp_server: 1.1.1.1
script_name: poap.py

# Management Network
management_network: 2.2.2.0
management_netmask: 255.255.255.0
management_broadcast: 2.2.2.255
management_gateway: 2.2.2.1
```

**environments/servers/inventory**

> You will want to familarize yourself with how `Ansible` manages `inventory` [here](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html). This can be confusing, in this case `POAP` upgrades the switch, not `Ansible`. All `Ansible` does is sets up the `environment` for `POAP`, so your only inventory item will be the `server` that is used for that `environment`.

```yaml
[poap]
1.1.1.1 # This should match the scp_server variable in poap.yml file
```
**roles/poap/handlers/main.yml**

> `Handlers` in `Ansible` specify custom ways of managing services. You will see the specific name of the `service` and its desired `state`. When you envoke this handler from the `playbook` task list in `tasks/main.yaml` it will `restart` the service as indicated by the `state`.

```yaml
---
- name: restart isc-dhcp-server.service
  service:
    name=isc-dhcp-server.service
    state=restarted

- name: restart xinetd
  service:
    name=xinetd.service
    state=restarted
```

**roles/poap/tasks/main.yml**

> `Tasks` in `Ansible` describes exactly what the `playbook` is doing. `Debug` options can `enabled` or `disabled` for `troubleshooting`. I have created a user called `poap` in the `Ubuntu` server. If you are using a different user, you can change the `owner` and `group` to whatever you like.

```yaml
---
#- debug:
#    msg: "VAR {{ dhcp_reservations }}"

#- debug:
#    msg: "VAR {{ item.name }}"
#  with_items: "{{ dhcp_reservations }}"

- name: setup isc-dhcp-server dhcp
  template:
    src=dhcp.conf.j2
    dest=/etc/dhcp/dhcpd.conf
    owner=root
    group=root
    mode=0644
  notify: restart isc-dhcp-server.service

- name: start isc-dhcp-server.service
  service:
    name=isc-dhcp-server.service
    state=started
    enabled=yes

- name: setup tftp server
  template:
    src=tftp.j2
    dest=/etc/xinetd.d/tftp
    owner=root
    group=root
    mode=0644
  notify: restart xinetd

- name: start xinetd
  service:
    name=xinetd
    state=started
    enabled=yes

- name: Create Directory Structure
  file:
    path=/var/lib/tftpboot/
    state=directory
    owner=nobody
    mode=0777

- name: copy poap.py script to server
  template:
    src=poap.py.j2
    dest=/var/lib/tftpboot/poap.py
    owner=poap
    group=poap
    mode=0755

# The switch configuration is named by the switche's serial number
- name: setup configuration files for switch
  template:
    src=switch.cfg.j2
    {% raw %}dest=/var/lib/tftpboot/conf.{{ item.sn }}{% endraw %}
    owner=poap
    group=poap
    mode=0755
  {% raw %}with_items: "{{ dhcp_reservations }}"{% endraw %} # This will cycle through each item in the dhcp_reservations list.

# This script is done in python 2.7
- name: execute createmd5.py python script
  command: '/var/lib/tftpboot/createmd5.py'
```

**roles/poap/templates/createmd5.py.j2**

> This script was written by me years ago and it probably sucks and can be re-done. Go for it!

This `Python` script will be copied to the server and run. It will take all files in `/var/lib/tftpboot/` except for `createmd5.py` and `poap.py` and make an `MD5` file. It is necessary for all files being used by `Cisco POAP` to have an `MD5` equivelant to ensure the switch downloads a correct copy of the file during the `POAP` process by calculating the `MD5 cheksum`. Cisco specifies how to do it in thier documentation [Here](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/fundamentals/configuration/guide/b_Cisco_Nexus_9000_Series_NX-OS_Fundamentals_Configuration_Guide_7x/b_Cisco_Nexus_9000_Series_NX-OS_Fundamentals_Configuration_Guide_7x_chapter_0100.html#id_53294)

What I cannot find in `Cisco documentation` is that `switch` configurations also need to have a cooresponding `MD5` which I learned the hard way; the script does this automatically.

```python
#!/usr/bin/python

import sys
import os


def main(regex):
    """
    Main function, calls on other functions, start of the script.
    """
    filesL = []

    cmd1 = 'f=/var/lib/tftpboot/poap.py ; cat $f | sed '
    cmd2 = '"/^#md5sum/d" '
    cmd3 = '> $f.md5 ; sed -i '
    cmd4 = '"s/^#md5sum=.*/#md5sum=\"'
    cmd5 = '$(md5sum $f.md5 | sed '
    cmd6 = '"s/ .*//"'
    cmd7 = ')\"/" $f'

    poapcmd = '%s%s%s%s%s%s%s' % (
        cmd1, cmd2, cmd3, cmd4, cmd5, cmd6, cmd7
    )

    directory = os.path.dirname(os.path.realpath(__file__)) + '/'

    for filename in os.listdir(directory):
        if filename.endswith(('.md5')):
            dfile = directory + filename
            os.remove(dfile)
        else:
            pass

    for filename in os.listdir(directory):
        if (filename == 'createmd5.py') or (filename == 'poap.py'):
            pass
        else:
            filesL.append(filename)

    for file in filesL:
        command = 'md5sum %s > %s.md5' % (directory + file, directory + file)
        os.system(command)
        print 'Created MD5 For: %s' % (file)

    for filename in os.listdir(directory):
        if 'poap.py' == filename:
            os.system(poapcmd)
            print 'Created MD5 For: %s' % (filename)
        else:
            pass

    for filename in os.listdir(directory):
        os.chmod(directory + filename, 0o777)

if __name__ == '__main__':
    main(main)
```

**roles/poap/templates/dhcp.conf.j2**

You will need to update the following sections to match your specific envirnment.

    [x] INTERFACES
    [x] option domain-name "mydomain.com";
    [x] option domain-name-servers 1.1.1.1;

In your host specific arguments you will see `option dhcp-client-identifier "\000{{ item.sn }}";`, this will allow the `DHCP` server to specify options for each `switch` and identify each `switch` by its `serial number`. The requirement for this is `\000 + serial number`.

```
#
# DHCP Server Configuration File
#

INTERFACES="ens3";

# Configure the default and maximum lease times
max-lease-time 3600;
default-lease-time 3600;

# Disable dynamic DNS updates
ddns-update-style none;

# Make the server authoritative for the network segments that
# are configured, and tell it to send DHCPNAKs to bogus requests
authoritative;

# Allow each client to have exactly one lease, and expire
# old leases if a new DHCPDISCOVER occurs
one-lease-per-client true;

# Tell the server to look up the host name in DNS
# get-lease-hostnames true;

# Domain name to distribute to clients
option domain-name "mydomain.com";

# List of name servers to distribute to clients
option domain-name-servers 1.1.1.1;

# Log to the local0 facility by default
log-facility local0;

# Ping the IP address that is being offered to make sure it isn't
# configured on another node. This has some potential repercussions
# for clients that don't like delays.
# ping-check true;
{% raw %}
subnet {{ management_network }} netmask {{ management_netmask }} {
    option subnet-mask {{ management_netmask }};
    option broadcast-address {{ management_broadcast }};
    option routers {{ management_gateway }};
}

# Define host specific arguments
{% for item in dhcp_reservations %}
host {{item.name}} {
    option dhcp-client-identifier "\000{{ item.sn }}";
    fixed-address {{ item.fixed_address }};
    option routers {{ item.default_gateway }};
    option host-name "{{item.name}}";
    option bootfile-name "{{ script_name }}";
    option tftp-server-name "{{ item.tftp_server_address }}";
}
{% endfor %}
{% endraw %}
```

**roles/poap/templates/switch.cfg.j2**

I have provided a very basic `template`; however, you will want to update this `template` to match a basic `template` for your specific `switch`. If you want your `switch` to be reachable after `POAP` finishes you need to configure your `Management VRF interface` with `correct IP` information as laid out in the `group_vars/poap.yaml` file.

```
{% raw %}
!
! CONFIGURATION FILE FOR {{ item.name }}
!
! VARIABLES
!
hostname {{ item.name }}
!
crypto key param rsa label {{ item.name }} modulus 2048
!
vrf context management
  ip route 0.0.0.0/0 {{ item.default_gateway }}
!
interface mgmt0
  description management
  ip address {{ item.fixed_address }} {{ item.subnet }}
  no ip redirects
  no ipv6 redirects
  vrf member management

! INCLUDE THE REST OF YOUR CONFIGURATION HERE
{% endraw %}
```

**roles/poap/templates/tftp.j2**

> Simple configuration for `TFTP service`, makes the TFTP root `/var/lib/tftpboot`.

```
service tftp
{
protocol        = udp
port            = 69
socket_type     = dgram
wait            = yes
user            = nobody
server          = /usr/sbin/in.tftpd
server_args     = /var/lib/tftpboot
disable         = no
}
```

Run your `Ansible playbook`, debug issues (there will likely be some initially), login to your first `switch` via `console` to watch the `POAP` process, and enjoy `POAP`.

# Conclusion

Once you have your server environment setup, a `Nexus` switch with its `management interface` that can communicate with the `server`, and the `Ansible` environment setup correcly, you can easily install new `Nexus switches` into your `data center` environment using `POAP` for zero touch provisioning on day 1.
