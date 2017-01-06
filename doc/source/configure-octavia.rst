=========================================================
Configuring the Octavia Load Balancing service (optional)
=========================================================

.. note::

   This feature is experimental at this time and it has not been fully
   production tested yet.

Octavia is an OpenStack project which provides operator-grade Load Balancing
(as opposed to the namespace driver) by deploying each individual load
balancer to its own virtual machine and leveraging haproxy to perform the
load balancing.

Octavia is scalable and has built-in high availability through active-passive.

OpenStack-Ansible deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Create the openstack-ansible container(s) for Octavia
#. Run the os-octavia playbook
#. Eventually the os-neutron playbook needs to be rerun.

Setup a neutron network for use by octavia
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In a general case, neutron networking can be a simple flat network. However,
in a complex case, this can be whatever you need and want. Ensure
you adjust the deployment accordingly. The following is an example:


.. code-block:: bash

    neutron net-create cleaning-net --shared \
                                    --provider:network_type flat \
                                    --provider:physical_network mgmt

    neutron subnet-create ironic-net 172.19.0.0/22 --name mgmt-subnet
                          --ip-version=4 \
                          --allocation-pool start=172.19.1.100,end=172.19.1.200 \
                          --enable-dhcp \
                          --dns-nameservers list=true 8.8.4.4 8.8.8.8

Building Octavia images
~~~~~~~~~~~~~~~~~~~~~~~

Images using the ``diskimage-builder`` must be built outside of a container.
For this process, use one of the physical hosts within the environment.

#. Install the necessary packages:

   .. code-block:: bash

      apt-get install -y qemu uuid-runtime curl kpartx git jq python-pip

#. Install the necessary pip packages:

   .. code-block:: bash

     pip install argparse Babel>=1.3 dib-utils PyYAML

#. Clone the necessary repositories

   .. code-block:: bash

     git clone https://github.com/openstack/octavia.git
     git clone https://git.openstack.org/openstack/diskimage-builder.git


#. Run Octavia's diskimage script

   In the ``octavia/diskimage-create`` directory run:
   .. code-block:: bash

     ./diskimage-create.sh


#. Upload the created user images into the Image (glance) Service:

   .. code-block:: bash

      glance image-create --name amphora-x64-haproxy --visibility private --disk-format qcow2 \
         --container-format bare --tags octavia-amphora-image </var/lib/octavia/amphora-x64-haproxy.qcow2

You can find more information abpout the diskimage script and the process at
https://github.com/openstack/octavia/tree/master/diskimage-create

Here is a script to perform all those tasks at once:
   .. code-block:: bash

          #/bin/sh
          apt-get install -y qemu uuid-runtime curl kpartx git jq
          pip -v >/dev/null || {apt-get install -y python-pip}
          pip install argparse Babel>=1.3 dib-utils PyYAML
          pushd /tmp
          git clone https://github.com/openstack/octavia.git
          git clone https://git.openstack.org/openstack/diskimage-builder.git
          pushd  octavia/diskimage-create
          ./diskimage-create.sh
          mv amphora-x64-haproxy.qcow2 /tmp
          popd
          popd
          #upload image
          glance image-create --name amphora-x64-haproxy --visibility private --disk-format qcow2 \
            --container-format bare --tags octavia-amphora-image </var/lib/octavia/amphora-x64-haproxy.qcow2

.. note::
    If you have trouble installing dib-utils from pipy consider installing it directly from souce
    ` pip install git+https://github.com/openstack/dib-utils.git`

Creating the cryptographic certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
    For production installation make sure that you review this very carefully with your
    own security requirements and potantially use your own CA to sign the certificates.

#. Run the certificate script.

   In the bin directory of the Octavia project you cloned above run:
   .. code-block:: bash

      mkdir /var/lib/octavia/certs
      source create_certificates.sh /var/lib/octavia/certs `pwd`/../etc/certificates/openssl.cnf

.. note::
   The certificates will be created in ``/var/lib/octavia/certs`` where the
   ansible script are expecting them.

Optional: Configuring Octavia with ssh access to the amphora
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In rare cases it might be beneficial to gain ssh access to the
amphora for additional trouble shooting. Follow these steps to
enable access.

#. Create an ssh key

   .. code-block:: bash

      ssh-keygen

#. Upoad the key into nova as the *octavia* user:

   .. code-block:: bash

     openstack keypair create --public-key <public key file> octavia_key

   .. note::
      To find the octavia user's username and credentials review
      the octavia-config file
      on any octavia container in /etc/octavia.

#. Configure Octavia accordingly

   Add a ``octavia_ssh_enabled: True`` to the user file in
   /etc/openstack-deploy


Optional: Tuning Octavia for production use
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Please have a close look at the ``main.yml`` for tunable parameters.
The most important change is to set Octavia into ACTIVE_STANDBY mode
by adding ``octavia_loadbalancer_topology: ACTIVE_STANDBY`` to the
user file in /etc/openstack-deploy

To speed up the creation of load balancers or in a SINGLE topolgy
to speed up the failover a spare pool can be used.
The variable ``octavia_spare_amphora_pool_size`` controls
the size of the pool. The system will try
to prebuild this number so using too big a number will
consumes a lot of unnecessary resources.
