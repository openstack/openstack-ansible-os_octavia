Basic Octavia Load Balancer Troubleshooting Guide
=================================================

This guide provides a quick and practical walkthrough for creating and troubleshooting
a basic HTTP load balancer using OpenStack Octavia.

Scenario Overview
-----------------

We assume:

#. Two backend instances exist in subnet ``private-subnet-1 (10.0.2.0/24)``

#. Instances are reachable on port 80 (e.g., simple web servers running)

#. The subnet is connected to a router (required for Floating IP access)

Creating Load Balancer
----------------------

#. Create the load balancer:

   .. code-block:: console

      # openstack loadbalancer create --name load-balancer-1 --vip-subnet-id private-subnet-1

#. Create listener:

   .. code-block:: console

      # openstack loadbalancer listener create \
           --name listener-1 \
           --protocol HTTP \
           --protocol-port 80 \
           load-balancer-1

#. Create pool (using Round Robin algorithm):

   .. code-block:: console

      # openstack loadbalancer pool create \
           --name pool-1 \
           --lb-algorithm ROUND_ROBIN \
           --listener listener-1 \
           --protocol HTTP

#. Add members:

   .. code-block:: console

      # openstack loadbalancer member create \
           --subnet-id private-subnet-1 \
           --address 10.0.2.252 \
           --protocol-port 80 \
           pool-1

      # openstack loadbalancer member create \
           --subnet-id private-subnet-1 \
           --address 10.0.2.57 \
           --protocol-port 80 \
           pool-1

Assign Floating IP to VIP
-------------------------

Ensure your subnet is connected to a router, then assign a Floating IP:

.. code-block:: console

    # openstack floating ip set --port <load_balancer_vip_port_id> <floating_ip_id>

Verification in Horizon / CLI
-----------------------------

After successful creation, two Amphora instances will be created (ACTIVE_STANDBY setup):

- MASTER
- BACKUP

You can view them via CLI:

.. code-block:: console

    # openstack loadbalancer amphora list

Typical output looks like:

.. code-block:: console

   +--------------------------------------+--------------------------------------+-----------+--------+----------------+-----------+
   | id                                   | loadbalancer_id                      | status    | role   | lb_network_ip  | ha_ip     |
   +--------------------------------------+--------------------------------------+-----------+--------+----------------+-----------+
   | 3b42cd08-564f-4415-b7cb-9c6e3920f2d4 | c66e66c9-9904-4a07-bc88-3eaa24d2c3ef | ALLOCATED | MASTER | 172.29.232.161 | 10.0.2.64 |
   | 62172a88-08d4-4a12-8a9a-4721d66e1544 | c66e66c9-9904-4a07-bc88-3eaa24d2c3ef | ALLOCATED | BACKUP | 172.29.233.104 | 10.0.2.64 |
   +--------------------------------------+--------------------------------------+-----------+--------+----------------+-----------+

- ``172.29.232.0/22`` -> management network (lb-mgmt-net)
- ``10.0.2.64`` -> VIP address

Connecting to Amphora (Master)
------------------------------

Ensure:

- SSH access is enabled
- You have connectivity to the management network

.. code-block:: console

   # ssh -i .ssh/octavia_key ubuntu@172.29.232.161

Inspect Network Configuration
-----------------------------

Default namespace (management interface):

.. code-block:: console

   ubuntu@amphora:~$ ip a

Sample output:

.. code-block:: console

    ens3: <BROADCAST,MULTICAST,UP,LOWER_UP>
        inet 172.29.232.161/22

This interface belongs to the **management network**.

Network Namespaces
------------------

Octavia uses a dedicated namespace for HAProxy:

.. code-block:: console

   ubuntu@amphora:~$ sudo ip netns list
   amphora-haproxy (id: 0)

Inspect interfaces inside the namespace:

.. code-block:: console

   ubuntu@amphora:~$ sudo ip netns exec amphora-haproxy ip a

Sample output:

.. code-block:: console

   eth1: <BROADCAST,MULTICAST,UP,LOWER_UP>
         inet 10.0.2.69/24
         inet 10.0.2.64/32

- ``10.0.2.64`` -> VIP address (active on MASTER)
- ``10.0.2.69`` -> Amphora instance IP

The VIP is configured as a /32 and moves between MASTER/BACKUP during failover.

HAProxy Service Inspection
--------------------------

Each apmhora instance runs its own HAProxy service.

#. Check service status:

   .. code-block:: console

      ubuntu@amphora:~$ sudo systemctl status haproxy-<load-balancer-id>

#. Check configuration:

   .. code-block:: console

      ubuntu@amphora:~$ sudo cat /var/lib/octavia/<load-balancer-id>/haproxy.cfg

   Configuration snippet:

   .. code-block:: console

       peers c66e66c999044a07bc883eaa24d2c3ef_peers
           peer cbZyp_Dj-MpsKs2NJeavavY7dbU 10.0.2.69:1025
           peer 9PCRPduKu_p18PXO49ZHWE1bApI 10.0.2.200:1025

       frontend 265a327c-4486-484a-a471-8fdd597c1911
           bind 10.0.2.64:80
           mode http
           default_backend a972f1f9-9f9f-4b48-af21-276f3e0d22f3:265a327c-4486-484a-a471-8fdd597c1911

       backend a972f1f9-9f9f-4b48-af21-276f3e0d22f3:265a327c-4486-484a-a471-8fdd597c1911
           balance roundrobin
           server efa07206-9039-42fe-87a8-6dd57818b1e2 10.0.2.252:80 weight 1
           server 74c9b591-b816-4cf7-8e87-efa76ab5bcaf 10.0.2.57:80 weight 1

   Key lines:

   - ``peers ...`` -> defines synchronization between Amphora nodes for HA state sharing
   - ``peer <id> <ip>:1025`` -> remote Amphora nodes participating in sync
   - ``bind 10.0.2.64:80`` -> HAProxy listens on VIP address
   - ``mode http`` -> layer 7 load balancing
   - ``default_backend ...`` -> directs traffic to backend pool
   - ``balance roundrobin`` -> distributes requests evenly across members
   - ``server <id> <ip>:80`` -> backend instances receiving traffic
   - ``weight 1`` -> equal traffic distribution

Common Troubleshooting Checks
-----------------------------

1. VIP is not reachable
~~~~~~~~~~~~~~~~~~~~~~~

- Check Floating IP assignment
- Verify router connectivity
- Confirm security groups allow port 80

2. No traffic to backend
~~~~~~~~~~~~~~~~~~~~~~~~

- Verify backend instances are running web servers
- Test directly:

  .. code-block:: console

     ubuntu@amphora:~$ curl -I http://10.0.2.252
     ubuntu@amphora:~$ curl -I http://10.0.2.57

3. HAProxy not running
~~~~~~~~~~~~~~~~~~~~~~

- Check service:

  .. code-block:: console

     ubuntu@amphora:~$ sudo systemctl status haproxy-<load-balancer-id>

4. Wrong IP binding
~~~~~~~~~~~~~~~~~~~

- Ensure VIP exists in namespace:

  .. code-block:: console

     ubuntu@amphora:~$ sudo ip netns exec amphora-haproxy ip a

This is a **basic Octavia troubleshooting workflow** which serves as a starting point
for debugging load balancer issues in OpenStack environments.
