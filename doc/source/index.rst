=============================================================
OpenStack-Ansible role for the Octavia Load Balancing Service
=============================================================

.. toctree::
   :maxdepth: 2

   configure-octavia.rst

This is an OpenStack-Ansible role to deploy the Octavia Load Balancing
service. See the `role-octavia spec`_ for more information.

.. _role-octavia spec: TBD


To clone or view the source code for this repository, visit the role repository
for `os_octavia <https://github.com/openstack/openstack-ansible-os_octavia>`_.

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.


Required variables
~~~~~~~~~~~~~~~~~~

None.


Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml


Tags
====

This role supports the ``octavia-install`` and ``octavia-config`` tags.
Use the ``octavia-install`` tag to install and upgrade. Use the
``octavia-config`` tag to maintain configuration of the service.

