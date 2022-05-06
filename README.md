# k8s-metal-lab

Just documenting my baremetal kubernetes setup.

- [k8s-metal-lab](#k8s-metal-lab)
  - [Principles](#principles)
  - [Tools](#tools)
  - [Rationale](#rationale)
  - [Hardware](#hardware)
    - [Physical NICs](#physical-nics)
    - [Local Storage](#local-storage)
    - [CPU and Memory](#cpu-and-memory)
  - [Citations and Acknowledgements](#citations-and-acknowledgements)
  - [SSH and Passwordless sudo](#SSH)
  - [Ansible Setup](#ansible-setup)
  - [TODO](#TODO)

## Principles

This exercise and cluster must be:

- Documented and repeatable
- Functional and useful
- Automated from setup to teardown (as much as possible)
- As declarative, generic, agnostic, and un-opionionated as possible

## Tools

- Git for documentation and saving playbooks, manifests, and scripts
- Debian 11 for stability and supportability
- Ansible for setup and teardown (and bash where necessary)
- containerd for container runtime
- kubeadm for setup
- Flannel for networking
- MetalLB for load balancing
- A NFS provisioner TBD for persistent storage
- Prometheus and Grafana for monitoring and reporting
- kubectl and helm for deployments

## Rationale

I've used Kubespray before but it didn't give me any practice with Ansible and largely hid the nuts and bolts of Kubernetes. I eventually used Ansible for my Raspberry Pi cluster but the cluster was underpowered.

Kubernetes the Hardway is a field too far. This, I hope, is a happy medium that would prove useful in many scenarios.

## Hardware

- Four NUCs
- Consumer-grade router with a built-in DHCP server that I can resrves IP address on
- Gigabit switch plugged into my router
- Simple NAS for NFS storage

### Physical NICs

To keep things simple, I'll statically assign IPs or create reservations in my DHCP server. I'll also name the host by IP address since I plan to make every node a worker node, including those with control plane roles.

|MAC|IP|HOSTNAME|
|---|---|---|
|94:c6:91:15:02:32|192.168.50.10|ip-192-168-50-10|
|f4:4d:30:63:b9:1e|192.168.50.11|ip-192-168-50-11|
|f4:4d:30:6a:3f:19|192.168.50.12|ip-192-168-50-12|
|f4:4d:30:6b:c8:d6|192.168.50.13|ip-192-168-50-13|

### Storage

The internal SSDs are not very large and I would like my persistent data to exist where any worker can access it so losing a node doesn't mean maunally restoring data. For this reason, I will use a NFS provisioner to leverage my NAS for persistent volumes.

### CPU and Memory

The NUCs come with legacy generation Intel Core i5 CPUs and 8GB of RAM. Nothing special but far more powerful than my Raspberry Pi cluster.

## Citations and Acknowledgements

Much of what I've learned has be "self-taught" as in I didn't have a mentor or teacher. But I'm inspired by and borrow heavily from Jeff Geerling.

I'm sticking to the official Kubernetes documentation for much of the Kubernetes-specific setup.

## SSH and Passwordless sudo

### Setup SSH keys

Create a SSH keys.

```bash
ssh-keygen
```

```text
Generating public/private rsa key pair.
Enter file in which to save the key (/home/matt/.ssh/id_rsa): 
Created directory '/home/matt/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/matt/.ssh/id_rsa
Your public key has been saved in /home/matt/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:uY+jYmFsKqiCJOzLuH5Z7sRwybN6gl2NQ2ENpJt8cnU matt@lappy
The key's randomart image is:
+---[RSA 3072]----+
|    .oo          |
|    .o .         |
|   .. .. E       |
|  . +.o ..       |
|.  B.Bo S        |
|.o  %=o. .       |
|* o+==. .        |
|Bo.==o  .o       |
|B*oo+o.....      |
+----[SHA256]-----+

```

Follow interactive setup instructions or press Enter three times to accept defaults.

Add the new key to the SSH agent.

```bash
ssh-add
```

```text
Identity added: /home/matt/.ssh/id_rsa (matt@lappy)
```

Copy public key to each server.

```bash
ssh-copy-id debian@192.168.50.10
```

Expected output:

```text
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
debian@192.168.50.10's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'debian@192.168.50.10'"
and check to make sure that only the key(s) you wanted were added.
```

Repeat this step for each server.

Confirm we can SSH into each server without a password.

```bash
ssh debian@192.168.50.10
```

### Passwordless sudo

While on the server, enable passwordless sudo.

```bash
sudo visudo
```

Update the following line:

```text
%sudo   ALL=(ALL:ALL) ALL
```

to read:

```text
%sudo  ALL=(ALL:ALL) NOPASSWD: ALL
```

Save the file and log out of the server.

## Ansible Setup

Install Ansible (I'm using Arch but follow distrobution's preferred installation method).

```bash
yay -S ansible-core ansible
```

Alternatively, [install with pip](#https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip).

Test basic ansible functionality.

```bash
ansible localhost -m ping
```

Expected output:

```text
[WARNING]: No inventory was parsed, only implicit localhost is available
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

If this error is encountered:

```text
ModuleNotFoundError: No module named 'markupsafe'
```

Install using pip.

```bash
pip install markupsafe
```

Create an inventory file.

```text
# inventory.ini
192.168.50.10
192.168.50.11
192.168.50.12
192.168.50.13
```

Initialize an ansible config.

```bash
ansible-config init --disabled > ansible.cfg
```

Disable host key checking.

```text
# ansible.cfg
[defaults]
...
host_key_checking=False
```

Set default remote user.

```text
# ansible.cfg
[defaults]
...
remote_user=debian
```

Set default inenvotry file.

```text
[defaults]
...
inventory=inventory.ini
```

Ping the inventory.

```bash
ansible all -m ping
```

Expected output:

```text
192.168.50.10 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
192.168.50.13 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
192.168.50.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
192.168.50.11 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

Create some utility playbooks.

Reboot:

```yaml
# reboot.yaml
---
- name: Reboot
  hosts: all
  become: yes
  tasks:
    - name: Reboot
      ansible.builtin.reboot:
```

Shutdown:

```yaml
# shutdown.yaml
---
- name: Shutdown
  hosts: all
  become: yes
  tasks:
    - name: Shutdowm
      community.general.shutdown:
```

Update:

```yaml
# update.yaml
---
- name: Update
  hosts: all
  become: true
  tasks:  
    - name: Upgrade installed apt packages
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 600
        upgrade: dist
    - name: Check if a reboot is required
      stat:
        path: /var/run/reboot-required
      notify: reboot_required
  handlers:
    - name: reboot_required
      reboot:
        reboot_timeout: 3600
```

Example usage:

```bash
ansible-playbook update.yaml
```

## Playbook

```yaml

```

## TODO

- Baremetal or virtual OS deployment via PXE
- Automate statically assigning IP instead of using DHCP reservations.
