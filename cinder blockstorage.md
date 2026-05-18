================================================================
CINDER (BLOCK STORAGE SERVICE)
OpenStack Administration Lab Notes
================================================================

WHAT IS CINDER?
---------------
Cinder provides persistent block storage to VMs in OpenStack.
Unlike the VM root disk which is deleted when VM is deleted,
a Cinder volume persists independently.

Key characteristics:
  - Volumes persist even after VM deletion
  - Can be detached from one VM and attached to another
  - Like an external USB drive or AWS EBS volume
  - Supports snapshots and backups

CINDER COMPONENTS:
  - cinder-api       : Accepts and validates API requests
  - cinder-scheduler : Decides which storage backend to use
  - cinder-volume    : Manages actual volume on storage backend

STORAGE BACKENDS:
  - LVM    : Logical Volume Manager (default in DevStack)
  - Ceph   : Distributed storage (common in production)
  - NFS    : Network File System
  - iSCSI  : Storage Area Network protocol
  - NetApp, Pure Storage, EMC (enterprise backends)

VOLUME STATES:
  - creating   : Being provisioned on backend
  - available  : Ready to attach to a VM
  - in-use     : Currently attached to a VM
  - error      : Something went wrong
  - deleting   : Being removed

================================================================
LAB COMMANDS & OUTPUT
================================================================

1. CHECK CINDER SERVICE STATUS
-------------------------------
Purpose: Verify all storage services are healthy

  $ openstack volume service list

  Output:
  +------------------+------------------+------+---------+-------+
  | Binary           | Host             | Zone | Status  | State |
  +------------------+------------------+------+---------+-------+
  | cinder-scheduler | myvm             | nova | enabled | up    |
  | cinder-volume    | myvm@lvmdriver-1 | nova | enabled | up    |
  +------------------+------------------+------+---------+-------+

EXPLANATION:
  - cinder-scheduler : Picks best backend for new volumes
  - cinder-volume    : Manages volumes on actual storage
  - myvm@lvmdriver-1 : Host @ backend name
  - lvmdriver-1      : LVM is our storage backend
  - State: up        = service is healthy


2. CHECK VOLUME TYPES
---------------------
Purpose: See available storage tiers/backends

  $ openstack volume type list

  Output:
  +--------------------------------------+-------------+-----------+
  | ID                                   | Name        | Is Public |
  +--------------------------------------+-------------+-----------+
  | 02954696-32c5-4828-ab02-91dd37eb801e | lvmdriver-1 | True      |
  | 0f52e926-ae8d-4c65-a79f-3dfdc509b759 | __DEFAULT__ | True      |
  +--------------------------------------+-------------+-----------+

EXPLANATION:
  - __DEFAULT__  : Used when no type specified in request
  - lvmdriver-1  : Maps to our LVM storage backend
  - In production: SSD, HDD, Gold, Silver tiers etc.
  - Is Public: True = any project can request this type


3. CREATE A VOLUME
------------------
Purpose: Provision a new persistent block storage device

  $ openstack volume create \
    --size 1 \
    --description "My first volume" \
    myvolume

  Output:
  +---------------------+--------------------------------------+
  | Field               | Value                                |
  +---------------------+--------------------------------------+
  | attachments         | []                                   |
  | bootable            | false                                |
  | encrypted           | False                                |
  | id                  | d4167fea-28f1-4715-b415-482bcf3f67ce |
  | multiattach         | False                                |
  | name                | myvolume                             |
  | size                | 1                                    |
  | status              | creating                             |
  | type                | lvmdriver-1                          |
  +---------------------+--------------------------------------+

EXPLANATION:
  - size: 1           : 1 GB volume
  - status: creating  : Being provisioned on LVM backend
  - bootable: false   : Data volume, not a boot volume
  - attachments: []   : Not attached to any VM yet
  - multiattach: False: Can only attach to one VM at a time
  - encrypted: False  : Volume not encrypted

  Verify volume is ready:
  $ openstack volume list
  | myvolume | available | 1 |   <- ready to attach


4. ATTACH VOLUME TO VM
----------------------
Purpose: Connect storage device to a running VM

  $ openstack server add volume myserver2 myvolume

  Output:
  +-----------------------+--------------------------------------+
  | Field                 | Value                                |
  +-----------------------+--------------------------------------+
  | Device                | /dev/vdb                             |
  | Delete On Termination | False                                |
  | Server ID             | 03ef4241-3aa0-4085-ae0d-f95368acce27 |
  | Volume ID             | d4167fea-28f1-4715-b415-482bcf3f67ce |
  +-----------------------+--------------------------------------+

  Verify attachment:
  $ openstack volume list
  | myvolume | in-use | 1 | Attached to myserver2 on /dev/vdb |

EXPLANATION:
  - Device: /dev/vdb       : Second disk in VM (/dev/vda = root)
  - Delete On Termination: False = volume survives VM deletion
  - status changes: available -> in-use after attachment

  Inside VM verify disk:
  $ sudo fdisk -l /dev/vdb
  Disk /dev/vdb: 1 GiB, 1073741824 bytes, 2097152 sectors

  Format and mount:
  $ sudo mkfs.ext4 /dev/vdb
  $ sudo mkdir /mydata && sudo mount /dev/vdb /mydata


5. DETACH VOLUME FROM VM
------------------------
Purpose: Disconnect storage from VM (required before snapshot)

  $ openstack server remove volume myserver2 myvolume

  Verify:
  $ openstack volume list
  | myvolume | available | 1 |   <- back to available


6. CREATE A SNAPSHOT
--------------------
Purpose: Point-in-time copy of a volume for backup/restore

  $ openstack volume snapshot create \
    --volume myvolume \
    mysnapshot

  Output:
  +-------------+--------------------------------------+
  | Field       | Value                                |
  +-------------+--------------------------------------+
  | id          | 9a56144b-1cb4-40b9-b65b-af913bd95e2b |
  | name        | mysnapshot                           |
  | size        | 1                                    |
  | status      | creating                             |
  | volume_id   | d4167fea-28f1-4715-b415-482bcf3f67ce |
  +-------------+--------------------------------------+

  Verify snapshot is ready:
  $ openstack volume snapshot list
  | mysnapshot | available | 1 |

EXPLANATION:
  - Snapshot captures exact state of volume at that moment
  - Volume must be in 'available' state (detached from VM)
  - Use before risky operations so you can rollback
  - snapshot_id links snapshot to source volume


7. RESTORE VOLUME FROM SNAPSHOT
---------------------------------
Purpose: Create new volume from a snapshot (data recovery)

  $ openstack volume create \
    --snapshot mysnapshot \
    --size 1 \
    myvolume-restored

  Output:
  +-------------+--------------------------------------+
  | Field       | Value                                |
  +-------------+--------------------------------------+
  | name        | myvolume-restored                    |
  | snapshot_id | 9a56144b-1cb4-40b9-b65b-af913bd95e2b |
  | status      | creating                             |
  | size        | 1                                    |
  +-------------+--------------------------------------+

  Verify:
  $ openstack volume list
  | myvolume-restored | available | 1 |
  | myvolume          | available | 1 |

EXPLANATION:
  - snapshot_id shows this volume was created from snapshot
  - New volume contains all data from snapshot point in time
  - Original volume and snapshot remain unchanged
  - Can attach restored volume to any VM for data recovery

================================================================
COMMON CINDER OPERATIONS IN PRODUCTION
================================================================

Extend a volume (increase size):
  $ openstack volume set --size 5 myvolume
  Note: VM may need filesystem resize after extension

Transfer volume between projects:
  $ openstack volume transfer request create myvolume
  $ openstack volume transfer request accept <transfer-id> <auth-key>

Create bootable volume from image:
  $ openstack volume create \
    --image cirros-0.5.2-x86_64-disk \
    --size 5 \
    --bootable \
    boot-volume

List volume backups:
  $ openstack volume backup list

================================================================
KEY QUESTIONS - CINDER
================================================================

Q: What is the difference between a volume and a snapshot?
A: A volume is a persistent block storage device that can be
   attached to VMs. A snapshot is a point-in-time read-only
   copy of a volume used for backup or creating new volumes.

Q: What happens to a Cinder volume when a VM is deleted?
A: By default (Delete On Termination: False) the volume
   persists and returns to 'available' state. It can be
   attached to another VM. If Delete On Termination was set
   to True at attachment time, volume is deleted with the VM.

Q: Why must a volume be detached before taking a snapshot?
A: Cinder requires the volume to be in 'available' state for
   snapshot. An attached volume is 'in-use'. Force snapshots
   of in-use volumes are possible but risk data inconsistency.

Q: What storage backends does Cinder support?
A: LVM (default), Ceph (most common in production), NFS,
   iSCSI, Fibre Channel, NetApp, Pure Storage, EMC, HPE,
   IBM Storwize, and many more via drivers.

Q: How do you increase volume size?
A: openstack volume set --size <new-size> <volume>
   Then inside VM: resize2fs /dev/vdb (for ext4)
   Note: Can only increase, not decrease volume size.

================================================================
END
================================================================
