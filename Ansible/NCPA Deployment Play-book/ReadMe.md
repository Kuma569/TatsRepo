## Background

[Nagios Cross-Platform Agent](https://github.com/NagiosEnterprises/ncpa) is a simple, light-weight, Monitoring Agent software which can be used with [Nagios](https://www.nagios.org). For Azure Stack's IaaS VMs monitoring, NCPA can be deployed on VMs using an Ansible play-book. This is a simple NCPA deployment solution using Ansible, and this ReadMe describes how to do it.

## Prerequisites

- Ansible server
  In this ReadMe article, Ansible on Ubuntu 18 LTS is used.

- Nagios server
  In this Readme article, Nagios XI is used.

- Windows Remote Manager (WinRM) on Windows managed hosts(VMs).
  Ansible will use WinRM to connect Windows Servers.

- SSL certificate exchange between Ansible and Linux managed hosts(VMs).
  Ansible will use SSH connection for the Ansible.

- Bi-directional port 5693 (TCP) has to be opened from your VMs to Nagios XI server for sending monitored metrics and events.

## Solution summary

1. Copy **"template"** and **"vars"** folders and **"win-ncpainstall.yml"** file and placed them in the Ansible service account user directory on the Ansible server.
2. Set up configurations and variables according to your Ansible & Nagios environment.
3. Run play-book to install NCPA on a target host.

### 1.  Copy folders and yaml file over to the Ansible server

Place these folders and files under home directory in the Ansible.

Repo under the **NCPA Deployment Play-book**:
    - Template folder: templates
    - Variables folder: vars
    - Play-book: win-ncpainstall.yml

Copy them over to Ansible's directories:

    - Templates folder: /home/"Ansible service account name"/ncpa_win_autoresister/templates
    - Variables folder: /home/"Ansible service account name"/ncpa_win_autoresister/vars
    - Play-book file: /home/"Ansible service account name"/win-ncpainstall.yml

### 2. Set-up configurations and variables

#### Templates

For NCPA's [passive checks](https://www.nagios.org/ncpa/help/2.0/passive.html) . It configures what needs to be monitored, and when needs to be notified to Nagios server. The following configuration is for a standard Windows server.

```t

#
# AUTO GENERATED NRDP CONFIG FROM WINDOWS INSTALLER
#

[passive checks]

# Host check  - This is to stop "pending check" status in Nagios
%HOSTNAME%|__HOST__ = system/agent_version

#System uptime events :  warn on anything that's been up for less than an hour or over 12 months, and critical on anything that's been up for less than a minute or more than 2 years.
%HOSTNAME%|System_Uptime|300 = system/uptime?check=true&warning=3600:31104000&critical=60:62208000


# Service checks
%HOSTNAME%|CPU Usage = cpu/percent --warning 80 --critical 90 --aggregate avg

##Disk (C: to Z:) Remove the comment for a drive(s) that should be monitored. C: and D: are set by default. 
%HOSTNAME%|Disk_Usage_C|300 = disk/logical/C:|/used_percent --warning 80.00 --critical 90.00 --units Gi
%HOSTNAME%|Disk_Free_Space_C|300 = disk/logical/C:|/free --warning 1200: --critical 1000: --units M
%HOSTNAME%|Disk_Usage_D|300 = disk/logical/D:|/used_percent --warning 80.00 --critical 90.00 --units Gi
%HOSTNAME%|Disk_Free_Space_D|300 = disk/logical/D:|/free --warning

##Memory & Swap
%HOSTNAME%|Swap_Usage|900 = memory/swap --warning 80 --critical 99 --units Gi
%HOSTNAME%|Memory_Usage|900 = memory/virtual --warning 95 --critical 99.99 --units Gi

##Proc
%HOSTNAME%|Process_Count|600 = processes --warning 500 --critical 600
%HOSTNAME%|Process_Count|300 = processes?mem_percent=10&cpu_percent=10&combiner=or&units=Gi&check=true&warning=4&critical=9

##Network for 10 Gb bandwidth, form Windows perf counters.
%HOSTNAME%|NetIF_Bytes_Sent|900 = windowscounters/Network Interface(*)/Bytes Sent/sec?sleep=5&check=true&warning=104217728&critical=134217728
%HOSTNAME%|NetIF_Bytes_Received|900 = windowscounters/Network Interface(*)/Bytes Received/sec?sleep=5&check=true&warning=104217728&critical=134217728
%HOSTNAME%|NetIF_Packets_Sent|900 = windowscounters/Network Interface(*)/Packets Sent/sec?sleep=5&check=true&warning=812740&critical=14880960
%HOSTNAME%|NetIF_Packets_Received|900 = windowscounters/Network Interface(*)/Packets Received/sec?sleep=5&check=true&warning=812740&critical=14880960
%HOSTNAME%|NetIF_Output_Queue_Length|900 = windowscounters/Network Interface(*)/Output Queue Length?check=true&warning=1&critical=2


#NCPA Service Check
%HOSTNAME%|Service_Check_NCPA|60 = services?service=ncpapassive&service=ncpalistener&check=true&status=running


```

Note: Ansible will deploy this template as a [Jinja2](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html#templating-jinja2) template.

#### Variables

Variables for Ansible access to the VMs and NCPA's settings. All these variables' information should be provided form Ansible admin and Nagios admin.

```yml

---
ansible_become_user: <Ansible service account name> # Use secret in Ansible for Prod.
ansible_become_pass: <PassWord> # Use secret in Ansible for Prod.

# for using share server to get NCPA
#share_server: "<share server fqdn>"

template_path: /home/<Ansible service account name>/templates/

nrdp_url: "https://<Nagios XI FQDN>/nrdp/"
nrdp_token: <Nagios API token - Nagios Admin will provide>
nrdp_conf_path: "C:\\Program Files (x86)\\Nagios\\NCPA\\etc\\ncpa.cfg.d"

ncpa_token: <ncpa token - you provide>
ncpa_excutable_url: "https://assets.nagios.com/downloads/ncpa/ncpa-2.2.0.exe" #NCPA's download URL
ncpa_dest_path: c:\Temp\ncpa-2.2.0.exe

```

#### Play-book

Play-book for NCPA for Windows server. Change target hosts or host groups. Rest of variables and configurations should be described in the var file.

```yml
---
# Play that will install the Nagios Cross Platform Agent for Windows.
- name: Play to download, install, and configure Nagios Cross Platform Agent
# All hosts in hte Ansible hosts file
# hosts: all

# Single host example
  hosts: hoge.hoge.com

# Import variables
  vars_files:
    - vars/common_vars.yml
  gather_facts: no

  tasks:
  - name: Create download directory
    win_file:
      path: c:\\Temp
      state: directory

  - name: Copy NCPA installer files locally from Online Nagios Asset site
    win_get_url:
      url: "{{ ncpa_excutable_url }}"
      dest: "{{ ncpa_dest_path }}"
      validate_certs: no

# Example of using share folder as a source.
#    win_copy:
#      src: \\{{ share_server }}\nagios\NCPA_2.2.0
#      dest: c:\installers
#      remote_src: True
#    become: True
#    become_method: runas
#    become_flags: logon_type=new_credentials logon_flags=netcredentials_only

  - name: Install NCPA
    win_package:
      path: "{{ ncpa_dest_path }}"
      arguments: '/S /TOKEN="{{ ncpa_token }}" /NRDPURL="{{ nrdp_url }}" /NRDPTOKEN="{{ nrdp_token }}"'
      product_id: 'NCPA'

  - name: Copy the ncpa.cfg template for Windows
    win_template:
      src: "{{ template_path }}win-nrdp.cfg.j2"
      dest: "{{ nrdp_conf_path }}\\nrdp.cfg"

  - name: Restart NCPA
    win_service:
      name: ncpapassive
      state: restarted
```

### 3. Run play-book

Run the command below from Ansible server:

```bash

$ cd /home/"Ansible service account name"/
$ sudo ansible-playbook win-ncpainstall.yml -vvvv

```

Note: "-vvvv" is a debug message option. Remove if it's not necessary.

You will see outputs like below:

```json
.
.
.
TASK [Restart NCPA] ***************************************************************************************************************
task path: /home/"Ansible service account name"/win-ncpainstall.yml:43
Using module file /usr/lib/python2.7/dist-packages/ansible/modules/windows/win_service.ps1
Pipelining is enabled.
<your windows VM fqdn> ESTABLISH WINRM CONNECTION FOR USER: ansible01admin on PORT 5986 TO <your windows VM fqdn>
EXEC (via pipeline wrapper)
changed: [<your windows VM fqdn>] => {
    "can_pause_and_continue": false,
    "changed": true,
    "depended_by": [],
    "dependencies": [],
    "description": "NCPA Passive Server",
    "desktop_interact": false,
    "display_name": "NCPA Passive - ncpapassive",
    "exists": true,
    "name": "ncpapassive",
    "path": "\"C:\\Program Files (x86)\\Nagios\\NCPA\\ncpa_passive.exe\"",
    "start_mode": "delayed",
    "state": "running",
    "username": "LocalSystem"
}
META: ran handlers
META: ran handlers

PLAY RECAP ************************************************************************************************************************
<your windows VM fqdn> : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
.
.
.

```

Depends on how the Nagios Server is set up, but right after Restart NCPA task above, NCPA will start sending configured metrics to Nagios server. Check with your Nagios admin if they can see the metrics or not. If not, check if NCPA is up and running, and 5693 port is opened.

Also check with these articles:
[NCPA Documentation](https://www.nagios.org/ncpa/help.php)  
[NCPA Common problem](https://support.nagios.com/kb/category.php?id=91)  
[NCPA GitHub](https://github.com/NagiosEnterprises/ncpa/issues)  

-- End
