
=================================
Upgrading from OpenNebula 3.8.x
=================================

This section describes the installation procedure for systems that are already running a 3.8.x OpenNebula. The upgrade will preserve all current users, hosts, resources and configurations; for both Sqlite and MySQL backends.

Read the Compatibility Guide for `4.0 <http://archives.opennebula.org/documentation:archives:rel4.0:compatibility>`_, `4.2 <http://archives.opennebula.org/documentation:archives:rel4.2:compatibility>`_, `4.4 <http://docs.opennebula.org/4.4/release_notes44/compatibility.html>`_, `4.6 <http://docs.opennebula.org/4.6/release_notes/release_notes/compatibility.html>`_, `4.8 <http://docs.opennebula.org/4.8/release_notes/release_notes/compatibility.html>`_, `4.10 <http://docs.opennebula.org/4.10/release_notes/release_notes/compatibility.html>`_, `4.12 <http://docs.opennebula.org/4.12/release_notes/release_notes/compatibility.html>`_, `4.14 <http://docs.opennebula.org/4.14/release_notes/release_notes/compatibility.html>`_ and :ref:`5.0 <compatibility>`, and the `Release Notes <http://opennebula.org/software/release/>`_ to know what is new in OpenNebula 5.0.

.. warning:: With the new :ref:`multi-system DS <system_ds_multiple_system_datastore_setups>` functionality, it is now required that the system DS is also part of the cluster. If you are using System DS 0 for Hosts inside a Cluster, any VM saved (stop, suspend, undeploy) **will not be able to be resumed after the upgrade process**.

.. warning::
    Two drivers available in 3.8 are now discontinued: **ganglia** and **iscsi**.

    -  **iscsi** drivers have been moved out of the main OpenNebula distribution and are available (although not supported) as an `addon <https://github.com/OpenNebula/addon-iscsi>`__.
    -  **ganglia** drivers have been moved out of the main OpenNebula distribution and are available (although not supported) as an `addon <https://github.com/OpenNebula/addon-ganglia>`__.

.. warning::
    There are combinations of **VMware storage** no longer supported (see the VMFS Datastore guide for the supported configurations).

    If you want to upgrade and you are using SSH, NFS or VMFS without SSH-mode, you will need to manually migrate your images to a newly created VMFS with SSH-mode datastore. To do so implies powering off all the VMs with images in any of the deprecated datastores, upgrade OpenNebula, create a VMFS datastore and then manually register the images from those deprecated datastores into the new one. `Let us know <http://opennebula.org/community/mailinglists/>`_ if you have doubts or problems with this process.

Preparation
===========

Before proceeding, make sure you don't have any VMs in a transient state (prolog, migr, epil, save). Wait until these VMs get to a final state (runn, suspended, stopped, done). Check the :ref:`Managing Virtual Machines guide <vm_guide_2>` for more information on the VM life-cycle.

.. warning:: In 4.14 the ``FAILED`` state dissapears. You need to delete all the VMs in this state **before** the new version is installed.

The network drivers in OpenNebula 5.0 are located in the Virtual Network, rather than in the host. The upgrade process may ask you questions about your existing VMs, Virtual Networks and hosts, and as such it is wise to have the following information saved beforehand, since in the upgrade process OpenNebula will be stopped.

.. prompt:: text $ auto

  $ onevnet list -x > networks.txt
  $ onehost list -x > hosts.txt
  $ onevm list -x > vms.txt

The list of valid network drivers in 5.0 Wizard are:

* ``802.1Q``
* ``dummy``
* ``ebtables``
* ``fw``
* ``ovswitch``
* ``vxlan``

Stop OpenNebula and any other related services you may have running: EC2, OCCI, and Sunstone. As ``oneadmin``, in the front-end:

.. code::

    $ sunstone-server stop
    $ oneflow-server stop
    $ econe-server stop
    $ occi-server stop
    $ one stop

Backup
======

Backup the configuration files located in **/etc/one**. You don't need to do a manual backup of your database, the onedb command will perform one automatically.

Installation
============

Follow the :ref:`Platform Notes <uspng>` and the :ref:`Installation guide <ignc>`, taking into account that you will already have configured the passwordless ssh access for oneadmin.

Make sure to run the ``install_gems`` tool, as the new OpenNebula version may have different gem requirements.

It is highly recommended **not to keep** your current ``oned.conf``, and update the ``oned.conf`` file shipped with OpenNebula 5.0 to your setup. If for any reason you plan to preserve your current ``oned.conf`` file, read the :ref:`Compatibility Guide <compatibility>` and the complete oned.conf reference for `3.8 <http://opennebula.org/documentation:archives:rel3.8:oned_conf>`__ and :ref:`5.0 <oned_conf>` versions.

Database Upgrade
================

The database schema and contents are incompatible between versions. The OpenNebula daemon checks the existing DB version, and will fail to start if the version found is not the one expected, with the message 'Database version mismatch'.

You can upgrade the existing DB with the 'onedb' command. You can specify any Sqlite or MySQL database. Check the :ref:`onedb reference <onedb>` for more information.

.. warning:: Make sure at this point that OpenNebula is not running. If you installed from packages, the service may have been started automatically.

.. note::

    If you have a MAC_PREFIX in :ref:`oned.conf <oned_conf>` different than the default ``02:00``, open 
    ``/usr/lib/one/ruby/onedb/local/4.5.80_to_4.7.80.rb`` and change the value of the ``ONEDCONF_MAC_PREFIX`` constant.

After you install the latest OpenNebula, and fix any possible conflicts in oned.conf, you can issue the 'onedb upgrade -v' command. The connection parameters have to be supplied with the command line options, see the :ref:`onedb manpage <cli>` for more information. Some examples:

.. code::

    $ onedb upgrade -v --sqlite /var/lib/one/one.db

.. code::

    $ onedb upgrade -v -S localhost -u oneadmin -p oneadmin -d opennebula

If everything goes well, you should get an output similar to this one:

.. code::

    $ onedb upgrade -v -u oneadmin -d opennebula
    MySQL Password:
    Version read:
    Shared tables 3.8.0 : OpenNebula 3.8.0 daemon bootstrap
    Local tables  3.8.0 : OpenNebula 3.8.0 daemon bootstrap

    MySQL dump stored in /var/lib/one/mysql_localhost_opennebula.sql
    Use 'onedb restore' or restore the DB using the mysql command:
    mysql -u user -h server -P port db_name < backup_file


    >>> Running migrators for shared tables
      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.0_to_3.8.1.rb
      > Done in 0.36s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.1_to_3.8.2.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.2_to_3.8.3.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.3_to_3.8.4.rb
      > Done in 0.56s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.4_to_3.8.5.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.8.5_to_3.9.80.rb

    ATTENTION: manual intervention required
    Virtual Machine deployment files have been moved from /var/lib/one to
    /var/lib/one/vms. You need to move these files manually:

        $ mv /var/lib/one/[0-9]* /var/lib/one/vms

      > Done in 1.10s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.9.80_to_3.9.90.rb

    ATTENTION: manual intervention required
    IM and VM MADS have been renamed in oned.conf. To keep your
    existing hosts working, you need to duplicate the drivers with the
    old names.

    For example, for kvm you will have IM_MAD "kvm" and VM_MAD "kvm", so you
    need to add IM_MAD "im_kvm" and VM_MAD "vmm_kvm"

    IM_MAD = [
          name       = "kvm",
          executable = "one_im_ssh",
          arguments  = "-r 0 -t 15 kvm" ]


    IM_MAD = [
          name       = "im_kvm",
          executable = "one_im_ssh",
          arguments  = "-r 0 -t 15 kvm" ]

    VM_MAD = [
        name       = "kvm",
        executable = "one_vmm_exec",
        arguments  = "-t 15 -r 0 kvm",
        default    = "vmm_exec/vmm_exec_kvm.conf",
        type       = "kvm" ]

    VM_MAD = [
        name       = "vmm_kvm",
        executable = "one_vmm_exec",
        arguments  = "-t 15 -r 0 kvm",
        default    = "vmm_exec/vmm_exec_kvm.conf",
        type       = "kvm" ]

      > Done in 0.41s

      > Running migrator /usr/lib/one/ruby/onedb/shared/3.9.90_to_4.0.0.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.0.0_to_4.0.1.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.0.1_to_4.1.80.rb
      > Done in 0.09s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.1.80_to_4.2.0.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.2.0_to_4.3.80.rb
      > Done in 0.68s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.3.80_to_4.3.85.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.3.85_to_4.3.90.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.3.90_to_4.4.0.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.4.0_to_4.4.1.rb
      > Done in 0.00s

      > Running migrator /usr/lib/one/ruby/onedb/shared/4.4.1_to_4.5.80.rb
      > Done in 0.39s

    Database migrated from 3.8.0 to 4.5.80 (OpenNebula 4.5.80) by onedb command.

    >>> Running migrators for local tables
    Database already uses version 4.5.80

    Total time: 3.60s

.. note:: Make sure you keep the backup file. If you face any issues, the onedb command can restore this backup, but it won't downgrade databases to previous versions.

Check DB Consistency
====================

After the upgrade is completed, you should run the command ``onedb fsck``.

First, move the 4.0 backup file created by the upgrade command to a safe place.

.. code::

    $ mv /var/lib/one/mysql_localhost_opennebula.sql /path/for/one-backups/

.. warning::

    To fix known issues found since the last release, you need to update the fsck file shipped with OpenNebula with the on from the stable branch of the repository:

    .. prompt:: text $ auto

        $ wget https://raw.githubusercontent.com/OpenNebula/one/one-5.0/src/onedb/fsck.rb -O /usr/lib/one/ruby/onedb/fsck.rb

Then execute the following command:

.. code::

    $ onedb fsck -S localhost -u oneadmin -p oneadmin -d opennebula
    MySQL dump stored in /var/lib/one/mysql_localhost_opennebula.sql
    Use 'onedb restore' or restore the DB using the mysql command:
    mysql -u user -h server -P port db_name < backup_file

    Total errors found: 0

Virtual Machine Directories
=================================

.. note:: Only for OpenNebula versions < 3.8.3

If you are upgrading from a version **lower than 3.8.3**, you need to move the Virtual Machine deployment files from '/var/lib/one' to '/var/lib/one/vms':

.. code::

    $ mv /var/lib/one/[0-9]* /var/lib/one/vms

Driver Names
============================

OpenNebula default driver names have changed in the configuration file. Now the names of the vmm and im drivers are not prepended by the type of driver:

* vmm_kvm → kvm
* vmm_xen → xen
* vmm_vmware → vmware
* vmm_ec2 → ec2
* vmm_dummy → dummy
* im_kvm → kvm
* im_xen → xen
* im_vmware → vmware
* im_ec2 → ec2
* im_ganglia → ganglia
* im_dummy → dummy

To keep your existing hosts working, you need to duplicate the drivers with the old names.

For example, for kvm you will have IM_MAD ``kvm`` and VM_MAD ``kvm``, so you need to add IM_MAD ``im_kvm`` and VM_MAD ``vmm_kvm``

.. code::

    IM_MAD = [
          name       = "kvm",
          executable = "one_im_ssh",
          arguments  = "-r 3 -t 15 kvm" ] 
    
    IM_MAD = [
          name       = "im_kvm",
          executable = "one_im_ssh",
          arguments  = "-r 3 -t 15 kvm" ]
    
    VM_MAD = [
        name       = "kvm",
        executable = "one_vmm_exec",
        arguments  = "-t 15 -r 0 kvm",
        default    = "vmm_exec/vmm_exec_kvm.conf",
        type       = "kvm" ]
    
    VM_MAD = [
        name       = "vmm_kvm",
        executable = "one_vmm_exec",
        arguments  = "-t 15 -r 0 kvm",
        default    = "vmm_exec/vmm_exec_kvm.conf",
        type       = "kvm" ]

Manual Intervention Required
============================

.. note:: Ignore this section if onedb didn't output the following message

If you have a datastore configured to use a tm driver not included in the OpenNebula distribution, the onedb upgrade command will show you this message:

.. code::

    ATTENTION: manual intervention required

    The Datastore <id> <name> is using the
    custom TM MAD '<tm_mad>'. You will need to define new
    configuration parameters in oned.conf for this driver, see
    http://opennebula.org/documentation:rel4.4:upgrade

Since OpenNebula 4.4, each tm\_mad driver has a TM\_MAD\_CONF section in oned.conf. If you developed the driver, it should be fairly easy to define the required information looking at the existing ones:

.. code::

    # The  configuration for each driver is defined in TM_MAD_CONF. These
    # values are used when creating a new datastore and should not be modified
    # since they define the datastore behaviour.
    #   name      : name of the transfer driver, listed in the -d option of the
    #               TM_MAD section
    #   ln_target : determines how the persistent images will be cloned when
    #               a new VM is instantiated.
    #       NONE: The image will be linked and no more storage capacity will be used
    #       SELF: The image will be cloned in the Images datastore
    #       SYSTEM: The image will be cloned in the System datastore
    #   clone_target : determines how the non persistent images will be
    #                  cloned when a new VM is instantiated.
    #       NONE: The image will be linked and no more storage capacity will be used
    #       SELF: The image will be cloned in the Images datastore
    #       SYSTEM: The image will be cloned in the System datastore
    #   shared : determines if the storage holding the system datastore is shared
    #            among the different hosts or not. Valid values: "yes" or "no"
     
    TM_MAD_CONF = [
        name        = "lvm",
        ln_target   = "NONE",
        clone_target= "SELF",
        shared      = "yes"
    ]

Update the Drivers
==================

You should be able now to start OpenNebula as usual, running 'one start' as oneadmin. At this point, execute ``onehost sync`` to update the new drivers in the hosts.

.. warning:: Doing ``onehost sync`` is important. If the monitorization drivers are not updated, the hosts will behave erratically.

Setting new System DS
=====================

With the new :ref:`multi-system DS <system_ds_multiple_system_datastore_setups>` functionality, it is now required that the system DS is also part of the cluster. If you are using System DS 0 for Hosts inside a Cluster, any VM saved (stop, suspend, undeploy) **will not be able to be resumed after the upgrade process**.

You will need to have at least one system DS in each cluster. If you don't already, create new system DS with the same definition as the system DS 0 (TM\_MAD driver). Depending on your setup this may or may not require additional configuration on the hosts.

You may also try to recover saved VMs (stop, suspend, undeploy) following the steps described in this `thread of the users mailing list <http://lists.opennebula.org/pipermail/users-opennebula.org/2013-December/025727.html>`__.

Create the Security Group ACL Rule
================================================================================

There is a new kind of resource introduced in 4.12: :ref:`Security Groups <security_groups>`. If you want your existing users to be able to create their own Security Groups, create the following :ref:`ACL Rule <manage_acl>`:

.. code::

    $ oneacl create "* SECGROUP/* CREATE *"

Create the Virtual Router ACL Rule
================================================================================

There is a new kind of resource introduced in 5.0: :ref:`Virtual Routers <vrouter>`. If you want your existing users to be able to create their own Virtual Routers, create the following :ref:`ACL Rule <manage_acl>`:

.. code::

    $ oneacl create "* VROUTER/* CREATE *"

.. note:: For environments in a Federation: This command needs to be executed only once in the master zone, after it is upgraded to 5.0.

Testing
=======

OpenNebula will continue the monitoring and management of your previous Hosts and VMs.

As a measure of caution, look for any error messages in oned.log, and check that all drivers are loaded successfully. After that, keep an eye on oned.log while you issue the onevm, onevnet, oneimage, oneuser, onehost **list** commands. Try also using the **show** subcommand for some resources.

Restoring the Previous Version
==============================

If for any reason you need to restore your previous OpenNebula, follow these steps:

-  With OpenNebula 5.0 still installed, restore the DB backup using 'onedb restore -f'
-  Uninstall OpenNebula 5.0, and install again your previous version.
-  Copy back the backup of /etc/one you did to restore your configuration.

Known Issues
============

If the MySQL database password contains special characters, such as ``@`` or ``#``, the onedb command will fail to connect to it.

The workaround is to temporarily change the oneadmin's password to an ASCII string. The `set password <http://dev.mysql.com/doc/refman/5.6/en/set-password.html>`__ statement can be used for this:

.. code::

    $ mysql -u oneadmin -p

    mysql> SET PASSWORD = PASSWORD('newpass');
