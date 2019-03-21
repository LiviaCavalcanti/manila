====================
Dell EMC VMAX driver
====================

The Dell EMC Shared File Systems service driver framework (EMCShareDriver)
utilizes the Dell EMC storage products to provide the shared file systems
to OpenStack. The Dell EMC driver is a plug-in based driver which is designed
to use different plug-ins to manage different Dell EMC storage products.

The VMAX plug-in manages the VMAX to provide shared file systems. The EMC
driver framework with the VMAX plug-in is referred to as the VMAX driver
in this document.

This driver performs the operations on VMAX eNAS by XMLAPI and the file
command line. Each back end manages one Data Mover of VMAX. Multiple
Shared File Systems service back ends need to be configured to manage
multiple Data Movers.

Requirements
~~~~~~~~~~~~

-  VMAX eNAS OE for File version 8.1 or higher

-  VMAX Unified or File only

-  The following licenses should be activated on VMAX for File:

   -  CIFS

   -  NFS

   -  SnapSure (for snapshot)

   -  ReplicationV2 (for create share from snapshot)

Supported shared file systems and operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The driver supports CIFS and NFS shares.

The following operations are supported:

-  Create a share.

-  Delete a share.

-  Allow share access.

   Note the following limitations:

   -  Only IP access type is supported for NFS.
   -  Only user access type is supported for CIFS.

-  Deny share access.

-  Create a snapshot.

-  Delete a snapshot.

-  Create a share from a snapshot.

While the generic driver creates shared file systems based on cinder
volumes attached to nova VMs, the VMAX driver performs similar operations
using the Data Movers on the array.

Pre-configurations on VMAX
~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Enable Unicode on Data Mover.

   The VMAX driver requires that the Unicode is enabled on Data Mover.

   .. warning::

      After enabling Unicode, you cannot disable it. If there are some
      file systems created before Unicode is enabled on the VMAX,
      consult the storage administrator before enabling Unicode.

   To check the Unicode status on Data Mover, use the following VMAX eNAS File
   commands on the VMAX control station:

   .. code-block:: console

      server_cifs <MOVER_NAME> | head
      # MOVER_NAME = <name of the Data Mover>

   Check the value of I18N mode field. UNICODE mode is shown as
   ``I18N mode = UNICODE``.

   To enable the Unicode for Data Mover, use the following command:

   .. code-block:: console

      uc_config -on -mover <MOVER_NAME>
      # MOVER_NAME = <name of the Data Mover>

   Refer to the document Using International Character Sets on VMAX for
   File on `EMC support site <http://support.emc.com>`_ for more
   information.

#. Enable CIFS service on Data Mover.

   Ensure the CIFS service is enabled on the Data Mover which is going
   to be managed by VMAX driver.

   To start the CIFS service, use the following command:

   .. code-block:: console

      server_setup <MOVER_NAME> -Protocol cifs -option start [=<n>]
      # MOVER_NAME = <name of the Data Mover>
      # n = <number of threads for CIFS users>

   .. note::

      If there is 1 GB of memory on the Data Mover, the default is 96
      threads. However, if there is over 1 GB of memory, the default
      number of threads is 256.

   To check the CIFS service status, use the following command:

   .. code-block:: console

      server_cifs <MOVER_NAME> | head
      # MOVER_NAME = <name of the Data Mover>

   The command output will show the number of CIFS threads started.

#. NTP settings on Data Mover.

   VMAX driver only supports CIFS share creation with share network
   which has an Active Directory security-service associated.

   Creating CIFS share requires that the time on the Data Mover is in
   sync with the Active Directory domain so that the CIFS server can
   join the domain. Otherwise, the domain join will fail when creating
   a share with this security service. There is a limitation that the
   time of the domains used by security-services, even for different
   tenants and different share networks, should be in sync. Time
   difference should be less than 10 minutes.

   We recommend setting the NTP server to the same public NTP
   server on both the Data Mover and domains used in security services
   to ensure the time is in sync everywhere.

   Check the date and time on Data Mover with the following command:

   .. code-block:: console

      server_date <MOVER_NAME>
      # MOVER_NAME = <name of the Data Mover>

   Set the NTP server for Data Mover with the following command:

   .. code-block:: console

      server_date <MOVER_NAME> timesvc start ntp <host> [<host> ...]
      # MOVER_NAME = <name of the Data Mover>
      # host = <IP address of the time server host>

   .. note::

      The host must be running the NTP protocol. Only 4 host entries
      are allowed.

#. Configure User Mapping on the Data Mover.

   Before creating CIFS share using VMAX driver, you must select a
   method of mapping Windows SIDs to UIDs and GIDs. DELL EMC recommends
   using usermapper in single protocol (CIFS) environment which is
   enabled on VMAX eNAS by default.

   To check usermapper status, use the following command syntax:

   .. code-block:: console

      server_usermapper <movername>
      # movername = <name of the Data Mover>

   If usermapper does not start, use the following command
   to start the usermapper:

   .. code-block:: console

      server_usermapper <movername> -enable
      # movername = <name of the Data Mover>

   For a multiple protocol environment, refer to Configuring VMAX eNAS User
   Mapping on `EMC support site <http://support.emc.com>`_ for
   additional information.

#. Configure network connection.

   Find the network devices (physical port on NIC) of the Data Mover that
   has access to the share network.

   To check the device list, go
   to :menuselection:`Unisphere > Settings > Network > Device`.

Back-end configurations
~~~~~~~~~~~~~~~~~~~~~~~

The following parameters need to be configured in the
``/etc/manila/manila.conf`` file for the VMAX driver:

.. code-block:: ini

   emc_share_backend = vmax
   emc_nas_server = <IP address>
   emc_nas_password = <password>
   emc_nas_login = <user>
   vmax_server_container = <Data Mover name>
   vmax_share_data_pools = <Comma separated pool names>
   share_driver = manila.share.drivers.dell_emc.driver.EMCShareDriver
   vmax_ethernet_ports = <Comma separated ports list>
   emc_ssl_cert_verify = True
   emc_ssl_cert_path = <path to cert>

- `emc_share_backend`
    The plug-in name. Set it to ``vmax`` for the VMAX driver.

- `emc_nas_server`
    The control station IP address of the VMAX system to be managed.

- `emc_nas_password` and `emc_nas_login`
    The fields that are used to provide credentials to the
    VMAX system. Only local users of VMAX File is supported.

- `vmax_server_container`
    Name of the Data Mover to serve the share service.

- `vmax_share_data_pools`
    Comma separated list specifying the name of the pools to be used
    by this back end. Do not set this option if all storage pools
    on the system can be used.
    Wild card character is supported.

    Examples: pool_1, pool_*, *

- `vmax_ethernet_ports (optional)`
    Comma-separated list specifying the ports (devices) of Data Mover
    that can be used for share server interface. Do not set this
    option if all ports on the Data Mover can be used.
    Wild card character is supported.

    Examples: fxg-9-0, fxg-_*, *

- `emc_ssl_cert_verify (optional)`
    By default this is True, setting it to False is not recommended

- `emc_ssl_cert_path (optional)`
    The path to the This must be set if emc_ssl_cert_verify is True which is
    the recommended configuration.  See ``SSL Support`` section for more
    details.

Restart of the ``manila-share`` service is needed for the configuration
changes to take effect.

SSL Support
-----------

#. Run the following on eNas Control Station, to display the CA certification
   for the active CS.

   .. code-block:: console

      $ /nas/sbin/nas_ca_certificate -display

   .. warning::

      This cert will be different for the secondary CS so if there is a failover
      a different certificate must be used.

#. Copy the contents and create a file with a .pem extention on your manila host.

   .. code-block:: ini

      -----BEGIN CERTIFICATE-----
      the cert contents are here
      -----END CERTIFICATE-----

#. To verify the cert by running the following and examining the output:

   .. code-block:: console

      $ openssl x509 -in test.pem -text -noout

   .. code-block:: ini

      Certificate:
       Data:
           Version: 3 (0x2)
           Serial Number: xxxxxx
       Signature Algorithm: sha1WithRSAEncryption
           Issuer: O=VNX Certificate Authority, CN=xxx
           Validity
               Not Before: Feb 27 16:02:41 2019 GMT
               Not After : Mar  4 16:02:41 2024 GMT
           Subject: O=VNX Certificate Authority, CN=xxxxxx
           Subject Public Key Info:
               Public Key Algorithm: rsaEncryption
                   Public-Key: (2048 bit)
                   Modulus:
                       xxxxxx
                   Exponent: xxxxxx
           X509v3 extensions:
               X509v3 Subject Key Identifier:
                   xxxxxx
               X509v3 Authority Key Identifier:
                   keyid:xxxxx
                   DirName:/O=VNX Certificate Authority/CN=xxxxxx
                   serial:xxxxx

               X509v3 Basic Constraints:
                   CA:TRUE
               X509v3 Subject Alternative Name:
                   DNS:xxxxxx, DNS:xxxxxx.localdomain, DNS:xxxxxxx, DNS:xxxxx
       Signature Algorithm: sha1WithRSAEncryption
               xxxxxx

#. As it is the capath and not the cafile that is expected, copy the file to either
   new directory or an existing directory (where other .pem files exist).

#. Run the following on the directory

   .. code-block:: console

      $ c_rehash $PATH_TO_CERTS

#. Update manila.conf with the directory where the .pem exists.

   .. code-block:: ini

       emc_ssl_cert_path = /path_to_certs/

#. Restart manila services.


Restrictions
~~~~~~~~~~~~

The VMAX driver has the following restrictions:

-  Only IP access type is supported for NFS.

-  Only user access type is supported for CIFS.

-  Only FLAT network and VLAN network are supported.

-  VLAN network is supported with limitations. The neutron subnets in
   different VLANs that are used to create share networks cannot have
   overlapped address spaces. Otherwise, VMAX may have a problem to
   communicate with the hosts in the VLANs. To create shares for
   different VLANs with same subnet address, use different Data Movers.

-  The **Active Directory** security service is the only supported
   security service type and it is required to create CIFS shares.

-  Only one security service can be configured for each share network.

-  The domain name of the ``active_directory`` security
   service should be unique even for different tenants.

-  The time on the Data Mover and the Active Directory domains used in
   security services should be in sync (time difference should be less
   than 10 minutes). We recommended using same NTP server on both
   the Data Mover and Active Directory domains.

-  On eNAS, the snapshot is stored in the SavVols. eNAS system allows the
   space used by SavVol to be created and extended until the sum of the
   space consumed by all SavVols on the system exceeds the default 20%
   of the total space available on the system. If the 20% threshold
   value is reached, an alert will be generated on eNAS. Continuing to
   create snapshot will cause the old snapshot to be inactivated (and
   the snapshot data to be abandoned). The limit percentage value can be
   changed manually by storage administrator based on the storage needs.
   We recommend the administrator configures the notification on the
   SavVol usage. Refer to Using eNAS SnapSure document on `EMC support
   site <http://support.emc.com>`_ for more information.

-  eNAS has limitations on the overall numbers of Virtual Data Movers,
   filesystems, shares, and checkpoints. Virtual Data Mover(VDM) is
   created by the eNAS driver on the eNAS to serve as the Shared File
   Systems service share server. Similarly, the filesystem is created,
   mounted, and exported from the VDM over CIFS or NFS protocol to serve
   as the Shared File Systems service share. The eNAS checkpoint serves
   as the Shared File Systems service share snapshot. Refer to the NAS
   Support Matrix document on `EMC support
   site <http://support.emc.com>`_ for the limitations and configure the
   quotas accordingly.

Driver options
~~~~~~~~~~~~~~

Configuration options specific to this driver:

.. include:: ../../tables/manila-vmax.inc
