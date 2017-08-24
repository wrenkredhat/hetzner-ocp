# Set up OCP on libvirt on Fedora 25

## Insalling server

You need to install Fedora 25 to the server via VNC console from Heztner Robot UI.

You'll need VNC client to access VNC based installer, like [VNC Viewer](https://www.realvnc.com/en/download/viewer/)

Activate VNC console, [screenshot](images/01_vnc_console.png)

Reboot server [screenshot](images/02_reboot.png)


Minimal install + basic tools is enough

I dont have good opinion about how to partition disks...In my tests I created a big root
````
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdb               8:16   0  1.8T  0 disk
└─sdb1            8:17   0  1.8T  0 part
  └─fedora-root 253:0    0  3.6T  0 lvm  /
sda               8:0    0  1.8T  0 disk
├─sda2            8:2    0  1.8T  0 part
│ ├─fedora-swap 253:1    0 23.7G  0 lvm  [SWAP]
│ └─fedora-root 253:0    0  3.6T  0 lvm  /
└─sda1            8:1    0    1G  0 part /boot
````

Remember to set some good root password during installation.

## Initialize tools

Install ansible and git

````
dnf install -y ansible git
````

Create ssh key (no passphrase)

````
ssh-keygen
````

Clone configs and playbook. You need to add your ssh key to your Gitlab account.

````
git clone ssh://git@gitlab.consulting.redhat.com:2222/tigers/hetzner-ocp.git
````

## Install libvirt and setup environment

````
cd hetzner-ocp
ansible-playbook playbooks/setup.yml
export RHN_USERNAME=yourid@redhat.com
export RHN_PWD=yourpwd
export ANSIBLE_HOST_KEY_CHECKING=False
````

## Provision guest

Check ```vars/guests.yml``` and modify it to correspond your environment. By default following VMs are installed:

* bastion
* master01
* infranode01
* node01


Sample guest definition

````
    - name: bastion
      url: http://static.ocp.ninja/rhel73/
      cpu: 1
      mem: 1024
      virt_type: kvm
      virt_hypervisor: hvm
      network: bridge=virbr0
      os:
          type: linux
          variant: rhel7.3
      disks:
          os:
            size: 12
            options: format=qcow2,cache=none,io=native
          data:
            size: 1
            options: format=qcow2,cache=none,io=native
      extra_args: ip=dhcp inst.ks=http://static.ocp.ninja/ks/rhel-73-ocp.ks console=tty0 console=ttyS0,115200 quiet systemd.show_status=yes
````

Basically you need to change only num of VMs and/or cpu and mem values.

Provision VMs
````
ansible-playbook playbooks/provision.yml
````

Provisioning of the hosts take a while and they are in running state until installation is finnished. When guest list is empty, all guest are done and ready to be started.

````
virsh list
# installation still running
 Id    Name                           State
----------------------------------------------------
 34    bastion                        running
 35    master01                       running
 36    infranode01                    running
 37    node01                         running
 38    node02                         running
 39    node03                         running

````
When list of running guests is empty, all guests have been installed.

Start all VMs

````
ansible-playbook playbooks/startall.yml
````

Copy SSH key to all VMs. Password is p

````
ansible-playbook -i /tmp/inventory -k playbooks/prepare_ssl.yml
````

## Prepare bastion for OCP installation
You'll need your RHN username, password and subscription pool id (Employee SKU). You can get pool id from https://access.redhat.com/management/

When you have all mentioned above run.

````
ansible-playbook -i /tmp/inventory playbooks/prepare_guests.yml --extra-vars "rhn_username=$RHN_USERNAME rhn_password=$RHN_PWD"
````

## Clean up everything

````
ansible-playbook playbooks/clean.yml
````
