Setting up a two node (using VMs) OpenStack on Fedora
=====================================================

[1] Packages
------------

Install Fedora trunk repo::

    $ cd /etc/yum.repos.d && wget \
    http://repos.fedorapeople.org/repos/openstack/openstack-trunk/fedora-openstack-trunk.repo


[2] Setup Keystone
------------------

Install and setup Keystone::

    $ yum install openstack-keystone \
      openstack-utils dnsmasq-utils -y
    $ openstack-db --init --service keystone

It'll ask whether to install and run mysql-server, say yes::

    $ mysqld is not running.  Would you like to start it now? (y/n): y
    Note: Forwarding request to 'systemctl enable mysqld.service'.
    ln -s '/usr/lib/systemd/system/mysqld.service' '/etc/systemd/system/multi-user.target.wants/mysqld.service'
    Since this is a fresh installation of MySQL, please set a password for the 'root' mysql user.
    Enter new password for 'root' mysql user: 
    Enter new password again: 
    Verified connectivity to MySQL.
    Creating 'keystone' database.
    Initializing the keystone database, please wait...
    Complete!


Setup Keystone service token and endpoint::

    $ export SERVICE_TOKEN=$(openssl rand -hex 10)
    $ export SERVICE_ENDPOINT=http://192.168.122.126:35357/v2.0
    $ echo $SERVICE_TOKEN > /tmp/ks_admin_token
    $ cat /tmp/ks_admin_token 
    216b79a72dc87f53b275


Initialize certificates for PKI based tokens::

    $  keystone-manage pki_setup --keystone-user keystone \
       --keystone-group keystone
    Generating RSA private key, 2048 bit long modulus
    ...+++
    ..............................+++
    e is 65537 (0x10001)
    Generating RSA private key, 2048 bit long modulus
    .........+++
    .....+++
    e is 65537 (0x10001)
    Using configuration from /etc/keystone/ssl/certs/openssl.conf
    Check that the request matches the signature
    Signature ok
    The Subject's Distinguished Name is as follows
    countryName           :PRINTABLE:'US'
    stateOrProvinceName   :PRINTABLE:'Unset'
    localityName          :PRINTABLE:'Unset'
    organizationName      :PRINTABLE:'Unset'
    commonName            :PRINTABLE:'www.example.com'
    Certificate is to be certified until Jul 15 14:28:23 2023 GMT (3650 days)
    
    Write out database with 1 new entries
    Data Base Updated


Make 'keystone' user owner of /etc/keystone/ssl directory::

    $ chown -R keystone:keystone /etc/keystone/ssl 
 

Start Keystone service, check status, and enable it::

    $ systemctl start openstack-keystone.service
    $ systemctl status openstack-keystone.service
    $ sysetmctl enable openstack-keystone.service


Create Keystone services
~~~~~~~~~~~~~~~~~~~~~~~~

Create Keystone service::

    $ keystone service-create --name keystone --type identity \
     --description "Keystone Identity Service"
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |    Keystone Identity Service     |
    |      id     | 714bc75e53dd40989e07328066fd4b17 |
    |     name    |             keystone             |
    |     type    |             identity             |
    +-------------+----------------------------------+

Create an endpoint::

    $ keystone endpoint-create --service_id 714bc75e53dd40989e07328066fd4b17 \
    > --publicurl 'http://192.168.122.126:5000/v2.0' \
    > --adminurl 'http://192.168.122.126:35357/v2.0' \
    > --internalurl 'http://192.168.122.126:5000/v2.0'
    +-------------+-----------------------------------+
    |   Property  |               Value               |
    +-------------+-----------------------------------+
    |   adminurl  | http://192.168.122.126:35357/v2.0 |
    |      id     |  65630ff8cabe470b921c7327185fc57d |
    | internalurl |  http://192.168.122.126:5000/v2.0 |
    |  publicurl  |  http://192.168.122.126:5000/v2.0 |
    |    region   |             regionOne             |
    |  service_id |  714bc75e53dd40989e07328066fd4b17 |
    +-------------+-----------------------------------+


Create users
~~~~~~~~~~~~

Create a user::

    $ keystone user-create --name admin --pass redhat
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | db463b145db94ce8bd542107512ebbbc |
    |   name   |              admin               |
    | tenantId |                                  |
    +----------+----------------------------------+


Create a role::

    $ keystone role-create --name admin 
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |    id    | d2fa7f1917b246f6a399c18805310232 |
    |   name   |              admin               |
    +----------+----------------------------------+


Create a tenant::

    $ keystone tenant-create --name admin
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |                                  |
    |   enabled   |               True               |
    |      id     | 36cb79612cf84bbb82f16c025994ccee |
    |     name    |              admin               |
    +-------------+----------------------------------+


Add a role to the 'admin' user, and assign the user to a tenant::

    $ keystone user-role-add --user admin \
      --role admin --tenant admin


Create a Keystone rc file for 'admin' user::

    $ cat >> ~/keystonerc_admin <<EOF
    > export OS_USERNAME=admin
    > export OS_TENANT_NAME=admin
    > export OS_PASSWORD=redhat
    > export OS_AUTH_URL=http://192.168.122.126:35357/v2.0/
    > export PS1='[\u@\h \W(keystone_admin)]\$ '
    > EOF


Source the keystonerc file, and list users::

    $ . keystonerc_admin 

    $ keystone user-list
    +----------------------------------+-------+---------+-------+
    |                id                |  name | enabled | email |
    +----------------------------------+-------+---------+-------+
    | db463b145db94ce8bd542107512ebbbc | admin |   True  |       |
    +----------------------------------+-------+---------+-------+


Create an unprivileged user::

    $ keystone user-create --name kashyap --pass redhat
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | c8c56c6ba2f442a2bce009d05d030f50 |
    |   name   |             kashyap              |
    | tenantId |                                  |
    +----------+----------------------------------+


Create a role 'user'::

    $ keystone role-create --name user
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |    id    | b480a9af4f734b478c9b08aaede17183 |
    |   name   |               user               |
    +----------+----------------------------------+

Create a tenant::

    $ keystone tenant-create --name ostenant
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |                                  |
    |   enabled   |               True               |
    |      id     | 7534bb0241584f06a8a9263bbdb9e3f6 |
    |     name    |             ostenant             |
    +-------------+----------------------------------+

Associate the user to the role and tenant::

    $ keystone user-role-add --user kashyap \
      --role user --tenant ostenant

Create a Keystone rc file for the user::

    $ cat >> ~/keystonerc_kashyap <<EOF
    > export OS_USERNAME=kashyap
    > export OS_TENANT_NAME=ostenant
    > export OS_PASSWORD=redhat
    > export OS_AUTH_URL=http://192.168.122.126:35357/v2.0/
    > export PS1='[\u@\h \W(keystone_kashyap)]\$ '
    > EOF

(Optionally, test it)::

   $ . keystonerc_kashyap
   $ keystone user-list
   $ . keystonerc_admin
   $ keystone user-list


Turn off Qpid authentication (for test deployment)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

   $ yum install qpid-cpp-server -y
   $ sed -i 's/auth=.*/auth=no/g' /etc/qpidd.conf
   $ systemctl start qpidd.service
   $ systemctl enable qpidd.service


[3] Setup Glance
----------------

Install
~~~~~~~

    $ yum install openstack-glance -y

Initialize the Glance database::

    $ openstack-db --init --service glance
    Please enter the password for the 'root' MySQL user: 
    Verified connectivity to MySQL.
    Creating 'glance' database.
    Initializing the glance database, please wait...
    Complete!


Integrate Glance with Keystone
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a 'services' tenant, which will be the credentials used by
individual services 
::

    $ keystone tenant-create --name services
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |                                  |
    |   enabled   |               True               |
    |      id     | 3bfff90676ce4902bd2c207388f28488 |
    |     name    |             services             |
    +-------------+----------------------------------+


Create a user for Glance::

    $ keystone user-create --name glance --pass redhat
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | 902a7cc3f9a74baf9bb9d0dc71042c0d |
    |   name   |              glance              |
    | tenantId |                                  |
    +----------+----------------------------------+


Associate the user with the 'services' tenant::

    $ keystone user-role-add --user glance \
      --role admin --tenant services


Update the glance-api.conf::

    $ openstack-config --set /etc/glance/glance-api.conf \
      paste_deploy flavor keystone
    $ openstack-config --set /etc/glance/glance-api.conf \
      keystone_authtoken admin_tenant_name services
    $ openstack-config --set /etc/glance/glance-api.conf \
      keystone_authtoken admin_user glance
    $ openstack-config --set /etc/glance/glance-api.conf \
      keystone_authtoken admin_password redhat


Update the glance-registry.conf::

    $ openstack-config --set /etc/glance/glance-registry.conf \
      paste_deploy flavor keystone
    $ openstack-config --set /etc/glance/glance-registry.conf \
      keystone_authtoken admin_tenant_name services
    $ openstack-config --set /etc/glance/glance-registry.conf \
      keystone_authtoken admin_user glance
    $ openstack-config --set /etc/glance/glance-registry.conf \
      keystone_authtoken admin_password redhat


Start the 'openstack-glance-registry' service, check the status, and
enable the service
::

    $ systemctl start openstack-glance-registry.service
    $ systemctl status openstack-glance-registry.service
    $ systemctl enable openstack-glance-registry.service


Install python-cinderclient package and start the glance-api.service::

    $ yum install python-cinderclient -y
    $ systemctl start openstack-glance-api.service
    $ systemctl status openstack-glance-api.service
    $ systemctl enable openstack-glance-api.service


Create glance service and an endpoint::

    $ keystone service-create --name glance --type image --description
    "Glance Image Service"
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |       Glance Image Service       |
    |      id     | 4eb17742f585461b836bb38548893874 |
    |     name    |              glance              |
    |     type    |              image               |
    +-------------+----------------------------------+


Create endpoint::

    keystone endpoint-create --service_id 4eb17742f585461b836bb38548893874 \
    --publicurl http://192.168.122.126:9292 \
    --adminurl http://192.168.122.126:9292 \
    --internalurl http://192.168.122.126:9292
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    |   adminurl  |   http://192.168.122.126:9292    |
    |      id     | 9a28d7b761204774a5539c4a694f1a35 |
    | internalurl |   http://192.168.122.126:9292    |
    |  publicurl  |   http://192.168.122.126:9292    |
    |    region   |            regionOne             |
    |  service_id | 4eb17742f585461b836bb38548893874 |
    +-------------+----------------------------------+


Check if endpoint was created successfully::

    $ glance index

(The above command should return an empty table.)


Add an image to Glance
~~~~~~~~~~~~~~~~~~~~~~

Import Fedora 19 image into glance::

    $ glance image-create --name fedora19 --is-public true \
      --disk-format qcow2 --container-format bare \
      < Fedora-x86_64-19-20130627-sda.qcow2

List it::

    $ glance image-list


[4] Setup Cinder
----------------

Install
~~~~~~~

    $ keystone user-create --name cinder --pass redhat
    +----------+----------------------------------+
    | Property |              Value               |
    +----------+----------------------------------+
    |  email   |                                  |
    | enabled  |               True               |
    |    id    | b8b51e3a3d884723989c1786a7950a9e |
    |   name   |              cinder              |
    | tenantId |                                  |
    +----------+----------------------------------+
    
    
Add an 'admin' role, and associate the user with it::

    $ keystone user-role-add --user cinder \
      --role admin --tenant services

    $ yum install openstack-cinder -y


Configure Cinder
~~~~~~~~~~~~~~~~

Initialize mysql database::

    $ openstack-db --init --service cinder
    Please enter the password for the 'root' MySQL user: 
    Verified connectivity to MySQL.
    Creating 'cinder' database.
    Initializing the cinder database, please wait...
    Complete!


Update cinder.conf with Keystone credentail data for Cinder::

    $ openstack-config --set /etc/cinder/cinder.conf \
      DEFAULT auth_strategy keystone
    $ openstack-config --set /etc/cinder/cinder.conf \
      keystone_authtoken admin_tenant_name services
    $ openstack-config --set /etc/cinder/cinder.conf \
      keystone_authtoken admin_user cinder
    $ openstack-config --set /etc/cinder/cinder.conf \
      keystone_authtoken admin_password redhat


Update QPID configuration for Cinder::

    $ openstack-config --set /etc/cinder/cinder.conf \
      DEFAULT qpid_hostname 192.168.122.126
    $ openstack-config --set /etc/cinder/cinder.conf \
      DEFAULT qpid_port 5672


Cinder Volumes
~~~~~~~~~~~~~~

Create::

    $ dd if=/dev/zero of=/cinder-volumes bs=1 count=0 seek=5G
    0+0 records in
    0+0 records out
    0 bytes (0 B) copied, 0.000237208 s, 0.0 kB/s
    

Setup loop device::

    $ losetup -fv /cinder-volumes 


List the loop devices::

    $ losetup -l
    NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE
    /dev/loop0         0      0         0  0 /cinder-volumes


Create volume group::

    $ vgcreate cinder-volumes /dev/loop0
      WARNING: Failed to connect to lvmetad: No such file or directory. Falling back to internal scanning.
      No physical volume label read from /dev/loop0
      Physical volume "/dev/loop0" successfully created
      Volume group "cinder-volumes" successfully created


Enable and start the LVM2 service::

    $ systemctl enable lvm2-lvmetad.service
    $ systemctl start lvm2-lvmetad.service
    $ systemctl status lvm2-lvmetad.service

Display the volume group::

    $ vgdisplay cinder-volumes
      WARNING: Failed to connect to lvmetad: No such file or directory. Falling back to internal scanning.
      --- Volume group ---
      VG Name               cinder-volumes
      System ID             
      Format                lvm2
      Metadata Areas        1
      Metadata Sequence No  1
      VG Access             read/write
      VG Status             resizable
      MAX LV                0
      Cur LV                0
      Open LV               0
      Max PV                0
      Cur PV                1
      Act PV                1
      VG Size               5.00 GiB
      PE Size               4.00 MiB
      Total PE              1279
      Alloc PE / Size       0 / 0   
      Free  PE / Size       1279 / 5.00 GiB
      VG UUID               ORCGXI-pjJc-ORFG-l9bE-Iy2y-DTyb-EgIm2E


Update ISCSI targets.conf with Cinder volume config info, start the
service, and enable it::

    $ echo "include /etc/cinder/volumes/*" >> /etc/tgt/targets.conf
    $ systemctl start tgtd.service
    $ systemctl status tgtd.service
    $ systemctl enable tgtd.service


Start Cinder-{api,scheduler,volume} services and check their status (to
ensure they're running
::

    $ systemctl start openstack-cinder-api.service
    $ systemctl status openstack-cinder-api.service
    $ systemctl start openstack-cinder-scheduler.service
    $ systemctl status openstack-cinder-scheduler.service
    $ systemctl start openstack-cinder-volume.service


Enable the above service::

    $ systemctl enable openstack-cinder-api.service
    $ systemctl enable openstack-cinder-scheduler.service
    $ systemctl enable openstack-cinder-volume.service


Integrate Cinder with Keystone
------------------------------

Create Cinder service::

    $ keystone service-create --name cinder \
      --type volume --description "Cinder Volume Service"
    +-------------+----------------------------------+
    |   Property  |              Value               |
    +-------------+----------------------------------+
    | description |      Cinder Volume Service       |
    |      id     | 4c1528ccee0340d8be5945a31dba4474 |
    |     name    |              cinder              |
    |     type    |              volume              |
    +-------------+----------------------------------+


Create Cinder end point::

    $ keystone endpoint-create --service_id 4c1528ccee0340d8be5945a31dba4474 \
    --publicurl "http://192.168.122.126:8776/v1/\$(tenant_id)s" \
    --adminurl "http://192.168.122.126:8776/v1/\$(tenant_id)s" \
    --internalurl "http://192.168.122.126:8776/v1/\$(tenant_id)s"
    +-------------+----------------------------------------------+
    |   Property  |                    Value                     |
    +-------------+----------------------------------------------+
    |   adminurl  | http://192.168.122.126:8776/v1/$(tenant_id)s |
    |      id     |       0bcbf485e60c4851ba73dbb6b1f841c8       |
    | internalurl | http://192.168.122.126:8776/v1/$(tenant_id)s |
    |  publicurl  | http://192.168.122.126:8776/v1/$(tenant_id)s |
    |    region   |                  regionOne                   |
    |  service_id |       4c1528ccee0340d8be5945a31dba4474       |
    +-------------+----------------------------------------------+


Test Cinder
-----------

Create a test volume::

    $ cinder create --display-name testvol 1
    +---------------------+--------------------------------------+
    |       Property      |                Value                 |
    +---------------------+--------------------------------------+
    |     attachments     |                  []                  |
    |  availability_zone  |                 nova                 |
    |       bootable      |                False                 |
    |      created_at     |      2013-07-18T06:22:54.455270      |
    | display_description |                 None                 |
    |     display_name    |               testvol                |
    |          id         | 1ba76a69-4a37-4715-958d-208f0764c437 |
    |       metadata      |                  {}                  |
    |         size        |                  1                   |
    |     snapshot_id     |                 None                 |
    |     source_volid    |                 None                 |
    |        status       |               creating               |
    |     volume_type     |                 None                 |
    +---------------------+--------------------------------------+


List logical volumes::

    $ lvs | grep cinder-volumes
    volume-1ba76a69-4a37-4715-958d-208f0764c437 cinder-volumes -wi-ao--- 1.00g


[5] Setup Quamtum
-----------------

Install
~~~~~~~

Packages::

    # FIXME: The below --skip-broken is just a workaround.
    $ yum install python-neutronclient \
      openstack-quantum openstack-quantum-openvswitch \
      --skip-broken -y
   

Start and enable the Open vSwitch (OVS) service::

    $ systemctl start openvswitch.service
    $ systemctl status openvswitch.service


Add an Open vSwitch bridge, and list the contents of OVS database::

    $ ovs-vsctl add-br br-int
    $ ovs-vsctl show
    f7ab97ce-df69-480f-8471-f367e2aa4bbc
        Bridge br-int
            Port br-int
                Interface br-int
                    type: internal
        ovs_version: "1.10.0"


Before proceeding further, add a network card to the controller node:

    $ virsh attach-interface --domain fedostk --type bridge 
      --source virbr1  --model virtio --mac 52:54:00:4e:72:6e 


Add an interface file::

    $ cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-eth1
    DEVICE=eth1
    BOOTPROTO=static
    NM_CONTROLLED=no
    ONBOOT=yes
    TYPE=Ethernet
    EOF

    $ ifup eth1

    $ ovs-vsctl add-br br-eth1

    $ ovs-vsctl show
    f7ab97ce-df69-480f-8471-f367e2aa4bbc
        Bridge "br-eth1"
            Port "br-eth1"
                Interface "br-eth1"
                    type: internal
        Bridge br-int
            Port br-int
                Interface br-int
                    type: internal
        ovs_version: "1.10.0"


    $ ovs-vsctl add-port br-eth1 eth1

    $ ovs-vsctl show
    f7ab97ce-df69-480f-8471-f367e2aa4bbc
        Bridge "br-eth1"
            Port "br-eth1"
                Interface "br-eth1"
                    type: internal
            Port "eth1"
                Interface "eth1"
        Bridge br-int
            Port br-int
                Interface br-int
                    type: internal
        ovs_version: "1.10.0"


Add an external network bridge::

    $ cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-br-ex
    DEVICE=br-ex
    BOOTPROTO=static
    ONBOOT=yes
    IPADDR=192.168.122.126
    NETMASK=255.255.255.0
    GATEWAY=192.168.122.1
    EOF


Take backup of eth0::

    $ cp /etc/sysconfig/network-scripts/ifcfg-eth0 \
      /root/ifcfg-eth0-backup


Unconfigure eth0::

    $ cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
    DEVICE=eth0
    TYPE=Ethernet
    ONBOOT="yes"
    NM_CONTROLLED=no
    BOOTPROTO=static
    EOF

Create external bridge and attach it to eth0::

    $ ovs-vsctl add-br br-ex
    [265369.820293] device br-ex entered promiscuous mode

    $ ovs-vsctl add-port br-ex eth0
    [265392.657220] device eth0 entered promiscuous mode


Display OVS database contents::

    $ ovs-vsctl show
    f7ab97ce-df69-480f-8471-f367e2aa4bbc
        Bridge br-ex
            Port br-ex
                Interface br-ex
                    type: internal
            Port "eth0"
                Interface "eth0"
        Bridge "br-eth1"
            Port "br-eth1"
                Interface "br-eth1"
                    type: internal
            Port "eth1"
                Interface "eth1"
        Bridge br-int
            Port br-int
                Interface br-int
                    type: internal
        ovs_version: "1.10.0"

Restart networking::

    $ systemctl stop NetworkManager

    $ systemctl restart network

    $ systemctl status network.service
    network.service - LSB: Bring up/down networking
       Loaded: loaded (/etc/rc.d/init.d/network)
       Active: active (exited) since Sat 2013-07-20 11:45:25 EDT; 1min 11s ago
      Process: 23589 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)
    
    Jul 20 11:45:22 fedostk network[23589]: Bringing up loopback interface:  [  ...]
    Jul 20 11:45:23 fedostk network[23589]: Bringing up interface br-ex:  [  OK  ]
    Jul 20 11:45:24 fedostk network[23589]: Bringing up interface eth0:  [  OK  ]
    Jul 20 11:45:25 fedostk network[23589]: Bringing up interface eth1:  [  OK  ]
    Jul 20 11:45:25 fedostk systemd[1]: Started LSB: Bring up/down networking.


    $ service network status
    Configured devices:
    lo br-ex eth0 eth1
    Currently active devices:
    lo eth0 br-int eth1 br-eth1 br-ex



Finally, the br-ex interface should have the IP address::

    $ ifconfig br-ex
    br-ex: flags=67<UP,BROADCAST,RUNNING>  mtu 1500
            inet 192.168.122.126  netmask 255.255.255.0  broadcast 192.168.122.255
            inet6 fe80::f47b:7cff:fe56:7949  prefixlen 64  scopeid 0x20<link>
            ether f6:7b:7c:56:79:49  txqueuelen 0  (Ethernet)
            RX packets 69564  bytes 3602422 (3.4 MiB)
            RX errors 0  dropped 67886  overruns 0  frame 0
            TX packets 32  bytes 2336 (2.2 KiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    $ route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.122.1   0.0.0.0         UG    0      0        0 br-ex
    169.254.0.0     0.0.0.0         255.255.0.0     U     1002   0        0 eth0
    169.254.0.0     0.0.0.0         255.255.0.0     U     1006   0        0 eth1
    169.254.0.0     0.0.0.0         255.255.0.0     U     1008   0        0 br-ex
    192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 br-ex

