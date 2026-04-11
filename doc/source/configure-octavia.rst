==============================================
Configuring the Octavia Load Balancing service
==============================================

Octavia is an OpenStack project which provides operator-grade Load Balancing
(as opposed to the namespace driver) by deploying each individual load
balancer to its own instance and leveraging HAProxy to perform the
load balancing.

Octavia is scalable and has built-in high availability through active-passive.

OpenStack-Ansible deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Create ``br-lbaas`` bridge on the controllers. Creating br-lbaas is done during
   the deployers host preparation and is out of scope of OpenStack-Ansible.
   Some explanation of how br-lbaas is used is given below.

#. Create the OpenStack-Ansible container(s) for Octavia. To do that you need
   to define hosts for ``octavia-infra_hosts`` group in
   ``openstack_user_config.yml``. Once you do this, run the following playbook:

   .. code-block:: console

      # openstack-ansible openstack.osa.containers_lxc_create --limit octavia_all,octavia-infra_hosts

#. Define required overrides of the variables in defaults/main.yml of the
   OpenStack-Ansible Octavia role.

#. Run the OpenStack-Ansible Octavia playbook:

   .. code-block:: console

      # openstack-ansible openstack.osa.octavia

#. Run the HAProxy playbook to add the new Octavia API endpoints to the
   load balancer.

   .. code-block:: console

      # openstack-ansible openstack.osa.haproxy --tags haproxy-service-config

#. In order to enable Octavia dashboard panel in your Horizon dashboard, run
   Horizon playbook:

   .. code-block:: console

      # openstack-ansible openstack.osa.horizon

Define project quota for Amphora driver
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Amphora driver for Octavia is a default option on spawning Load Balancers
with Octavia. The driver relies on OpenStack Nova/Neutron services to spawn
instances from a specialized image which serve as Load Balancers.

These instances are created in a ``service`` project by default, and projects have
no direct access to them.

With that operator must ensure, that the ``service`` project
has sufficient quotas defined to handle all projects Load Balancers in it.

The suggested way of doing that is through leveraging the
``openstack.osa.openstack_resources`` playbook and defining following
variables in ``user_variables.yml`` or ``group_vars/utility_all``:

.. code-block:: yaml

   # In case of `octavia_loadbalancer_topology` set to ACTIVE_STANDBY (default)
   # each Load Balancer will create 2 instances
   _max_amphora_instances: 10000
   openstack_user_identity:
      quotas:
        - name: "service"
          # Default Amphora flavor is 1 Core, 1024MB RAM
          cores: "{{ _max_amphora_instances }}"
          ram: "{{ (_max_amphora_instances | int) * 1024 }}"
          instances: "{{ _max_amphora_instances }}"
          port: "{{ (_max_amphora_instances | int) * 10 }}"
          server_groups: "{{ ((_max_amphora_instances | int) * 0.5) | int | abs }}"
          server_group_members: 50
          # A security group is created per Load Balancer listener
          security_group: "{{ (_max_amphora_instances | int) * 1.5 | int | abs }}"
          security_group_rule: "{{ ((_max_amphora_instances | int) * 1.5 | int | abs) * 100 }}"
          # If `octavia_cinder_enabled: true` also define these
          volumes: "{{ _max_amphora_instances }}"
          # Volume size is defined with `octavia_cinder_volume_size` with default of 20
          gigabytes: "{{ (_max_amphora_instances | int) * 20 }}"

These values will be applied on running ``openstack-ansible openstack.osa.openstack_resources``,
or as part of ``openstack.osa.setup_openstack`` playbook.

.. warning::

   Octavia uses ``ACTIVE_STANDBY`` topology by default
   (``octavia_loadbalancer_topology``) and enables amphora
   anti-affinity (``octavia_enable_anti_affinity: true``).
   Deployments with only one compute host will fail to create
   load balancers.

Setup a neutron network for use by Octavia
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Octavia needs connectivity between the control plane and the
load balancing instances. For this purpose a provider network should be
created which gives L2 connectivity between the octavia services
on the controllers (either containerised or deployed on metal)
and the Octavia amphora instances. Refer to the appropriate documentation
for the octavia service and consult the tests in this project
for a working example.

Special attention needs to be applied to the provider network
``--allocation-pool`` not to have ip addresses which overlap with
those assigned to hosts, lxc containers or other infrastructure such
as routers or firewalls which may be in use.

An example which gives ``172.29.232.0-9/22`` to the OSA dynamic inventory
and the remainder of the addresses to the neutron allocation pool
without overlap is as follows:

In ``openstack_user_config.yml`` the following:

.. code-block:: yaml

   #the address range for the whole lbaas network
   cidr_networks:
      lbaas: 172.29.232.0/22

   #the range of ip addresses excluded from the dynamic inventory
   used_ips:
      - "172.29.232.10,172.29.235.200"

And define in ``user_variables.yml``:

.. code-block:: yaml

   #the range of addresses which neutron can allocate for amphora instances
   octavia_management_net_subnet_allocation_pools: "172.29.232.10-172.29.235.200"

.. note::
    The system will deploy an iptables firewall if ``octavia_ip_tables_fw`` is set
    to ``true`` (the default). This adds additional protection to the control plane
    in the rare instance a load balancing vm is compromised. Please review carefully
    the rules and adjust them for your installation. Please be aware that logging
    of dropped packages is not enabled and you will need to add those rules manually.

FLAT networking scenario
------------------------

In a general case, neutron networking can be a simple flat network. However in
a complex case, this can be whatever you need and want. Ensure you adjust the
deployment accordingly. An example entry into ``openstack_user_config.yml`` is
shown below:

.. code-block:: yaml

     - network:
        container_bridge: "br-lbaas"
        container_type: "veth"
        container_interface: "eth14"
        host_bind_override: "bond0"  # Defines neutron physical network mapping
        ip_from_q: "octavia"
        type: "flat"
        net_name: "octavia"
        group_binds:
          - octavia_worker
          - octavia_housekeeping
          - octavia_health_manager
          # in case of OVN
          - neutron_ovn_gateway
          # in case of OVS
          - neutron_openvswitch_agent

There are a couple of variables which need to be adjusted if you don't use
``lbaas`` for the provider network name and ``lbaas-mgmt`` for the neutron
name. Furthermore, the system tries to infer certain values based on the
inventory which might not always work and hence might need to be explicitly
declared. Review the file ``defaults/main.yml`` for more information.

The Octavia Ansible role can create the required neutron networks itself.
Please review the corresponding settings - especially
``octavia_management_net_subnet_cidr`` should be adjusted to suit your
environment. Alternatively, the neutron network can be pre-created elsewhere
and consumed by Octavia.

VLAN networking scenario
------------------------

In case you want to leverage standard vlan networking for the Octavia
management network the definition in ``openstack_user_config.yml`` may
look like this:

.. code-block:: yaml

    - network:
        container_bridge: "br-lbaas"
        container_type: "veth"
        container_interface: "eth14"
        ip_from_q: "lbaas"
        type: "raw"
        net_name: lbaas
        group_binds:
          - octavia_worker
          - octavia_housekeeping
          - octavia_health_manager
          # in case of OVN
          - neutron_ovn_gateway
          # in case of OVS
          - neutron_openvswitch_agent

Add extend ``user_variables.yml`` with following overrides:

.. code-block:: yaml

   octavia_provider_network_name: vlan
   octavia_provider_network_type: vlan
   octavia_provider_segmentation_id: 400
   octavia_provider_inventory_net_name: lbaas

In addition to this, you will need to ensure that you have an interface that
links neutron-managed br-vlan with br-lbaas on the controller nodes (for the case
when br-vlan already exists on the controllers when they also host the neutron
L3 agent). Making veth pairs or macvlans for this might be suitable.

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

#. Install the necessary packages and configure a Python virtual environment

   .. code-block:: bash

      apt-get install qemu-system uuid-runtime curl kpartx git jq python3-pip
      pip3 install virtualenv

      virtualenv -p /usr/bin/python3 /opt/octavia-image-build
      source /opt/octavia-image-build/bin/activate

#. Clone the necessary repositories and dependencies

   .. code-block:: bash

     # git clone https://opendev.org/openstack/octavia

       /opt/octavia-image-build/bin/pip install --isolated \
       git+https://opendev.org/openstack/diskimage-builder

#. Run Octavia's diskimage script

   In the ``octavia/diskimage-create`` directory run:

   .. code-block:: bash

     ./diskimage-create.sh

   Disable ``octavia-image-build`` venv:

   .. code-block:: bash

      deactivate

#. Upload the created user images into the Image (glance) Service:

   .. code-block:: bash

      openstack image create --disk-format qcow2 \
         --container-format bare --tag octavia-amphora-image --file amphora-x64-haproxy.qcow2 \
         --private --project service amphora-x64-haproxy

   .. note::
        Alternatively you can specify the new image in the appropriate settings and rerun the
        ansible with an appropriate tag.

You can find more information about the diskimage script and the process at
https://opendev.org/openstack/octavia/src/branch/master/diskimage-create

Here is a script to perform all those tasks at once:

    .. code-block:: shell-session

        #!/usr/bin/env bash
        set -euo pipefail

        VENV_DIR="/opt/octavia-image-build"
        WORK_DIR="/tmp"
        OCTAVIA_REPO="${WORK_DIR}/octavia"
        OUTPUT_IMAGE="amphora-x64-haproxy.qcow2"

        echo "Installing required packages..."
        apt-get update
        apt-get install -y \
            qemu-system \
            uuid-runtime \
            curl \
            kpartx \
            git \
            jq \
            python3-pip \
            python3-virtualenv \
            debootstrap

        echo "Creating Python virtual environment..."
        virtualenv -p /usr/bin/python3 "${VENV_DIR}"

        source "${VENV_DIR}/bin/activate"

        echo "Cloning Octavia repository..."
        cd "${WORK_DIR}"
        rm -rf "${OCTAVIA_REPO}"
        git clone https://opendev.org/openstack/octavia "${OCTAVIA_REPO}"

        echo "Installing diskimage-builder..."
        "${VENV_DIR}/bin/pip" install --isolated \
            git+https://opendev.org/openstack/diskimage-builder

        echo "Building Amphora image..."
        cd "${OCTAVIA_REPO}/diskimage-create"
        ./diskimage-create.sh

        echo "Moving output image to ${WORK_DIR}..."
        mv "${OUTPUT_IMAGE}" "${WORK_DIR}/"

        deactivate

        echo "Done. Image available at: ${WORK_DIR}/${OUTPUT_IMAGE}"

And upload image:

     .. code-block:: shell-session

        openstack image delete amphora-x64-haproxy
        openstack image create --disk-format qcow2 \
          --container-format bare --tag octavia-amphora-image --file /tmp/amphora-x64-haproxy.qcow2 \
          --private --project service amphora-x64-haproxy

Creating the cryptographic certificates
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::
    For production installation make sure that you review this very
    carefully with your own security requirements and potentially use
    your own CA to sign the certificates.

The system will automatically generate and use self-signed
certificates with different Certificate Authorities for control plane
and amphora. Make sure to store a copy in a safe place for potential
disaster recovery.

Optional: Configuring Octavia with ssh access to the amphora
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In rare cases it might be beneficial to gain ssh access to the
amphora for additional trouble shooting. Follow these steps to
enable access.

#. Configure Octavia accordingly

   Add a ``octavia_ssh_enabled: True`` to the user file in
   /etc/openstack-deploy

#. Run ``os_octavia`` role. SSH key will be generated and uploaded

.. note::
    SSH key will be stored on the ``octavia_keypair_setup_host`` (which
    by default is ``localhost``) in ``~/.ssh/{{ octavia_ssh_key_name }}``

Optional: Tuning Octavia for production use
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We suggest setting more specific ``octavia_cert_dir`` to prevent
accidental certificate rotation.
