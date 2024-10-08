===========================================================
Enterprise Linux Quickstart (Rocky Linux, CentOS, and RHEL)
===========================================================

Install Warewulf
================

The preferred way to install Warewulf on Enterprise Linux is using the
the RPMs published in `GitHub releases`_. For example, to install the
v4.5.7 release on Enterprise Linux 9:

.. code-block:: bash

   dnf install https://github.com/warewulf/warewulf/releases/download/v4.5.7/warewulf-4.5.7-1.el9.x86_64.rpm

Packages are available for el7, el8, and el9.

.. note::
   CentOS 7 (and el7 in general) uses ``yum``, and does not provide ``dnf``.

.. _GitHub releases: https://github.com/warewulf/warewulf/releases

Install Warewulf from source
----------------------------

If you prefer, you can also install Warewulf from source.

.. code-block:: bash

   dnf install git
   dnf install epel-release
   dnf install golang {libassuan,gpgme}-devel unzip tftp-server dhcp nfs-utils ipxe-bootimgs-{x86,aarch64}

   git clone https://github.com/warewulf/warewulf.git
   cd warewulf
   make
   make install

.. note::
   CentOS 7 (and el7 in general) only provides a single ``ipxe-bootimgs`` package.

Configure firewalld
===================

Restart firewalld to register the added service file, add the service
to the default zone, and reload.

.. code-block:: bash

   systemctl restart firewalld
   firewall-cmd --permanent --add-service=warewulf
   firewall-cmd --permanent --add-service=dhcp
   firewall-cmd --permanent --add-service=nfs
   firewall-cmd --permanent --add-service=tftp
   firewall-cmd --reload

Configure Warewulf
==================

Edit the file ``/etc/warewulf/warewulf.conf`` and ensure that you've
set the appropriate configuration parameters. Here are some of the
defaults for reference assuming that ``10.0.0.1/22`` is the IP
address of your cluster's private network interface.

.. code-block:: yaml

   WW_INTERNAL: 45
   ipaddr: 10.0.0.1
   netmask: 255.255.252.0
   network: 10.0.0.0
   warewulf:
     port: 9873
     secure: false
     update interval: 60
     autobuild overlays: true
     host overlay: true
     syslog: false
     datastore: /usr/share
     grubboot: false
   dhcp:
     enabled: true
     template: default
     range start: 10.0.1.1
     range end: 10.0.1.255
     systemd name: dhcpd
   tftp:
     enabled: true
     tftproot: /var/lib/tftpboot
     systemd name: tftp
     ipxe:
       "00:00": undionly.kpxe
       "00:07": ipxe-snponly-x86_64.efi
       "00:09": ipxe-snponly-x86_64.efi
       00:0B: arm64-efi/snponly.efi
   nfs:
     enabled: true
     export paths:
     - path: /home
       export options: rw,sync
       mount options: defaults
       mount: true
     - path: /opt
       export options: ro,sync,no_root_squash
       mount options: defaults
       mount: false
     systemd name: nfs-server
   container mounts:
   - source: /etc/resolv.conf
     dest: /etc/resolv.conf
     readonly: true
   paths:
     bindir: /usr/bin
     sysconfdir: /etc
     localstatedir: /var/lib
     ipxesource: /usr/share/ipxe
     srvdir: /var/lib
     firewallddir: /usr/lib/firewalld/services
     systemddir: /usr/lib/systemd/system
     wwoverlaydir: /var/lib/warewulf/overlays
     wwchrootdir: /var/lib/warewulf/chroots
     wwprovisiondir: /var/lib/warewulf/provision
     wwclientdir: /warewulf

.. note::

   The DHCP range from ``10.0.1.1`` to ``10.0.1.255`` is dedicated for
   DHCP during node boot and should not overlap with any static IP
   address assignments.

Enable and start the Warewulf service
=====================================

Warewulf provides a service, ``warewulfd``, which responds to node
boot requests.

.. code-block:: bash

   systemctl enable --now warewulfd

Configure system services automatically
=======================================

There are a number of services and configurations that Warewulf relies
on to operate. You can configure all such services with ``wwctl
configure --all``.

.. code-block:: bash

   wwctl configure --all

.. note::

   If you just installed the system fresh and have SELinux enforcing,
   you may need to run ``restorecon -Rv /var/lib/tftpboot/`` to label
   files written to q`tftpboot``.

Add a base node image container
===============================

This will pull a basic node image container from Docker Hub and import the
default running kernel from the controller node and set both in the
"default" node profile.

.. code-block:: bash

   wwctl container import docker://ghcr.io/warewulf/warewulf-rockylinux:9 rockylinux-9 --setdefault

Configure a default node profile
================================

In this example, all nodes share the netmask and gateway
configuration, so we can set them in the default profile.

.. code-block:: bash

   wwctl profile set -y default --netmask=255.255.252.0 --gateway=10.0.0.1
   wwctl profile list

Add a node
==========

Adding nodes can be done while setting configurations in one
command. Here we set the IP address of the default interface; and
setting the node to be discoverable causes the HW address to be added
to the configuration as the node boots.

Node names must be unique. If you are managing multiple clusters with
overlapping names, distinguish them using dot notation.

.. code-block:: bash

   wwctl node add n1 --ipaddr=10.0.2.1 --discoverable=true
   wwctl node list -a n1

The full node configuration comes from both cascading profiles and
node configurations which always supersede profile configurations.

Turn on your compute node and watch it boot!
