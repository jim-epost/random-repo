# random-repo
Testing stuff.
## Testing

Testing ansible playbooks on live systems can be... challenging. Or dangerous.  Or both. So I set up a testbed of virtual machines on my workstation that can be tested and trashed without locking everyone out of ssh.  It uses [Vagrant](https://www.vagrantup.com/) to spin up Ubuntu or CentOS images quickly.

    vagrant
    ├── files
    │   ├── Anne-Cybil.pub
    │   ├── bootstrap-ansible-RedHat.yml
    │   └── bootstrap-ansible-Ubuntu.yml
    └── vms
        ├── centos5
        ├── centos6
        ├── centos7
        ├── precise
        ├── trusty
        └── xenial

To start a fresh instance of a Ubuntu 14 ("trusty") virtual machine,

    cd vagrant/vms/trusty
    vagrant destroy 
    vagrant up --provider virtualbox --provision
    
which zaps any previous instance, installs and starts the virtual machine image running under VirtualBox, and runs the provisioning hooks in the virtual machine's Vagrantfile.

    trusty
    ├── Anne-Cybil.pub -> ../../files/Anne-Cybil.pub
    ├── bootstrap-ansible.yml -> ../../files/bootstrap-ansible-Ubuntu.yml
    └── Vagrantfile

The Vagrantfile provisioning hook's shell section handles any prerequisites the ansible hook needs. The ansible hook then bootstraps the normal ansible configuration on the virtual machine, using `bootstrap-ansible.yml`, which is symlinked to the file of the right OS flavor.  The ansible hook copies the `ansible_user`'s account's ssh public key from `Anne-Cybil.pub`.

The Vagrantfile sets a static IP address in the 192.168.177.0/24 range and also a unique forwarded port for vagrant's ssh connection

    config.vm.define "trusty-vm"
    config.vm.network "private_network", ip: "192.168.177.224"
    config.vm.network "forwarded_port", id: "ssh", guest:22, host:2224
    
That IP address has to agree with the virtual machines's `ansible_host` entry in the test inventory file, `test-inv.yml`

    trusty-vm:
      ansible_host:
        192.168.177.224

My workstation is 192.168.177.1.  Each virtual machine also has the usual NAT'ed network interface through VirtualBox, which `vagrant ssh` uses.

The test inventory file groups all of the virtual machines under `[test]`, and `group_vars/test` has a couple of things specific
to the test environment

    ansible_ssh_common_args : "-o StrictHostKeyChecking=no"
    munin_master_ip : 192.168.177.1
    
The first is to avoid problems with the `~/.ssh/known_hosts` file when a new image is spun up, and the second is some quirk with testing munin; the munin master has to use my workstation's private IP address instead of my workstation's normal IP address.  I haven't bothered to figure out why.

There's a `test-site.yml` playbook at the top level that simply imports the `test.yml` playbook, which invokes whatever role or roles are to be tested, say

    ---
    - hosts:
        all
      roles:
        - common
    
With that setup,

    ansible-playbook -i test-inv.yml test-site.yml
    
would run the `common` role on all the test virtual machines currently running,

    ansible-playbook -i test-inv.yml test-site.yml -- tags vars,ntp
    
would run just the `ntp` task `roles/common/tasks/ntp.yml` after pulling in data from `roles/common/vars`,

and

    ansible-playbook -i test-inv.yml test-site.yml --tags vars,ntp --limit trusty-vm
    
would run the `ntp` task on just the Ubuntu 14 virtual machine.







