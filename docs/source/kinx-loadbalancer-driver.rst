KINX-loadbalancer-driver
========================

What is KINX loadbalancer driver?
---------------------------------

KINX loadbalancer driver is Openstack neutron lbaasv2 driver which create VM-based reverse proxy & loadbalancer.

KINX Loadbalancer Driver Architecture
-------------------------------------

.. image:: images/klb_driver.png

Work Flow
---------

* Create **Loadbalancer**

  #. Create Security Group & Rule
  #. Create Server Group for seperation of VM server to different host
  #. Create Two Loadbalancer VM (Active-Active)
  #. Request storing virtual-loadbalancer information to vlb-api server (VLB ID, etc...)
  #. Wait until VM is active status
  #. Create Ceilometer Alarm (PPS Threshold)
  #. Request storing created VMs information to vlb-api server (VM ID, Alarm ID, etc...)
  #. Update Port Address Pair with VIP
  #. Request configuration of loadbalancer to created VMs (haproxy.cfg)
  #. Request configuration of peer between created VMs (haproxy.cfg)
  #. Request configuration of quagga for ECMP loadbalancing
  #. Request configuration of crontab that periodically send VMs Health Information

* Create **Listener**

  #. Update Security Group Rule that user requests such as port
  #. Recieve loadbalancer VMs information(LB-MGMT IP) from vlb-api databases
  #. Request configuration of listener to loadbalancer VMs

* Create **Pool**

  #. Recieve loadbalancer VMs information(LB-MGMT IP) from vlb-api databases
  #. Request configuration of pool to loadbalancer VMs

* Create **Member**

  #. Recieve loadbalancer VMs information(LB-MGMT IP) from vlb-api databases
  #. Request configuration of member to loadbalancer VMs

* Create **Healthmonitor**

  #. Recieve loadbalancer VMs information(LB-MGMT IP) from vlb-api databases
  #. Request configuration of healthmonitor to loadbalancer VMs

Pre-Installation
----------------

#. vLB MGMT Network creation(vLB VM <--> vLB Controller)::
    # Interface Configuration (VLAN 680)
    $ vim ifcfg-bond0.680
    auto bond0.680
    iface bond0.680 inet manual
    mtu 9000
    vlan-raw-device bond0

    $ vim ifcfg-br-lb-mgmt
    auto br-lb-mgmt
    iface br-lb-mgmt inet static
    bridge_ports bond0.680
    address 10.30.252.4/22

    # Interface & Bridge UP
    $ ifup bond0.680
    Set name-type for VLAN subsystem. Should be visible in /proc/net/vlan/config
    ifup: recursion detected for interface bond0 in parent-lock phase
    Added VLAN with VID == 680 to IF -:bond0:-
    ntp stop/waiting
    ntp stop/pre-start, process 10172

    $ ifup br-lb-mgmt
    ifup: waiting for lock on /run/network/ifstate.br-lb-mgmt
    ifup: interface br-lb-mgmt already configured

    # Create vLB MGMT Provider Network
    $ neutron net-create --shared --provider:physical_network physnet2 --provider:network_type vlan --provider:segmentation_id 680 lb-mgmt-net
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        |                                      |
    | created_at                | 2018-08-07T08:08:05                  |
    | description               |                                      |
    | dns_domain                |                                      |
    | id                        | 0589ee28-9e73-42ef-ba58-41e5b32a85ba |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 9000                                 |
    | name                      | lb-mgmt-net                          |
    | port_security_enabled     | True                                 |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet2                             |
    | provider:segmentation_id  | 680                                  |
    | qos_policy_id             |                                      |
    | router:external           | False                                |
    | shared                    | True                                 |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tags                      |                                      |
    | tenant_id                 | cc01d654476e402dbec197ffa7a8d8cf     |
    | updated_at                | 2018-08-07T08:08:05                  |
    +---------------------------+--------------------------------------+

    $ neutron subnet-create --name lb-mgmt-subnet --allocation-pool start=10.30.252.11,end=10.30.255.254 --no-gateway --dns-nameserver 8.8.8.8 0589ee28-9e73-42ef-ba58-41e5b32a85ba 10.30.252.0/22
    Created a new subnet:
    +-------------------+---------------------------------------------------+
    | Field             | Value                                             |
    +-------------------+---------------------------------------------------+
    | allocation_pools  | {"start": "10.30.252.11", "end": "10.30.255.254"} |
    | cidr              | 10.30.252.0/22                                    |
    | created_at        | 2018-08-07T08:18:36                               |
    | description       |                                                   |
    | dns_nameservers   | 8.8.8.8                                           |
    | enable_dhcp       | True                                              |
    | gateway_ip        |                                                   |
    | host_routes       |                                                   |
    | id                | 2ec6dabe-6094-4c45-8d17-37781ea3d4db              |
    | ip_version        | 4                                                 |
    | ipv6_address_mode |                                                   |
    | ipv6_ra_mode      |                                                   |
    | name              | lb-mgmt-subnet                                    |
    | network_id        | 0589ee28-9e73-42ef-ba58-41e5b32a85ba              |
    | subnetpool_id     |                                                   |
    | tenant_id         | cc01d654476e402dbec197ffa7a8d8cf                  |
    | updated_at        | 2018-08-07T08:18:36                               |
    +-------------------+---------------------------------------------------+
    ```

    $ iptables -A INPUT -s 10.30.252.0/22 -p tcp --dport {vlbapi port} -j ACCEPT

#. vLB L3 Network Creation::
    $ neutron net-create --shared --provider:physical_network physnet2 --provider:network_type vlan --provider:segmentation_id 660 lb-l3-net
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        |                                      |
    | created_at                | 2018-08-09T00:42:07                  |
    | description               |                                      |
    | dns_domain                |                                      |
    | id                        | bb0d0a66-e039-4b24-a4ba-2b27f1ad1169 |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 9000                                 |
    | name                      | lb-l3-net                            |
    | port_security_enabled     | True                                 |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet2                             |
    | provider:segmentation_id  | 660                                  |
    | qos_policy_id             |                                      |
    | router:external           | False                                |
    | shared                    | True                                 |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tags                      |                                      |
    | tenant_id                 | cc01d654476e402dbec197ffa7a8d8cf     |
    | updated_at                | 2018-08-09T00:42:07                  |
    +---------------------------+--------------------------------------+

    $ neutron subnet-create --name lb-l3-subnet --allocation-pool start=10.40.252.11,end=10.40.255.254 --gateway 10.40.252.1 --dns-nameserver 8.8.8.8 bb0d0a66-e039-4b24-a4ba-2b27f1ad1169 10.40.252.0/22
    Created a new subnet:
    +-------------------+---------------------------------------------------+
    | Field             | Value                                             |
    +-------------------+---------------------------------------------------+
    | allocation_pools  | {"start": "10.40.252.11", "end": "10.40.255.254"} |
    | cidr              | 10.40.252.0/22                                    |
    | created_at        | 2018-08-09T00:46:26                               |
    | description       |                                                   |
    | dns_nameservers   | 8.8.8.8                                           |
    | enable_dhcp       | True                                              |
    | gateway_ip        | 10.40.252.1                                       |
    | host_routes       |                                                   |
    | id                | e99180c0-be82-44c6-b387-b425973bd73b              |
    | ip_version        | 4                                                 |
    | ipv6_address_mode |                                                   |
    | ipv6_ra_mode      |                                                   |
    | name              | lb-l3-subnet                                      |
    | network_id        | bb0d0a66-e039-4b24-a4ba-2b27f1ad1169              |
    | subnetpool_id     |                                                   |
    | tenant_id         | cc01d654476e402dbec197ffa7a8d8cf                  |
    | updated_at        | 2018-08-09T00:46:26                               |
    +-------------------+---------------------------------------------------+

Installation
------------

#. Clone kinx-loadbalancer github repository::

    $ git clone https://github.com/kinxnet/kinx-loadbalancer.git

#. Copy kinx-loadbalancer driver to Openstack neutron_lbaas driver::

    $ cp -rf kinx-loadbalancer/kinx_loadbalancer_driver /usr/lib/python2.7/dist-packages/neutron_lbaas/drivers/kinx

#. Create availibility zone for kinx-loadbalancer

#. Add Kinx-loadbalancer configuration to ``/etc/neutron/neutron.conf``::

    [DEFAULT]
    dhcp_agents_per_network=3

    [service_providers]
    service_provider=LOADBALANCERV2:Kinx:neutron_lbaas.drivers.kinx_driver.driver.KinxDriver:default

    [kinx_haproxy]
    image_id=7de0af35-14d3-4771-8af5-48cbce75ea1d # Image ID
    base_flavor_id=880a8c79-6967-4c27-8ef7-1fe092bbeeda # Flavor ID
    auth_url=http://192.168.0.2:35357/v2.0/
    endpoint_url=http://192.168.0.2:9696/
    kinx_office_ip=211.196.205.71/32
    agent_user=kinx_haproxy_user
    agent_password=ee71f1d37d8c4f96ac4fbb5ebecd65a9
    agent_port=62000
    default_maxconn=12000
    loadbalancer_instance_volume_size=50
    availibility_zone=lbaas-az
    lb_api_agent_addr=192.168.0.4
    lb_api_agent_port=6543
    lb_mgmt_subnet=10.30.252.0/22
    lb_mgmt_net_id=0589ee28-9e73-42ef-ba58-41e5b32a85ba
    lb_mgmt_net_name=lb-mgmt-net
    lb_l3_subnet=10.40.252.0/22
    lb_l3_net_id=bb0d0a66-e039-4b24-a4ba-2b27f1ad1169
    lb_l3_net_name=lb-l3-net
    base_as_number=10000
    primary_l3_gateway_addr=10.40.252.2
    secondary_l3_gateway_addr=10.40.252.3
    l3_router_as_number=60201
    ceilometer_period=60
    ceilometer_evaluation_periods=3
    ceilometer_pps_threshold=20000

#. Restart neutron server::

    $ service neutron-server restart
