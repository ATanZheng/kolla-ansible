.. _operating-kolla:

===============
Operating Kolla
===============

Versioning
~~~~~~~~~~

Kolla uses the ``x.y.z`` `semver <https://semver.org/>`_ nomenclature for
naming versions. Kolla's initial Pike release was ``5.0.0`` and the initial
Queens release is ``6.0.0``. The Kolla community commits to release z-stream
updates every 45 days that resolve defects in the stable version in use and
publish those images to the Docker Hub registry.

To prevent confusion, the Kolla community recommends using an alpha identifier
``x.y.z.a`` where ``a`` represents any customization done on the part of the
operator. For example, if an operator intends to modify one of the Docker files
or the repos from the originals and build custom images for the Pike version,
the operator should start with version 5.0.0.0 and increase alpha for each
release. Alpha tag usage is at discretion of the operator. The alpha identifier
could be a number as recommended or a string of the operator's choosing.

To customize the version number uncomment ``openstack_release`` in globals.yml
and specify the desired version number or name (e.g. ``victoria``,
``wallaby``). If ``openstack_release`` is not specified, Kolla will deploy or
upgrade using the version number information contained in the kolla-ansible
package.

Upgrade procedure
~~~~~~~~~~~~~~~~~

.. note::

   If you have set ``enable_cells`` to ``yes`` then you should read the
   upgrade notes in the :ref:`Nova cells guide<nova-cells-upgrade>`.

Kolla's strategy for upgrades is to never make a mess and to follow consistent
patterns during deployment such that upgrades from one environment to the next
are simple to automate.

Kolla implements a one command operation for upgrading an existing deployment
consisting of a set of containers and configuration data to a new deployment.

Limitations and Recommendations
-------------------------------

.. note::

   Varying degrees of success have been reported with upgrading the libvirt
   container with a running virtual machine in it. The libvirt upgrade still
   needs a bit more validation, but the Kolla community feels confident this
   mechanism can be used with the correct Docker storage driver.

.. note::

   Because of system technical limitations, upgrade of a libvirt container when
   using software emulation (``virt_type = qemu`` in nova.conf), does not work
   at all. This is acceptable because KVM is the recommended virtualization
   driver to use with Nova.

.. note::

   Please note that when the ``use_preconfigured_databases`` flag is set to
   ``"yes"``, you need to have the ``log_bin_trust_function_creators`` set to
   ``1`` by your database administrator before performing the upgrade.

.. note::

   If you have separate keys for nova and cinder, please be sure to set
   ``ceph_nova_keyring: ceph.client.nova.keyring`` and ``ceph_nova_user: nova``
   in ``/etc/kolla/globals.yml``

Ubuntu Focal 20.04
------------------

The Victoria release adds support for Ubuntu Focal 20.04 as a host operating
system. Ubuntu users upgrading from Ussuri should first upgrade OpenStack
containers to Victoria, which uses the Ubuntu Focal 20.04 base container image.
Hosts should then be upgraded to Ubuntu Focal 20.04.

CentOS Stream 8
---------------

The Wallaby release adds support for CentOS Stream 8 as a host operating
system. CentOS Stream 8 support will also be added to a Victoria stable
release. CentOS Linux users upgrading from Victoria should first migrate hosts
and container images from CentOS Linux to CentOS Stream before upgrading to
Wallaby.

Preparation
-----------

While there may be some cases where it is possible to upgrade by skipping this
step (i.e. by upgrading only the ``openstack_release`` version) - generally,
when looking at a more comprehensive upgrade, the kolla-ansible package itself
should be upgraded first. This will include reviewing some of the configuration
and inventory files. On the operator/master node, a backup of the
``/etc/kolla`` directory may be desirable.

If upgrading to ``|KOLLA_OPENSTACK_RELEASE|``, upgrade the kolla-ansible
package:

.. code-block:: console

   pip install --upgrade git+https://opendev.org/openstack/kolla-ansible@|KOLLA_BRANCH_NAME|

If this is a minor upgrade, and you do not wish to upgrade kolla-ansible
itself, you may skip this step.

The inventory file for the deployment should be updated, as the newer sample
inventory files may have updated layout or other relevant changes.
Use the newer ``|KOLLA_OPENSTACK_RELEASE|`` one as a starting template, and
merge your existing inventory layout into a copy of the one from here::

    /usr/share/kolla-ansible/ansible/inventory/

In addition the ``|KOLLA_OPENSTACK_RELEASE|`` sample configuration files should
be taken from::

    # CentOS
    /usr/share/kolla-ansible/etc_examples/kolla

    # Ubuntu
    /usr/local/share/kolla-ansible/etc_examples/kolla

At this stage, files that are still at the previous version and need manual
updating are:

- ``/etc/kolla/globals.yml``
- ``/etc/kolla/passwords.yml``

For ``globals.yml`` relevant changes should be merged into a copy of the new
template, and then replace the file in ``/etc/kolla`` with the updated version.
For ``passwords.yml``, see the ``kolla-mergepwd`` instructions in
`Tips and Tricks`.

For the kolla docker images, the ``openstack_release`` is updated to
``|KOLLA_OPENSTACK_RELEASE|``:

.. code-block:: yaml

   openstack_release: |KOLLA_OPENSTACK_RELEASE|

Once the kolla release, the inventory file, and the relevant configuration
files have been updated in this way, the operator may first want to 'pull'
down the images to stage the ``|KOLLA_OPENSTACK_RELEASE|`` versions. This can
be done safely ahead of time, and does not impact the existing services.
(optional)

Run the command to pull the ``|KOLLA_OPENSTACK_RELEASE|`` images for staging:

.. code-block:: console

   kolla-ansible pull

At a convenient time, the upgrade can now be run (it will complete more
quickly if the images have been staged ahead of time).

Perform the Upgrade
-------------------

To perform the upgrade:

.. code-block:: console

   kolla-ansible upgrade

After this command is complete the containers will have been recreated from the
new images.

Tips and Tricks
~~~~~~~~~~~~~~~

Kolla Ansible CLI
-----------------

When running the ``kolla-ansible`` CLI, additional arguments may be passed to
``ansible-playbook`` via the ``EXTRA_OPTS`` environment variable.

``kolla-ansible -i INVENTORY deploy`` is used to deploy and start all Kolla
containers.

``kolla-ansible -i INVENTORY destroy`` is used to clean up containers and
volumes in the cluster.

``kolla-ansible -i INVENTORY mariadb_recovery`` is used to recover a
completely stopped mariadb cluster.

``kolla-ansible -i INVENTORY prechecks`` is used to check if all requirements
are meet before deploy for each of the OpenStack services.

``kolla-ansible -i INVENTORY post-deploy`` is used to do post deploy on deploy
node to get the admin openrc file.

``kolla-ansible -i INVENTORY pull`` is used to pull all images for containers.

``kolla-ansible -i INVENTORY reconfigure`` is used to reconfigure OpenStack
service.

``kolla-ansible -i INVENTORY upgrade`` is used to upgrades existing OpenStack
Environment.

``kolla-ansible -i INVENTORY check`` is used to do post-deployment smoke
tests.

``kolla-ansible -i INVENTORY stop`` is used to stop running containers.

``kolla-ansible -i INVENTORY deploy-containers`` is used to check and if
necessary update containers, without generating configuration.

``kolla-ansible -i INVENTORY prune-images`` is used to prune orphaned Docker
images on hosts.

``kolla-ansible -i INVENTORY1 -i INVENTORY2 ...`` Multiple inventories can be
specified by passing the ``--inventory`` or ``-i`` command line option multiple
times. This can be useful to share configuration between multiple environments.
Any common configuration can be set in ``INVENTORY1`` and ``INVENTORY2`` can be
used to set environment specific details.

``kolla-ansible -i INVENTORY gather-facts`` is used to gather Ansible facts,
for example to populate a fact cache.

.. note::

   In order to do smoke tests, requires ``kolla_enable_sanity_checks=yes``.

Passwords
---------

The following commands manage the Kolla Ansible passwords file.

``kolla-mergepwd --old OLD_PASSWDS --new NEW_PASSWDS --final FINAL_PASSWDS``
is used to merge passwords from old installation with newly generated
passwords during upgrade of Kolla release. The workflow is:

#. Save old passwords from ``/etc/kolla/passwords.yml`` into
   ``passwords.yml.old``.
#. Generate new passwords via ``kolla-genpwd`` as ``passwords.yml.new``.
#. Merge ``passwords.yml.old`` and ``passwords.yml.new`` into
   ``/etc/kolla/passwords.yml``.

For example:

.. code-block:: console

   mv /etc/kolla/passwords.yml passwords.yml.old
   cp kolla-ansible/etc/kolla/passwords.yml passwords.yml.new
   kolla-genpwd -p passwords.yml.new
   kolla-mergepwd --old passwords.yml.old --new passwords.yml.new --final /etc/kolla/passwords.yml

.. note::

   ``kolla-mergepwd``, by default, keeps old, unused passwords intact.
   To alter this behavior, and remove such entries, use the ``--clean``
   argument when invoking ``kolla-mergepwd``.

Hashicorp Vault can be used as an alternative to Ansible Vault for storing
passwords generated by Kolla Ansible. To use Hashicorp Vault as the secrets
store you will first need to generate the passwords, and then you can
save them into an existing KV using the following command:

.. code-block:: console

   kolla-writepwd \
   --passwords /etc/kolla/passwords.yml \
   --vault-addr <VAULT_ADDRESS> \
   --vault-token <VAULT_TOKEN>

.. note::

   For a full list of ``kolla-writepwd`` arguments, use the ``--help``
   argument when invoking ``kolla-writepwd``.

To read passwords from Hashicorp Vault and generate a passwords.yml:

.. code-block:: console

   mv kolla-ansible/etc/kolla/passwords.yml /etc/kolla/passwords.yml
   kolla-readpwd \
   --passwords /etc/kolla/passwords.yml \
   --vault-addr <VAULT_ADDRESS> \
   --vault-token <VAULT_TOKEN>

Tools
-----

Kolla ships with several utilities intended to facilitate ease of operation.

``tools/cleanup-containers`` is used to remove deployed containers from the
system. This can be useful when you want to do a new clean deployment. It will
preserve the registry and the locally built images in the registry, but will
remove all running Kolla containers from the local Docker daemon. It also
removes the named volumes.

``tools/cleanup-host`` is used to remove remnants of network changes
triggered on the Docker host when the neutron-agents containers are launched.
This can be useful when you want to do a new clean deployment, particularly one
changing the network topology.

``tools/cleanup-images --all`` is used to remove all Docker images built by
Kolla from the local Docker cache.
