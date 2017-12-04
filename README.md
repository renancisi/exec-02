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
 
 # Quick start

The fastest way to get running:

 * run: `# apt-get install -y ruby-libvirt`
 * run: `# apt-get install -y libxslt-dev libxml2-dev libvirt-dev zlib1g-dev`
 * Disable the default libvirt network: `# virsh net-autostart default --disable && systemctl restart libvirt-bin`


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

# Documentation and reasons for the used technologies
