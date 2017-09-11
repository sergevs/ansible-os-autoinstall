# Ansible for OS autoinstall

# Ovewrview

The project goal is to provide a flexible and as simple as possible configuration tool for unattended OS installation via network for various linux distributions. 
It was designed to setup dozen of production and other servers and can be used by professional system administrators or for development/testing/education purposes.
The playbook contains ready to use installation templates for various distributions. 

A copy of a OS distribution itself is not required(!) and with the supplied default configuration an OS installation will be performed
via internet from http://mirror.yandex.ru site. However if you are going to setup dozens of servers it's highly recommended 
to make and configure a local mirror for the required OS.

# Features

Templates for unattended installation included in the playbook:
* CentOS 6-7
* Fedora 21-26
* Debian Wheezy, Jessie, Stretch
* Ubunty Trusty, Xenial
* OpenSuse 12.\*, 13.\*
* (OpenSuse)Leap 41.\*, 42.\*

Other distributions which support kickstart/preseed/autoyast can be easily added.

The playbook supports any number of an OS configurations so if you have several groups of servers with different configuration 
you can describe it in separate autoinstall 'receipt' files.

# Caveats

The playbook is not supposed to cover setup for network boot services environment but focused on providing a ‘framework’ 
for generation autoinstall files for various distributions. It also generates pxe boot menus and have a basic functionality to generate EFI boot files.

In the order to effectively use the playbook for your particular purposes you have to understand the principles of Linux network boot
and have a base knowledge about autoinstall files for the required OS: 
* Fedora/Centos/Redhat: [kickstart]( http://pykickstart.readthedocs.io/en/latest/ )
* Debian/Ubuntu:        [preseed]( https://wiki.debian.org/DebianInstaller/Preseed )
* OpenSuse/Leap:        [autoyast]( https://www.suse.com/documentation/sles-12/book_autoyast/data/book_autoyast.html )

# Requirements

Ansible version >= 2.2.1.0

In the order to perform install you need working PXE boot environment services:
* dhcp: network boot ip address configuration
* tftp: pxe/efi image serving
* http: serve autoinstall (kickstart/preseed/autoyast) files

# Description

## Playbook structure
As the ansible standard book structure appeared to be superfluous for the project it was designed to make a simplified and more convenient directory structure.
Global configuration variables defined at [site_vars.yml](site_vars.yml)
Templates for autoinstall files at [templates/ks](templates/ks) , [templates/preseed](templates/preseed) and [templates/yast](templates/yast) directories.

The configuration implemented using hosts file and customizing autoinstall files.

## Host variables
### Networking parameters
If **ip** is not specified, DHCP will be used
* **ip**      : ip address of the host
* **netmask** : netmask of network
* **gateway** : default network gateway
* **hostname** : inventory host record
* **domain** : host domain
* **nameservers** : list of DNS servers

### OS parameters
Autoinstall file location and name convention:
templates/<ks|preseed|yast>/<**os**>.<**osver**>[.**type**].cfg.j2
as an instance [centos.7.cfg.j2](templates/ks/centos.7.cfg.j2)

* **os**      : the name of OS
* **osver**   : OS version
* **type**    : a custom variant for installation
* **rootpw**  : encrypted root password. default password is root

### Menu parameters
* **menu**    : all hosts with the same **menu** will be grouped and shown on the second level menu items

## Inventory groups
A host must belong to at least one of the groups:
* **ks**      : the group for kickstart
* **preseed** : the group for preseed
* **yast**    : the group for autoyast

The group is used to identify the type of autoinstall.

# Usage

    ansible-playbook -i hosts site.yaml

### See also [hosts](hosts)

### An example to setup network boot services for CentOS 7

    # configure epel release
    yum install http://mirror.yandex.ru/epel/7/x86_64/e/epel-release-7-10.noarch.rpm

    # install required packages
    yum install syslinux-tftpboot tftp-server dhcp xinetd nginx 

    # provide minimal dhcp configuration
    cat <<'EOF' > /etc/dhcp/dhcpd.conf
    allow unknown-clients;
    default-lease-time 1800;
    max-lease-time 7200;

    subnet 10.0.0.0 netmask 255.255.255.0 {
      range 10.0.0.128 10.0.0.254;
      option broadcast-address 10.0.0.255;
      option domain-name-servers 10.0.0.1;
      option domain-name localdomain;
      default-lease-time 1800;
      max-lease-time 7200;
      option netbios-name-servers 10.0.0.1;
      option routers 10.0.0.1;
      next-server 10.0.0.1;
      filename "gpxelinux.0";
    }
    EOF

    # configure nginx to serve autoinstall files
    cat <<'EOF' >/etc/nginx/default.d/ks.conf                                             
    location /kickstart {
    root /var/lib/;
    }
    EOF
  
    # start services
    service dhcpd start
    service xinetd start
    service nginx start
    

