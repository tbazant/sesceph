This is a basic salt module for ceph configuration and deployment.

All methods in this module are intended to be atomic and idempotent. Some state
changes are in practice made up of many steps, but will verify that all stages
have succeeded before presenting the result to the caller. Most functions are
fully idempotent in operation so can be repeated or retried as often as
necessary without causing unintended effects. This is so clients of this module
do not need to keep track of whether the operation was already performed or not.
Some methods do depend upon the successful completion of other methods. While
not strictly idempotent, this is considered acceptable modules having
dependencies on other methods operation should present clear error messages.

Installation
------------

Copy the content of "_modules/ceph" to

    /srv/salt/_modules/ceph

and run:

    salt '*' saltutil.sync_modules

This will distribute the runner to all salt minions. To verify this process has
succeeded, on a specific node it is best to try and access the online
documentation.

The module will not execute unless the python library ceph-cfg is installed.

The source is available here:

   https://github.com/oms4suse/python-ceph-cfg

If this library is packaged for your platform you can install it with the
following code in your sls file:

    packages:
      pkg:
        - installed
        - names:
          - python-ceph-cfg

Otherwise this library must be installed by hand on all platforms.

Documentation
-------------

To show ceph method:

    salt "ceph-node*" ceph -d

All API methods should provide documentation. To list all runners methods
available in your salt system please run:

    salt-run doc.execution

Execution
---------

All functions in this application are under the ceph namespace.

Example
~~~~~~~

Get a list of potential MON keys:

    # salt "*ceph-node*" ceph.keyring_create \
        keyring_type=admin

This will not persist the created key, but if a persistent key already exists
this function will return the persistent key.

Use the output to write the keyring to admin nodes:

    # salt "*ceph-node*" ceph.keyring_save \
        keyring_type=admin \
        secret='AQBR8KhWgKw6FhAAoXvTT6MdBE+bV+zPKzIo6w=='

Repeat the process for the MON keyring:

    # salt "*ceph-node*" ceph.keyring_create \
        keyring_type=mon

Use the output to write the keyring to MON nodes:

    # salt "*ceph-node*" ceph.keyring_save \
        keyring_type=admin \
        secret='AQCpY6ZW2KCRExAAxbJ+dljnln40wYmb7UvHcQ=='

Repeat the process for the osd keyring:

    # salt "*ceph-node*" ceph.keyring_create \
        keyring_type=osd

Use the output to write the keyring to osd and mon nodes:

    # salt "*ceph-node*" ceph.keyring_save \
        keyring_type=osd \
        secret='AQDant1WGP7qJBAA1Iqr9YoNo4YExai4ieXYMg=='

Repeat the process for the rgw keyring:

    # salt "*ceph-node*" ceph.keyring_create \
        keyring_type=rgw

Use the output to write the keyring to rgw and mon nodes:

    # salt "*ceph-node*" ceph.keyring_save \
        keyring_type=rgw \
        secret='AQDant1WGP7qJBAA1Iqr9YoNo4YExai4ieXYMg=='

Repeat the process for the mds keyring:

    # salt "*ceph-node*" ceph.keyring_create \
        keyring_type=mds

Use the output to write the keyring to mds and mon nodes:

    # salt "*ceph-node*" ceph.keyring_save \
        keyring_type=mds \
        secret='AQADn91WzLT9OBAA+LqKkXFBzwszBX4QkCkFYw=='

Create the monitor daemons:

    # salt "*ceph-node*" ceph.mon_create

The ceph.mon_create function requires both the admin and the mon keyring to
exist before this function can be successful.

Get monitor status:

    # salt "*ceph-node*" ceph.mon_status

The ceph.mon_status function requires the ceph.mon_create function to have
completed successfully.

List authorized keys:

    # salt "*ceph-node*" ceph.keyring_auth_list

The ceph.auth_list function will only execute successfully on nodes running
mon daemons which are in quorum.

Get a list of potential OSD keys:

    # salt "*ceph-node*" ceph.keyring_osd_create

Use one output to write the keyring to OSD nodes:

    # salt "*ceph-node*" ceph.keyring_osd_save '[client.bootstrap-osd]
    > key = AQAFNKZWaByNLxAAmIx9CbAaN+9L5KvZunmo2w==
    > caps mon = "allow profile bootstrap-osd"
    > '

Authorise the OSD boot strap:

    # salt "*ceph-node*" ceph.keyring_osd_auth_add

The ceph.keyring_osd_auth_add function will only execute successfully on nodes
running mon daemons which are in quorum.

Create some OSDs

    # salt "*ceph-node*" ceph.osd_prepare  osd_dev=/dev/vdb

The ceph.osd_prepare function will only execute successfully on nodes
with OSD boot strap keys writern.

SLS example
~~~~~~~~~~~

An example SLS file. After the writing of all keys:

    mon_create:
      module.run:
        - name:  ceph.mon_create

    keyring_osd_auth_add:
      module.run:
        - name:  ceph.keyring_osd_auth_add

    prepare:
      module.run:
        - name: ceph.osd_prepare
        - kwargs: {
            osd_dev: /dev/vdb
            }

Common Use cases
----------------

To discover OSD's

    salt 'ceph-node*' ceph.osd_discover

Will query all nodes whose name starts with 'ceph-node*' and return all OSDs
by cluster for example:

    ceph-node2.example.org:
        ----------
        5abcca4c-efb3-4c8f-96eb-cb85c30af50e:
            |_
              ----------
              dev:
                  /dev/vdc1
              dev_journal:
                  /dev/vdc2
              dev_parent:
                  /dev/vdc
              fsid:
                  be2a3e75-6190-406f-ab41-435ed5257319
              journal_uuid:
                  bd760a10-d78b-491a-9569-d80d329e0489
              magic:
                  ceph osd volume v026
              whoami:
                  7
            |_
              ----------
              dev:
                  /dev/vdb1
              dev_journal:
                  /dev/vdb2
              dev_parent:
                  /dev/vdb
              fsid:
                  4deff815-0e6f-498a-aa24-8fdbe41430de
              journal_uuid:
                  aac8012a-1d5e-414a-b969-54b8e3cf1240
              magic:
                  ceph osd volume v026
              whoami:
                  0

This allowed me to easily identify orphaned OSDs :)

Code layout
-----------

The code is structured with basic methods calling 3 main class types.

1. Models

This stores all the gathered configuration on the node to apply the function.

2. Loaders

These objects are used to update the Model.

3. Presenters

These objects are used to present the data in the model to the API users.
