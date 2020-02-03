..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This tech note describes user identification and authorization.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

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
- LDAP host group: ``<cluster>``

Example: amor cluster
---------------------

A group named amor would be configured as follows:

- IPA hostgroup: ``amor``
  - Host: ``amor01.cp.lsst.org``
  - Host: ``amor02.cp.lsst.org``
- IPA user group (for host access): ``amor``
  - Users in ``amor`` can SSH to the amor nodes
- IPA user group (for sudo access): ``amor-sudo``
  - Users in ``amor`` can SSH to the amor nodes
- IPA HBAC rule: ``amor-users``
  - The ``amor-users`` HBAC rule grants SSH access for ``amor`` user group to the ``amor`` host group.
- IPA Sudo rule: ``amor-sudo``

Users with access to amor hosts would be added to the ``amor`` unix group.

Users with sudo permissions to amor amor hosts would be added to the ``amor-sudo`` unix group.

::
   $ ipa hostgroup-show amor
     Host-group: amor
     Description: amor nodes
     Member hosts: amor02.cp.lsst.org, amor01.cp.lsst.org
     Member of Sudo rule: amor-sudo    # see: `ipa sudorule-show amor-sudo`
     Member of HBAC rule: amor-users   # see: `ipa hbacrule-show amor-users`

::
   $ ipa hbacrule-show amor-users
     Rule name: amor-users
     Service category: all
     Enabled: TRUE
     User Groups: amor   # see: `ipa group-show amor`
     Host Groups: amor   # see: `ipa hostgroup-show amor`

::
   $ ipa sudorule-show amor-sudo
     Rule name: amor-sudo
     Enabled: TRUE
     Command category: all
     RunAs User category: all
     RunAs Group category: all
     User Groups: amor-sudo  # see: `ipa group-show amor-sudo`
     Host Groups: amor       # see: `ipa hostgroup-show amor`

Example: Creating a new hostgroup and associated rules
------------------------------------------------------

**Create a user group that can SSH to HVAC servers**

::
   $ ipa group-add hvac --desc "Summit HVAC users"
   ------------------
   Added group "hvac"
   ------------------
     Group name: hvac
     GID: 73027

**Create a hostgroup to contain all HVAC servers**
::
   $ ipa hostgroup-add hvac --desc "Summit HVAC servers"
   ----------------------
   Added hostgroup "hvac"
   ----------------------
     Host-group: hvac
     Description: Summit HVAC servers

**Allow users in the hvac group to access servers in the hvac hostgroup**
::
   $ ipa hbacrule-add hvac-users
   ----------------------------
   Added HBAC rule "hvac-users"
   ----------------------------
     Rule name: hvac-users
     Enabled: TRUE

   $ ipa hbacrule-add-host hvac-users --hostgroups=hvac
     Rule name: hvac-users
     Enabled: TRUE
     Host Groups: hvac
   -------------------------
   Number of members added 1
   -------------------------

   $ ipa hbacrule-add-user hvac-users --groups=hvac
     Rule name: hvac-users
     Enabled: TRUE
     User Groups: hvac
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
