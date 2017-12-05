About this document
===================

minishift's quickstart guide https://docs.openshift.org/latest/minishift/getting-started/quickstart.html performed on CentOS 7

# Table of Contents
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Example](#example-minishift-box)
- [Documentation](#documentation-and-reasons-for-the-used-technologies)

# Prerequisites

The fastest way to get running:

 * [install vagrant](https://www.vagrantup.com/downloads.html)
 * [install plugin dependencies]( https://github.com/pradels/vagrant-libvirt)
 * [Enable BIOS Virt - Workstation Example](http://techgenix.com/enable-virtual-vt-xamd-v-support-in-vmware-workstation-8-183/)
 
 # Quick start

The fastest way to get running:

 * run: `# apt-get install -y ruby-libvirt`
 * run: `# apt-get install -y libxslt-dev libxml2-dev libvirt-dev zlib1g-dev`
 * run: `# adduser $userForVagrant libvirtd`
 * Disable the default libvirt network: `# virsh net-autostart default --disable && systemctl restart libvirt-bin`
 * run: `# git clone https://github.com/marcindulak/minishift-quickstart-with-vagrant-centos7.git`
 * run: `# cd minishift-quickstart-with-vagrant-centos7 && vagrant plugin install vagrant-libvirt && vagrant plugin install vagrant-scp`
 * run: `# vagrant up `


## Example Minishift Box

```yaml
# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

Vagrant.configure(2) do |config|
  config.vm.define 'minishift' do |machine|
    machine.vm.box = 'centos/7'
    machine.vm.box_url = machine.vm.box
    machine.vm.provider 'libvirt' do |p|
      p.memory = 3072
      p.cpus = 2
      p.nested = true
    end
  end
  # disable IPv6 on Linux
  $linux_disable_ipv6 = <<SCRIPT
sysctl -w net.ipv6.conf.default.disable_ipv6=1
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.lo.disable_ipv6=1
SCRIPT
  # setenforce 0
  $setenforce_0 = <<SCRIPT
if test `getenforce` = 'Enforcing'; then setenforce 0; fi
#sed -Ei 's/^SELINUX=.*/SELINUX=Permissive/' /etc/selinux/config
SCRIPT
  # stop firewalld
  $systemctl_stop_firewalld = <<SCRIPT
systemctl stop firewalld.service
SCRIPT
  config.vm.define "minishift" do |machine|
    # check for nested virtualization!
    machine.vm.provision :shell, :inline => 'egrep -c "(vmx|svm)" /proc/cpuinfo > /dev/null'
    machine.vm.provision :shell, :inline => 'hostname minishift', run: 'always'
    machine.vm.provision :shell, :inline => $setenforce_0, run: 'always'
    machine.vm.provision :shell, :inline => $linux_disable_ipv6, run: 'always'
    machine.vm.provision :shell, :inline => 'yum -y install wget git'
    machine.vm.provision :shell, :inline => 'yum -y groups install "X Window System"'
    machine.vm.provision :shell, :inline => 'systemctl set-default graphical.target'
  end
end
```
# Configure minishift on the guest VM:
 
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum -y install wget git'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'wget -q https://copr.fedorainfracloud.org/coprs/marcindulak/minishift/repo/fedora-rawhide/marcindulak-minishift-fedora-rawhide.repo -P /etc/yum.repos.d'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum -y install minishift'"`
 
 # Install Docker:
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum -y install yum-utils device-mapper-persistent-data lvm2'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum -y install docker-ce'" `
 * run: `# vagrant ssh minishift -c "sudo su - -c 'mkdir -p /etc/docker&& echo { \\\"storage-driver\\\": \\\"devicemapper\\\" } > /etc/docker/daemon.json'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'systemctl start docker'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'docker run hello-world'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'systemctl enable docker'"`
 
 # Configure libvirtd on the minishift VM, add user vagrant to the libvirt group:
 * run: `# vagrant ssh minishift -c "sudo su - -c 'yum -y install libvirt qemu-kvm'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'systemctl start libvirtd'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'systemctl enable libvirtd'"`
 * run: `# vagrant ssh minishift -c "sudo su - -c 'usermod -aG libvirt vagrant'"`

# Creating Static Json file on Host Machine and moving it into VM Guest:
* run: `# echo -e '{\n\t"service":​ ​{\n\t\t"oracle":​ ​"ok",​ ​\n\t\t"redis":​ ​"ok",​ ​\n\t\t"mongo":​ ​"down",​ ​\n\t\t"pgsql":​ ​"down",​ ​\n\t\t"mysql":​ ​"ok"\n\t}\n}' > /tmp/lab.json`
* run: `# vagrant scp /tmp/lab.json  minishift:/tmp/`

# Creating Nginx docker container with static json file:
* run: `vagrant ssh minishift -c "sudo su - -c 'docker run --name some-nginx -d -p 8080:80 -v /tmp/lab.json:/usr/share/nginx/html/lab.json:ro -d nginx'"`

## Test return example - ( `vagrant ssh minishift -c "sudo su - -c 'curl localhost:8080/lab.json'"` )

```yaml
{
	"service":​ ​{
		"oracle":​ ​"ok",​ ​
		"redis":​ ​"ok",​ ​
		"mongo":​ ​"down",​ ​
		"pgsql":​ ​"down",​ ​
		"mysql":​ ​"ok"
	}
}
```

# Documentation and reasons for the used technologies
For exercise two I decided to use the vagrant as host virtualizer of the minishift for the following reasons:
- Host / Application compatibility
- possibility to work with emulated libvirt on vmware workstation

For static json creation, I decided to manually create the host and move into the container, making it easy to attach to the Nginx container with the -v function. But we could have addressed the creation of the same in the run call with the environment variables of nginx and some simple loop.
