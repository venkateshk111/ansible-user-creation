# Ansible User ID Creation Automation - Linux

## Introduction

Ansible playbook to create Ansible service id on all ansible managed nodes

## Pre-Requisite:

1. Existence of one User id, here after referred as remote user (Example: venkatesh) with sudo permission to run Ansible Playbook
2. Remote user (venkatesh) password must be same across all the managed nodes where Ansible user id will be created.
3. Managed nodes must be allowing ssh connection from the Controller node (Ensure Security group or Firewall is not blocking ssh connection from Controller)
4. Ensure PasswordAuthentication on managed nodes is enabled under file "/etc/ssh/sshd_config". If not, Plesae enable it and restart sshd service. Ensure that you have approval from customer to make these changes.
5. On the Controller node, Ansible service id with sudo permission must exist.
6. On the Controller node, Ansible service id public key (id_rsa.pub) must be present under following location.  
    - /home/< ansible_user_id >/.ssh/id_rsa.pub
7. Playbook to be executed on controller node logged in as Ansible user

## Running the Playbook

1. Ensure you are logged onto your Ansible Controller Node as Ansible user.
2. Edit the "hosts" file to list hostnames or IP address of the managed nodes

Example:
```
10.20.3.15
10.20.3.20
172.31.2.22
```
3. Edit the file "ansible-user-creation/vars/main.yml" as below

    * **remote_user_id** : User name configured on managed nodes that will be used as remote_user.  
    * **ansible_user_id**: User name for Ansible service id to be created on managed nodes
    * **group_name**     : Group name for Ansible service id to be created on managed nodes

Example:
```
remote_user_id: venkatesh 
ansible_user_id : ansible
group_name : ansiblegrp 
```

4. Run the playbook using the ansible-playbook command as shown below

```
ansible-playbook -i hosts user-creation.yml -k -K
```

5. Enter password (for remote_user_id) as requested at the prompt
```
SSH password:
BECOME password[defaults to SSH password]:
```

### Sample Output
```
[ansible@ip-172-31-4-254 automation]$ ansible-playbook -i hosts user-creation.yml -k -K
SSH password:
BECOME password[defaults to SSH password]:

PLAY [all] ***************************************************************************************************************************************************

TASK [ansible-user-creation : Create Ansible User] *************************************************************************************************************
changed: [172.31.3.202]

TASK [ansible-user-creation : Add Ansible user to sudoers file] ************************************************************************************************
changed: [172.31.3.202]

TASK [ansible-user-creation : Copy ssh keys to authorized_keys file] *******************************************************************************************
changed: [172.31.3.202]

PLAY RECAP ***************************************************************************************************************************************************
172.31.3.202               : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[ansible@ip-172-31-4-254 automation]$
```

## Testing simple playbook using Ansible service id

Run ping playbook using the ansible-playbook command as shown below
```
ansible-playbook -i hosts ping.yml
```

### Sample Output
```
[ansible@ip-172-31-4-254 automation]$ ansible-playbook -i hosts ping.yml

PLAY [all] ***************************************************************************************************************************************************

TASK [ping all the hosts from inventory file] ****************************************************************************************************************
ok: [172.31.3.202]

TASK [Print ping Results] ************************************************************************************************************************************
ok: [172.31.3.202] => {
    "ping_pong": {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/libexec/platform-python"
        },
        "changed": false,
        "failed": false,
        "ping": "pong"
    }
}

PLAY RECAP ***************************************************************************************************************************************************
172.31.3.202               : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[ansible@ip-172-31-4-254 automation]$
```

## Troubleshooting

Below are some common known errors you could possibly encounter during this playbook run, and recommended resolution for the same.

1. Error: Host Key checking is enabled and sshpass does not support this

```
TASK [Create Ansible User] ***********************************************************************************************************************************
fatal: [172.31.3.202]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible because Host Key checking is enabled and sshpass does not support this.  Please add this host's fingerprint to your known_hosts file to manage this host."}
```
Solution: 
On Ansible Controller Node , disable (uncomment) SSH key host checking in /etc/ansible/ansible.cfg , to do so run the following command.
```
sed -i.bak 's/#host_key_checking/host_key_checking/g' /etc/ansible/ansible.cfg
```

2. Error: Failed to connect to the host via ssh

```
TASK [Create Ansible User] ***********************************************************************************************************************************
fatal: [172.31.3.202]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: venkatesh@172.31.3.202: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).", "unreachable": true}

```
Solution:  
Ensure you have approval from your customer to perform below actions     
- On Managed nodes, edit the file "/etc/ssh/sshd_config" and uncomment  "PasswordAuthentication yes"  and comment "PasswordAuthentication no"   
- Restart sshd service ( _service restart sshd_ or _systemctl restart sshd_)  

3. Error: Remote user "venkatesh" does not have sudo privileges 
```
TASK [Create Ansible User] ***********************************************************************************************************************************
fatal: [172.31.3.202]: FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/libexec/platform-python"}, "changed": false, "module_stderr": "Shared connection to 172.31.3.202 closed.\r\n", "module_stdout": "\r\nvenkatesh is not in the sudoers file.  This incident will be reported.\r\n", "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error", "rc": 1}
Remote user venkatesh does not have sudo previlages , ensure user venkatesh is part of sudo users

```
Solution:   
Ensure you have approval from your customer to perform below actions.  
Each customer may have different ways of doing this so please be sure before performing below action  
Add remote user to sudo group, one way to do it is 

For Amazon Linux, Amazon Linux 2, RHEL, and CentOS:
```
usermod -aG wheel venkatesh
```
For Ubuntu:
```
usermod -aG sudo venkatesh
```

4.  Error: Invalid/incorrect password OR Incorrect sudo password
```
TASK [ansible-user-creation : Create Ansible User] *************************************************************************************************************
fatal: [172.31.3.202]: UNREACHABLE! => {"changed": false, "msg": "Invalid/incorrect password: Permission denied, please try again.", "unreachable": true}
```
OR 
```
TASK [ansible-user-creation : Create Ansible User] *************************************************************************************************************
fatal: [172.31.3.202]: FAILED! => {"msg": "Incorrect sudo password"}

```
Solution:  
Ensure you enter/type correct password for remote user or the password is not locked out 

5.  Error: Failed to connect to the host via ssh , "Connection timed out"
```
TASK [Gathering Facts] ***************************************************************************************************************************************
fatal: [172.31.3.202]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 172.31.3.202 port 22: Connection timed out", "unreachable": true}

```
Solution:  
- Ensure the managed node is up and running and you are able to login to server manually.
- Managed nodes must be allowing ssh connection from the Controller node (Ensure Security groups or Firewall is not blocking ssh connection from Controller)


6.  Error: Failed to connect to the host via ssh , "No route to host"
```
TASK [Gathering Facts] ***************************************************************************************************************************************
fatal: [172.31.2.22]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: connect to host 172.31.2.22 port 22: No route to host", "unreachable": true}

```
Solution:  
Ensure that the "hosts" file has the correct hostname or the IP address  

7.  Error: Failed to connect to the host via ssh , "No route to host"
```
TASK [ansible-user-creation : Create Ansible User] ***************************************************************************************************************************
fatal: [80.0.20.205]: FAILED! => {"changed": false, "module_stderr": "Shared connection to 80.0.20.205 closed.\r\n", "module_stdout": "/bin/sh: /usr/local/bin/python: No such file or directory\r\n", "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error", "rc": 127}
```
Solution:  
On your Ansible Controller edit the hosts file used for executing the playbook to include below entry
```
[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

**License**

MIT License

Copyright (c) 2021 venkateshk111

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
