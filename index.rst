:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This tech note describes user identification and authorization.

IPA Server Topology
===================

FreeIPA servers are multi-master; meaning that changes to the directory can be
made at any given server and will be distributed to all other masters. Because
all IPA servers are masters and fully replicated, LDAP queries and changes can
be made at Cerro Pachon when the link to La Serena has been severed.

Two IPA servers (``ipa1.<site>.<domain>`` and ``ipa2.<site>.<domain>``) are
deployed at each site, providing quick LDAP lookups and redundancy.

Unix HBAC and Sudo
==================

Host access and sudo permissions are applied to groups of hosts (hostgroups),
and groups of users (user groups/unix groups). Access is always provided by
assigning users to user groups, and hosts to host-groups.

Two levels of access are provided: basic login access to the host (which is
generally done through SSH) and full sudo permissions.

- Unix user group for host access (HBAC): ``<cluster>``
- Unix user group for sudo access: ``<cluster>-sudo``
- IPA host group: ``<cluster>``

Two access rules are used: an HBAC rule that grants access to the host, and a
sudo rule that grants full sudo access.

- HBAC rule: ``<cluster>-users``
- Sudo rule: ``<cluster>-sudo``

.. note::

   Our current convention is that user groups and hostgroups are always
   singular. Sudo rules are always ``<cluster>-sudo`` and HBAC rules are always
   ``<cluster>-users``.

   This decision is subject to revision in the future, but until that point
   we'll stick to this convention to avoid small errors due to inconsistent
   pluralization.

Example: amor cluster
---------------------

A group named amor would be configured as follows:

- Unix user group for host access (HBAC): ``amor``
- Unix user group for sudo access: ``amor-sudo``
- IPA host group: ``amor``

The access rules are as follows:

- HBAC rule: ``amor-users``
- Sudo rule: ``amor-sudo``

Users with access to amor hosts would be added to the ``amor`` unix group.

Users with sudo permissions to amor amor hosts would be added to the ``amor-sudo`` unix group.

.. code-block:: console

   $ ipa hostgroup-show amor
     Host-group: amor
     Description: amor nodes
     Member hosts: amor02.cp.lsst.org, amor01.cp.lsst.org
     Member of Sudo rule: amor-sudo    # see: `ipa sudorule-show amor-sudo`
     Member of HBAC rule: amor-users   # see: `ipa hbacrule-show amor-users`

.. code-block:: console

   $ ipa hbacrule-show amor-users
     Rule name: amor-users
     Service category: all
     Enabled: TRUE
     User Groups: amor   # see: `ipa group-show amor`
     Host Groups: amor   # see: `ipa hostgroup-show amor`

.. code-block:: console

   $ ipa sudorule-show amor-sudo
     Rule name: amor-sudo
     Enabled: TRUE
     Command category: all
     RunAs User category: all
     RunAs Group category: all
     User Groups: amor-sudo  # see: `ipa group-show amor-sudo`
     Host Groups: amor       # see: `ipa hostgroup-show amor`

Example: Creating an ``hvac`` hostgroup and user group
------------------------------------------------------

In this example we create the following resources:

1. :ref:`**hvac** unix user group <create-hvac-group>` for host access (HBAC)
2. :ref:`**hvac-sudo** unix user group <create-hvac-sudo-group>` for sudo access
3. :ref:`**hvac** IPA host group <create-hvac-hostgroup>`
4. :ref:`**hvac-users** HBAC rule <create-hvac-users-hbacrule>`
5. :ref:`**hvac-sudo** Sudo rule <create-hvac-sudo-sudorule>`

User group creation
^^^^^^^^^^^^^^^^^^^

.. code-block:: console
   :name: create-hvac-group

   $ ipa group-add hvac --desc "Summit HVAC users"
   ------------------
   Added group "hvac"
   ------------------
     Group name: hvac
     Description: Summit HVAC users
     GID: 73027

.. code-block:: console
   :name: create-hvac-sudo-group

   $ ipa group-add hvac-sudo --desc "Summit HVAC sudo users"
   ------------------
   Added group "hvac-sudo"
   ------------------
     Group name: hvac-sudo
     Description: Summit HVAC sudo users
     GID: 73034

.. code-block:: console
   :name: create-hvac-hostgroup

   $ ipa hostgroup-add hvac --desc "Summit HVAC servers"
   ----------------------
   Added hostgroup "hvac"
   ----------------------
     Host-group: hvac
     Description: Summit HVAC servers

.. code-block:: console
   :name: create-hvac-users-hbacrule
   :emphasize-lines: 1,8,16

   $ ipa hbacrule-add hvac-users --servicecat=all
   ----------------------------
   Added HBAC rule "hvac-users"
   ----------------------------
     Rule name: hvac-users
     Service category: all
     Enabled: TRUE
   $ ipa hbacrule-add-host hvac-users --hostgroups=hvac
     Rule name: hvac-users
     Service category: all
     Enabled: TRUE
     Host Groups: hvac
   -------------------------
   Number of members added 1
   -------------------------
   $ ipa hbacrule-add-user hvac-users --groups=hvac
     Rule name: hvac-users
     Service category: all
     Enabled: TRUE
     User Groups: hvac
     Host Groups: hvac
   -------------------------
   Number of members added 1
   -------------------------

.. code-block:: console
   :name: create-hvac-users-sudorule
   :emphasize-lines: 1,10,20

   $ ipa sudorule-add hvac-sudo --cmdcat=all --runasusercat=all --runasgroupcat=all
   ---------------------------
   Added Sudo Rule "hvac-sudo"
   ---------------------------
     Rule name: hvac-sudo
     Enabled: TRUE
     Command category: all
     RunAs User category: all
     RunAs Group category: all
   $ ipa sudorule-add-user hvac-sudo --groups=hvac-sudo
     Rule name: hvac-sudo
     Enabled: TRUE
     Command category: all
     RunAs User category: all
     RunAs Group category: all
     User Groups: hvac-sudo
   -------------------------
   Number of members added 1
   -------------------------
   $ ipa sudorule-add-host hvac-sudo --hostgroups=hvac
     Rule name: hvac-sudo
     Enabled: TRUE
     Command category: all
     RunAs User category: all
     RunAs Group category: all
     User Groups: hvac-sudo
     Host Groups: hvac
   -------------------------
   Number of members added 1
   -------------------------

IPA Directory RBAC
==================

IPA Directory RBAC  differs from host access control because while host access
control provides access to hosts and sudo, IPA RBAC grants permissions to
modify the directory itself.

Roles bundle together groups of users, and groups of privileges.

A fully expanded RBAC role looks roughly like the following:

- Desktop Support (RBAC Role)
   - User groups: ``desktop-support`` (see: ``ipa group-show desktop-support``)
   - Privileges:
      - Stage User Provisioning (see ``ipa privilege-show "Stage User Provisioning"``)
         - System: Add Stage User (see ``ipa permission-show "System: Add Stage User"``)
            - Granted rights: add
            - ``Subtree: cn=staged users,cn=accounts,cn=provisioning,dc=lsst,dc=cloud``
         - System: Modify Stage User (see ``ipa permission-show "System: Modify Stage User"``)
            - Granted rights: modify
            - ``Subtree: cn=staged users,cn=accounts,cn=provisioning,dc=lsst,dc=cloud``
         - System: Delete Stage User (see ``ipa permission-show "System: Delete Stage User"``)
            - Granted rights: delete
            - ``Subtree: cn=staged users,cn=accounts,cn=provisioning,dc=lsst,dc=cloud``
      - VPN Group Administrators (see ``ipa privilege-show "VPN Group Administrators"``)
         - "Manage Chile VPN group" (see ``ipa permission-show "Manage Chile VPN group"``)
            - Granted rights: write
            - Target DN: ``cn=vpn-cl,cn=groups,cn=accounts,dc=lsst,dc=cloud``
            - Target group: ``vpn-cl``

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
