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

## Principles

This exercise and cluster must be:

- Documented and repeatable
- Functional and useful
- Automated from setup to teardown (as much as possible)
- As declarative, generic, agnostic, and non-opionionated as possible

## Tools

- Git for documentation and saving YAML and scripts
- Ansible for setup and teardown (and bash where necessary)
- kubeadm for setup
- Flannel for networking
- MetalLB for load balancing
- A NFS provisioner TBD for persistent storage
- Prometheus and Grafana for monitoring and reporting
- kubectl and YAML for deployments

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

### Local Storage

The internal SSDs are not very large and I would like my persistent data to exist where any worker can access it so losing a node doesn't mean maunally restoring data. For this reason, I will use a NFS provisioner to leverage my NAS for persistent volumes.

### CPU and Memory

The NUCs come with legacy generation Intel Core i5 CPUs and 8GB of RAM. Nothing special but far more powerful than my Raspberry Pi cluster.

## Citations and Acknowledgements

Much of what I've learned has be "self-taught" as in I didn't have a mentor or teacher. But I'm inspired by and borrow heavily from Jeff Geerling.

I'm sticking to the official Kubernetes documentation for much of the Kubernetes-specific setup.