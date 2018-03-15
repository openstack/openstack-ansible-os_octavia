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

Octavia needs connectivity between the control plane and the
load balancing VMs. For this purpose a provider network should be
created which bridges the octavia containers (if the control plane is installed
in a container) or hosts with VMs. Refer to the appropriate documentation
and consult the tests in this project. In a general case, neutron networking
can be a simple flat network. However in a complex case, this can be whatever
you need and want. Ensure you adjust the deployment accordingly. An example
entry into ``openstack_user_config.yml`` is shown below:

.. code-block:: yaml

     - network:
        container_bridge: "br-lbaas"
        container_type: "veth"
        container_interface: "eth14"
        host_bind_override: "eth14"
        ip_from_q: "octavia"
        type: "flat"
        net_name: "octavia"
        group_binds:
          - neutron_linuxbridge_agent
          - octavia-worker
          - octavia-housekeeping
          - octavia-health-manager

Make sure to modify the other entries in this file as well.

There are a couple of variables which need to be adjusted if you don't use
``lbaas`` for the provider network name and ``lbaas-mgmt`` for the neutron
name. Furthermore, the system tries to infer certain values based on the
inventory which might not always work and hence might need to be explicitly
declared. Review the file ``defaults/main.yml`` for more information.

Octavia can create the required neutron networks itself. Please review the
corresponding settings - especially ``octavia_management_net_subnet_cidr``
needs to be adjusted. Alternatively, they can be created elsewhere and
consumed by Octavia.

Special attention needs to be applied to the ``--allocation-pool`` to not have
ips which overlap with ips assigned to hosts or containers (see the
``used_ips`` variable in ``openstack_user_config.yml``)

.. note::
    The system will deploy an iptables firewall if ``octavia_ip_tables_fw`` is set
    to ``True`` (the default). This adds additional protection to the control plane
    in the rare instance a load balancing vm is compromised. Please review carefully
    the rules and adjust them for your installation. Please be aware that logging
    of dropped packages is not enabled and you will need to add those rules manually.

Building Octavia images
~~~~~~~~~~~~~~~~~~~~~~~

.. note::
    The default behavior is to download a test image from the OpenStack artifact
    storage the Octavia team provides daily. Because this image doesn't apply
    operating system security patches in a timely manner it is unsuited
    for production use.

    Some Operating System vendors might provide official amphora builds or an
    organization might maintain their own artifact storage - for those cases the
    automatic download can be leveraged, too.

Images using the ``diskimage-builder`` must be built outside of a container.
For this process, use one of the physical hosts within the environment.

#. Install the necessary packages:

   .. code-block:: bash

      apt-get install qemu uuid-runtime curl kpartx git jq python-pip

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

   .. note::
        Alternatively you can specify the new image in the appropriate settings and rerun the
        ansible with an appropriate tag.

You can find more information abpout the diskimage script and the process at
https://github.com/openstack/octavia/tree/master/diskimage-create

Here is a script to perform all those tasks at once:

   .. code-block:: bash

          #/bin/sh
          apt-get install qemu uuid-runtime curl kpartx git jq
          pip -v >/dev/null || {apt-get install python-pip}
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
    If you have trouble installing dib-utils from pipy consider installing it directly from source
    ` pip install git+https://github.com/openstack/dib-utils.git`

Creating the cryptographic certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
    For production installation make sure that you review this very carefully with your
    own security requirements and potantially use your own CA to sign the certificates.

The system will automatically generate and use self-signed certificates with different
Certificate Authorities for control plane and amphora. Make sure to store a copy in
a safe place for potential disaster recovery.

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


Optional: Enable Octavia V1 API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Beginning with the Queens release, neutron lbaas has started it's
deprecation cycle and therefore Octavia stand alone is the default
configuration beginning Rocky. If the neutron lbaas endpoint is still
needed consider deploying the neutron lbaas proxy plugin.

If for legacy reasons neutron lbaas still needs the internal Octavia V1
API endpoint add the following parameters to ``openstack_user_config.yml``:

.. code-block:: yaml

  # Enable Octavia support in Neutron
  neutron_lbaas_octavia: True
  # Enable LBaaS V2
  neutron_lbaasv2: True
  # Disable Octavia V2 API/standalone
  octavia_v2: False
  # Enable Octavia V1 API
  octavia_v1: True
  # event_streamer - set to True if you are using neutron lbaas with Octavia
  # (Octavia will stream events to the neutron DB)
  octavia_event_streamer: True
  # Enable provisioning status sync with neutron db
  octavia_sync_provisioning_status: True

Please note that in some settings the LBaaS plugin is directly enabled in the
``neutron_plugin_base`` so adjust this as necessary.

Optional: Tuning Octavia for production use
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Please have a close look at the ``main.yml`` for tunable parameters.
The most important change is to set Octavia into ACTIVE_STANDBY mode
by adding ``octavia_loadbalancer_topology: ACTIVE_STANDBY`` and
``octavia_enable_anti_affinity=True`` to ensure that the active and passive
amphora are (depending on the anti-affinity filter deployed in nova)  on two
different hosts to the user file in /etc/openstack-deploy

To speed up the creation of load balancers or in a SINGLE topolgy
to speed up the failover a spare pool can be used.
The variable ``octavia_spare_amphora_pool_size`` controls
the size of the pool. The system will try
to prebuild this number so using too big a number will
consumes a lot of unnecessary resources.

