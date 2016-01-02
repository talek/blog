---
layout: post
title: My Portable Oracle Playground
---

Have I told you I bought a new laptop? Yep, it's a [Lenovo
W530](http://shop.lenovo.com/gb/en/laptops/thinkpad/w-series/w530/).  I was
looking for a powerful toy, extensible, not very expensive and not too heavy.
Apparently, this W530 bad boy meets all the requirements: it has a CPU with 4
hyper-threaded cores, it supports up to 32GB of RAM, another HDD can be added by
replacing the CD-ROM with a hard drive caddy, it had a price of ~1000$ on EBay
(refurbished item) and it weights ~3kg. Ah, not to mention the keyboard. I hate
the numeric pad on a laptop, that's why I said NO to the new Lenovo W540.
Another requirement I paid attention to was the display. I needed to be
anti-glare. The old laptop I owned had a glare display and it was a PITA to use
it when light was pointing directly to the display. I got sick and tired of my
face mirrored in that fancy glare display. Anyway, let's cut to the chase and
talk about the way I configured my new laptop.

The OS
------

First of all, I rapidly get rid of the pre-installed Windows 8 operating system
and I installed [Arch Linux](https://www.archlinux.org/).  I love Arch Linux for
its simplicity and for being a rolling upgrade OS. In the past I wasn't amused
at all with [Ubuntu](http://www.ubuntu.com/) or
[Fedora](http://fedoraproject.org/), especially when it came to upgrade to the
next major version. What happened? Well, a lot of my customizations were simply
lost or not compatible with the new version. With Arch Linux I'm always up to
date and I don't bother with major versions at all. Of course, sometimes things
break on Arch Linux as well and manual intervention is needed, but most of these
cases are well documented on the community front page of Arch Linux users. Not
to mention their huge collection of [wiki](https://wiki.archlinux.org/) pages
which are great for learning.

As far as the desktop manager is concerned I am a big fan of
[XFCE](http://www.xfce.org/) which is in line with the simplicity of the Arch
Linux in general.  So, I don't waste my time with fancy desktop managers like
Gnome Shell or Unity. Maybe I'm getting old and that's why I find it difficult
to get into this new style of managing the desktop.

The Goals
---------

Arch Linux is not a supported OS for Oracle, that's for sure. But I didn't want
to pollute the system with all kind of Oracle versions anyway. My plan was to
build up a portable Oracle playground, everything being wrapped around the well
known virtualization tool (yep, I know you know it), VirtualBox. Unlike its
counter part, [VMWare](http://www.vmware.com/), I like VirtualBox more because
of its simplicity. Maybe also because of my bias for Oracle, Virtualbox being
under the same Oracle umbrella? Guilty as charged!  Of course, there are also
many things I don't like when it comes to Virtualbox. For example, why is it not
possible to detach the window of a running VM? I know I can use the headless
mode, but still...

Let's summarize the main goals I want to achieve:

  1. multiple VMs running on my laptop
  2. these virtual hosts must run in their isolated network. I don't want to let
     those VMs to register in my home/company network.
  3. full control of the DNS. I want to access every VM using a nice DNS name
     and those names must be under my full control. I don't want to bother a
     network admin to create a SCAN address for me in order to be able to build
     my own guineea pig RAC.
  4. the VMs must have access to the internet, especially for yum updates or
     other administrative tasks.
  5. creating a new VM should be as easy as possible and automatic. I don't want
     to click, click through all those boring installers.

Tools I Used
------------

Well, nothing special. Let's see:

  1. [bind](http://www.isc.org/downloads/bind/) for my own DNS server
  2. [VirtualBox](https://www.virtualbox.org/) as the tool of choice for
     virtualization
  3. [Packer](http://www.packer.io/), in order to automate the born of a new VM
  4. [squid](http://www.squid-cache.org/), to be able to give internet access to
     the virtual hosts
  5. [bash](https://www.gnu.org/software/bash/), for some custom scripts

Folders Layout
--------------

I'd like to keep things clear, therefore everything related to this virtual
playground was nicely packed within the following folders layout:

    mkdir -p /vmstage/hosts
    mkdir -p /vmstage/packer/bin
    mkdir -p /vmstage/packer/cache
    mkdir -p /vmstage/packer/config
    mkdir -p /vmstage/packer/output
    mkdir -p /vmstage/packer/public
    mkdir -p /vmstage/packer/scripts
    mkdir -p /vmstage/share/disks
    mkdir -p /vmstage/share/bazar

The OS user I use on my laptop is "talek", so I also had to change the ownership:

    chown talek:talek /vmstage -R

A short description of the folders above:

  - in `/vmstage/hosts` are all my VMs
  - in `/vmstage/packer` I put everything related to automating the
    provisioning of a new VM.
  - in `/vmstage/shared` is everything which is shared across VMs. The "bazar"
    is a public share and in "disks" are those shared disks involved especially
    in RAC configurations.

VirtualBox Setup
----------------

Arch Linux guys provide the VirtualBox package from their official repository.
You may use the [this guide](https://wiki.archlinux.org/index.php/VirtualBox) to
complete the whole VirtualBox installation.

As soon as the installation is finished, two settings are important to be
configured:

  1. in "Preferences/General" set the `/vmstage/hosts` as the default machine
     folder.
  2. in "Preferences/Network" add a host-only network called "vboxnet0". Double
     check the IP address of this interface using the configuration button from
     the same preferences window. It should be 192.168.56.1.

DNS Setup
---------

I decided to create an isolated network for my VMs, called "localnet". So, I
have registered my laptop as such (note the domain name):

    [talek@hugsy ~]$ cat /etc/hostname
    hugsy.localnet

Then, assuming you have installed "bind" from the official repository, go ahead
and add the following entries in your `/etc/named.conf` file:

    zone "localnet" IN {
            type master;
            file "localnet.zone";
            allow-update { none; };
            notify no;
    };

    zone "56.168.192.in-addr.arpa" {
            type master;
            file "192.168.56.in-arpa.zone";
    };

In the same `/etc/named.conf` file set the following properties:

    options {
        listen-on { 127.0.0.1; 192.168.56.1; };
        forwarders {192.168.1.1;8.8.8.8;};
    };

By these settings we ask bind to listen on localhost and on the host-only
VirtualBox address. The forwarders are used to delegate the outside internet
requests to the real ISP providers. In the above example, the first IP is my
home rooter address, the second one is from my office.

The first zone is for registering the VM hosts names, the other one is for
reverse DNS, which is especially important for RAC setups. The actual files are:

    [talek@hugsy ~]$ cat /var/named/localnet.zone
    $TTL 7200
    ; localnet
    @       IN      SOA     ns01.localnet. postmaster.localnet. (
                                            2007011601 ; Serial
                                            28800      ; Refresh
                                            1800       ; Retry
                                            604800     ; Expire - 1 week
                                            86400 )    ; Minimum
                    IN      NS      ns01
                    IN      NS      ns02
    ns01            IN      A       0.0.0.0
    ns02            IN      A       0.0.0.0
    localhost       IN      A       127.0.0.1
    @               IN      MX 10   mail
    imap            IN      CNAME   mail
    smtp            IN      CNAME   mail
    @               IN      A       0.0.0.0
    www             IN      A       0.0.0.0
    mail            IN      A       0.0.0.0
    @               IN      TXT     "v=spf1 mx"
    gateway         IN      A       192.168.56.1
    mercur          IN      A       192.168.56.2

    [talek@hugsy ~]$ cat /var/named/192.168.56.in-arpa.zone 
    $ORIGIN 56.168.192.IN-ADDR.ARPA.

    @                       1D IN SOA       localnet. ns.localnet. (
                                            42              ; serial (yyyymmdd##)
                                            3H              ; refresh
                                            15M             ; retry
                                            1W              ; expiry
                                            1D )            ; minimum ttl

                            1D IN NS        localnet.
    1                       1D IN PTR       localnet.
    2                          IN PTR       mercur.localnet.

As you can notice, I have already registered "mercur" host. Now, we need to tell
the system to resolve names through our local DNS server. In order to do this I
changed the `/etc/resolv.conf` file as shown below:

    search localnet
    nameserver 127.0.0.1

In addition, just to avoid the overwrite of this file by the dhcp daemon, I set
the immutable flag:

    chattr +i /etc/resolv.conf

Almost done! The "named" service must be enabled:

    systemctl enable named

Another thing I wanted to happen was to be able to edit and restart the DNS
server from my personal user. So, I added myself to the "named" group:

    gpasswd -a talek named

Then I set the ownership and the rights for the zone files as below:

    chown root:named /var/named/localnet.zone
    chown root:named /var/named/192.168.56.in-arpa.zone
    chmod g+w /var/named/localnet.zone
    chmod g+w /var/named/192.168.56.in-arpa.zone

To restart the DNS server from "talek" user, I used the well-known "sudo". I
just added these using the "visudo" command:

    talek hugsy =NOPASSWD: /usr/bin/systemctl restart named

With this setting in place I can easily do:

    sudo systemctl restart named

Great, done with the DNS!

Packer Setup
------------

Packer IO is not part of the official Arch Linux repositories. However, it is
available on [AUR](https://aur.archlinux.org/) and with
[yaourt](https://wiki.archlinux.org/index.php/yaourt) is just a matter of:

    yaourt packer

Then, I configured a packer template to install an Oracle Linux 6.3.

    [talek@hugsy ~]$ cat /vmstage/packer/config/ol63_x64_non_X.json
    {
      "variables": {
        "name": "",
        "memory": "1024",
        "ip": "127.0.0.1"
      },
      "builders": [{
        "type": "virtualbox-iso",
        "guest_os_type": "Oracle_64",
        "iso_url": "/vmstage/share/bazar/oskits/ol63_x64/V33411-01.iso",
        "iso_checksum": "3D1F4B2D17176821E32281B9B4A3D000",
        "iso_checksum_type": "none",
        "boot_command": [
          "<esc>",
          " linux ksdevice=eth0 ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ol63_non_x.ks",
          "<enter>",
          "<wait>"
        ],
        "boot_wait": "5s",
        "disk_size": 50000,
        "ssh_username": "root",
        "ssh_password": "muci123",
        "http_directory" : "/vmstage/packer/public",
        "http_port_min" : 9001,
        "http_port_max" : 9001,
        "headless" : false,
        "shutdown_command" : "shutdown -h now",
        "shutdown_timeout" : "10s",
        "vm_name" : "{{user `name`}}",
        "output_directory" : "/vmstage/packer/output/{{user `name`}}",
        "vboxmanage": [
          ["modifyvm", "{{.Name}}", "--memory", "{{user `memory`}}"],
          ["modifyvm", "{{.Name}}", "--cpus", "1"],
          ["modifyvm", "{{.Name}}", "--nic1", "nat"],
          ["sharedfolder", "add", "{{.Name}}", "--name", "bazar", "--hostpath", "/vmstage/share/bazar"]
        ]

      }],
      "provisioners": [{
        "type": "shell",
        "script": "/vmstage/packer/scripts/vm_oel_post_setup.sh",
        "environment_vars" : ["PACKER_VMNAME={{user `name`}}", "PACKER_VMIP={{user `ip`}}"]
      }]
    }

I will not cover all the Packer.IO details. They are described on Packer
official web site. However, note that I downloaded the Oracle Linux ISO and I
put it in `/vmstage/share/bazar/oskits/ol63_x64 folder`. This file is
referenced in `iso_url` attribute. The final provisioned image will be stored in
the `/vmstage/packer/output` folder and it will have a preconfigured VBox
share pointing to `/vmstage/share/bazar`. Other interesting attributes are
`boot_command` and `http_directory`. Packer uses them in order to boot the
machine using a kickstart template which is provided in
`/vmstage/packer/public`. The kickstart file is this:

    [talek@hugsy ~]$ cat /vmstage/packer/public/ol63_non_x.ks
    #platform=x86, AMD64, or Intel EM64T
    #version=DEVEL
    # Firewall configuration
    firewall --disabled
    # Install OS instead of upgrade
    install
    # Use CDROM installation media
    cdrom
    # Root password
    rootpw --iscrypted $1$mcvra6e/$B3vCQ0OpTXdOZf/YNDU1O.
    # System authorization information
    auth  --useshadow  --passalgo=sha512
    # Use text install
    text
    firstboot --disable
    # System keyboard
    keyboard us
    # System language
    lang en_US
    # SELinux configuration
    selinux --disabled
    # Installation logging level
    logging --level=info
    # Reboot after installation
    reboot
    # System timezone
    timezone --isUtc Europe/Bucharest
    # Use eth0 by default
    network --device eth0
    #network --device eth1
    # System bootloader configuration
    bootloader --location=mbr
    # Partition clearing information
    clearpart --all
    zerombr
    # Disk partitioning information
    part / --fstype="ext4" --grow --size=1

    %packages
    @base
    kernel-devel
    kernel-headers
    gcc
    binutils
    make
    patch
    glibc-headers
    glibc-devel

    %end

As soon as the OS in the virtual host is installed the Packer "provisioners"
kick in. In my configuration file I use a script which does all the post
installation tasks:

    [talek@hugsy ~]$ cat /vmstage/packer/scripts/vm_oel_post_setup.sh 
    #!/bin/bash

    sleep 10
    echo $PACKER_VMNAME
    hostname $PACKER_VMNAME
    echo "NETWORKING=yes" > /etc/sysconfig/network
    echo "HOSTNAME=$PACKER_VMNAME" >> /etc/sysconfig/network
    echo "GATEWAY=192.168.56.1" >> /etc/sysconfig/network

    echo "IPADDR=$PACKER_VMIP" >> /etc/sysconfig/network-scripts/ifcfg-eth0
    echo "DNS1=192.168.56.1" >> /etc/sysconfig/network-scripts/ifcfg-eth0
    echo 'IPV6INIT="no"' >> /etc/sysconfig/network-scripts/ifcfg-eth0
    echo 'BOOTPROTO="static"' >> /etc/sysconfig/network-scripts/ifcfg-eth0

    printf "\n$PACKER_VMIP ${PACKER_VMNAME}.localnet ${PACKER_VMNAME}\n" >> /etc/hosts

    printf "\nAddressFamily inet\n" >> /etc/ssh/sshd_config
    printf "\nGSSAPIAuthentication no\n" >> /etc/ssh/sshd_config
    printf "\nUseDNS no\n" >> /etc/ssh/sshd_config
    printf "\nHost *\n" >> /etc/ssh/ssh_config
    printf "\n  AddressFamily inet\n" >> /etc/ssh/ssh_config

    printf "\nexport http_proxy='http://192.168.56.1:8080'\n" >> /etc/profile
    printf "\nexport ftp_proxy='http://192.168.56.1:8080'\n" >> /etc/profile

    printf "\nproxy=http://192.168.56.1:8080\n" >> /etc/yum.conf

    printf "\nappend domain-name \"localnet\";\n" >> /etc/dhcp/dhclient-eth0.conf

    mount -o loop /root/VBoxGuestAdditions.iso /mnt
    /mnt/VBoxLinuxAdditions.run --nox11

    exit 0

The script above configures the networking of the new provisioned host and it
installs the VBox guest additions package.

The Proxy Setup
---------------

Our VMs talks to each other using the VirtualBox Host-Only adapter. This adapter
supports the concept of an isolated network I was talking about at the very
beginning of this post. The network traffic can take place just between VMs and
the host, but not outside. This means that the main host can reach the Internet
using the other non-VBox adapter, but the VMs can't. So, the workaround I found
is to use a proxy which listens on the Host-Only adapter and then funnels the
outside requests to the other non-VBox adapter.

So, I installed "squid" from the official repositories:

    pacman -S squid

And then I used
[this](https://dl.dropboxusercontent.com/u/6430366/blog/squid.conf)
configuration file as `/etc/squid/squid.conf`.

The last step was to enable the squid service:

    systemctl enable squid

Automation Baby
---------------

Everything needs to be glued nicely so that to be able to create a new VM as
easy as possible. So, in `/vmstage/Packer/bin` I created the following bash
script:

    #!/bin/bash

    PACKER_DIR="$(dirname $( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd ))"
    EDITOR="/home/talek/vim/bin/vim"

    printf 'VM Host Name: '
    read host_name

    printf 'Memory footprint: '
    read memory

    echo "Add $host_name to DNS..."
    $EDITOR -c 'silent edit /var/named/192.168.56.in-arpa.zone | silent split /var/named/localnet.zone'

    echo "Restart DNS daemon..."
    sudo systemctl restart named

    echo "Check DNS registration..."
    ip=`dig ${host_name} A +short +search | tail -n1`
    echo "IP address of $host_name is: $ip"

    if [ -z "$ip" ]; then
      echo "DNS registration error!"
      exit 1
    fi

    nslookup $ip > /dev/null
    if [ $? -ne 0 ]; then
      echo "Reverse DNS registration error!"
      exit 1
    else
      echo "Reverse DNS ok."
    fi

    printf "\n\n***Choose what do you want to provision***\n\n"
    PS3='OS template: '
    options=($(ls $PACKER_DIR/config))
    select opt in "${options[@]}"
    do
      echo "you selected: $opt"
      export PACKER_CACHE_DIR='/vmstage/packer/cache'
      packer-io build -var "name=$host_name" -var "memory=$memory" -var "ip=$ip" $PACKER_DIR/config/$opt

      echo "Importing image..."
      VBoxManage import ${PACKER_DIR}/output/${host_name}/${host_name}.ovf --options keepallmacs
      vboxmanage modifyvm ${host_name} --nic1 hostonly
      vboxmanage modifyvm ${host_name} --hostonlyadapter1 "vboxnet0"

      rm -rf $PACKER_DIR/output/$host_name
      break
    done

    printf "\n\n***done***\n\n"

Of course, I gave execute rights to this script and now I launch it as often as
needed. Below is a sample of the output of this script:

    [talek@hugsy bin]$ ./new_host
    VM Host Name: mercur
    Memory footprint: 2048
    Add mercur to DNS...
    Restart DNS daemon...
    Check DNS registration...
    IP address of mercur is: 192.168.56.2
    Reverse DNS ok.


    ***Choose what do you want to provision***

    1) ol63_x64_non_X.json
    2) ol63_x64_with_X.json
    OS template: 1
    you selected: ol63_x64_non_X.json
    virtualbox-iso output will be in this color.

    Warnings for build 'virtualbox-iso':

    * A checksum type of 'none' was specified. Since ISO files are so big,
    a checksum is highly recommended.

    ==> virtualbox-iso: Downloading or copying Guest additions
        virtualbox-iso: Downloading or copying: file:///usr/lib/virtualbox/additions/VBoxGuestAdditions.iso
    ==> virtualbox-iso: Downloading or copying ISO
        virtualbox-iso: Downloading or copying: file:///vmstage/share/bazar/oskits/ol63_x64/V33411-01.iso
    ==> virtualbox-iso: Starting HTTP server on port 9001
    ==> virtualbox-iso: Creating virtual machine...
    ==> virtualbox-iso: Creating hard drive...

    ....

    Successfully imported the appliance.


    ***done***

Mission accomplished! I know there's a lot of room for improvements but for now
I'm quite pleased with the results.
