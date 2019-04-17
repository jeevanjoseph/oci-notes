## Storage

There are 5 options for storage in OCI
  * Locally attached NVMe SSDs
  * Remotely mounted iSCSI Block Storage (based on NVMe SSDs as well)
  * File Storage Service (NFSv3, largest capacity 8ExB)
  * Object storage
  * Archive storage

### Locally attached NVMe SSD

This offers the best performace as these SSDs are physically attached to the compute instances.
They are only available in the DenseIO and HighIO shapes.

* They are local to a single instance and they offer no data protection. 
  * No backup policies from OCI block storage is applicable.
  * User is responsible for manual backups or setting up RAID or multi-AD redundancy
  * If the compute instance is gone, then the data is gone.
  * Data survives reboots, but not terminations of instances.
* Only cloud provider to provide an SLA on locally attached NVMe

### Block Storage

Block volumes are remote block devices that can be mounted on the OS and are backed by NVMe SSDs.

* Mounted over the network using either iSCSI or paravirualized drivers. 
  * Paravirtualized drivers make calls to the hypervisor API, and are applicabe to VM shapes(where a HV exists)
* Block volumes are AD local. They cannot be moved across ADs or regions
  * They can be backed up to object storage and then restored to another block device in another AD or Region
  * Boot Volumes are backed by Block Volumes.
* After attaching a block volume, you need to format the disk and mount it to be usable.
* Block volumes are automatically replicated to protect against data loss
* Encryption
  * Data is automatically encrypted at rest for
    * Block Volumes
    * Boot Volumes
    * Volume backups 
    * Encryption is based on Oracle managed keys or customer managed keys (KMS)
  * Data encryption in transit.
    * Data encryption for  in-transit data is available for Paravirtualized Volume attachments
* Data Eradication
  * Eventual Overerite data eradiaction
  * A terminated block volume's data is overwritten before the volume is allocated again
* Stats 
  * Size ranges from 50GB to 32TB in 1GB increments
  * An instance can have 32 volume attachments (includes boot volume) => 32*32TB = 1PB
  * 1000 backups for Universal credits and 500 backups for pay as you go
* Uses
  * Increase storage avaialable to an instance.
  * They can be used to scale isntances by preserving the block volume and reattaching to an instance of a different shape.
  * Durable storage that is backed up based on policy.
    * Block volumes are automatically replicated as well.

#### Backup & restore

* Backups are stored on the object storage and customer pays for the storage used.
* One off backups that offer full or incremental options
* Automated backup policy
  * Bronze - Montly incremental backups retained for 12months + 1 Yearly Full backup retained for 5 years.
  * Silver - Weekly incremental retained for 4 weeks + Bronze
  * Gold - Daily incremental for 7 days + silver

#### Volume Groups

 Group volumes together to perform backup and other tasks like cloning. Makes it easier to manage storage for a logical group - like an application - as a single unit. Example : backup the web tier and database of an application, even across conpartments with this feature.

* Volumes can belong to multiple compartments.
* Only volumes in the available status may be added to a group
* A Volume can be part of only one Volume Group at a time
* When deleting a volume thats a part of a VG, the volume has to be first removed from the VG before deletion
* When a volume group is deleted, the volumes are not deleted.
* When a volume group backup is deleted, the backups for every volume in the group is deleted.
* when a volume group is cloned, a new volume group with clones for every volume in the source will be created.
* A volume group can have up to 32 volumes and max 128TB total capacity

#### Cloning

* Cloning is disk to disk deep copy. The object storage is not involved.
* Can be done only within an AD since block volumes are AD specific.
* Clones become ready for use (attachment to an instance), but copy is async
* Backup and clone cannot run at the same time (request will be rejected, not queued~~~)

#### Resizing Volumes

* Size can only be increased, not decreased
* In place resize
  * Stop instance, detah volume, resize volume, reattach and resize the partition on the OS
* Back up volume and use a larger volume to restore to
* Clone a volume to a larger target volume.

### Object Storage & Archive Storage

* Highly Durable - muliple copies of the data automatically managed.
* Scope is __Regional__
  * Petabytes+ Max object size is 10TB.
  * Unlimited Storage, Unstuctured data (Content Repository, Archives, IOT data, HDFS Connector )
  * REST API to read and write objects (vs. native iSCSI for block storage)
* Storage classes for __hot__ (Standard) and __cold__ (Archive) data.
  * chosen when a bucket is created, and cannot be changed later.
  * Objects in archive tier need to be restored to standard tier to copy or download them
    * Once Archived, the restore is temporary and the object returns to archive tier after a time limit
    * Archived objects can be listed, deleted and renamed in exactly the same way as non-archived objects
    * A request to restore an already restored object will do nothing. It does not reset the time window when the object returns to Archive.
    * Calls to an Archive bucket are not metered.
* Supports private access from within OCI using service gateways (clients in oci need not have NAT gateways or public subnets)
* A __Namespace__ is allocated to the tenancy at tenancy creation.
  * Namespaces are not editable
* Buckets are private by default, can be made public
  * Buckets reside in a compartment and the IAM policies dicate access permissions
  * Bucket names are 1-256 chars and match [A-Za-z0-9_-.]
  * Once a bucket is created in a storage tier (Normal or Archive) it cannot be changed.
  * When a bucket is made public, its contents are accessible anonymously. List contents, get object metadata and download the objects as well.
  * At creation time the default is to be able to list contents and download objects. The List contents feature can be disabled.
* Objects
  * Objects do not have OCIDs
  * Objects are identified by name, which needs to be unique in the bucket.
  * Object names can be upto 1024 chars and all chars except line-feed, newlines and NULLs are allowed.
  * Object names are UTF-8 encoded, they should be less than 1024 after UTF-8 encoding.
  * URL to an object is of the form `/n/<object_storage_namespace>/b/<bucket>/o/<object_name>`
  * Hierachies and prefixes are supported.
    * /n/<object_storage_namespace>/b/<bucket>/o/level1/object1
    * /n/<object_storage_namespace>/b/<bucket>/o/level1/object2
    * /n/<object_storage_namespace>/b/<bucket>/o/level1/level2/object1
    * /n/<object_storage_namespace>/b/<bucket>/o/level1/prefix_object1
    * /n/<object_storage_namespace>/b/<bucket>/o/level1/prefix_object2
    * Bulk operations can be performed based on the prefixes or the heirarchy level.
* __Cross Region Copy__
  * copies one object at a time from one bucket to another.
    * buckets can be in different namespaces(tenancy) or compartments or regions
    * requires an IAM policy that grants the __manage__ privilege to the Object Storage service in the target location.
    * allow service objectstorage-<region_name> to manage object-family in compartment <compartment_name>
    * Needs policy in both source and target regions.
    * Copy cannot use an archive bucket as _SOURCE_. An archived object needs to be restored to the normal bucket before it can be copied to another bucket.
    * Copy _CAN_ specify an archive bucket as _TARGET_. The object is immediately archived.
    * The target bucket must exist. Buckets are not created by the copy operation
    * If the source object is changed during a copy, the copy fails.
    * Only one object can be copied at a time
* __Multi-Part Uploads__
  * useful for large files. Max object size is 10TB.
  * Parts can be from 10MB to 50GB per part.
  * Parts need part numbers and are consecutive - 1-10,000 is the range. 
* __Lifecycle Rules__
  * can be based on prefixes.
  * Only actions are __Archive__ and __Delete__.
  * There is __NO__ lifecylce rule to __RESTORE__
  * For an Archive tier bucket, the only lifecycle operation is delete.
* __Pre Authenticated Requests__
  * Provides a method for exposing a bucket or object for anonymoud access
    * The Creator of the PAR needs to have access that is being granted.
    * PAR can be timelimited (URL expires after time limit)
    * Can be bucket scoped or object scoped.
    * Permissions for object scope can be RO, WO, and RW.
    * Permissions for bucket scope can be WO. Cannot read for PARs that are bucket scoped.
    * Users cannot list the contents of a bucket (there is no RO access PAR to a bucket)
    * PARs are not tied to user's dreds, user can change passwords and will still work
    * __DO__ depend on the user and his permissions, if the User is deleted, the PARs will fail. If the permissions of the user who generated the PAR no longer allow the operation, it fails.

### File Storage Service

* AD local service, provides exa-scale storage.
  * Can be accessed from the same AD, different AD or across Regions, and CPE (VPN.FastConnect)
* Uses NFSv3.
* Data can be snapshotted. The snapshots are charged by the storage they consume.
  * Stored in /.snapshot directory. Readonly, space efficient (nothing changes/ nothing stored.)
  * Always incremental
* Can scale on demand without pre-provisioning
* Encrypted by default.
* Can be accesses from 
* __Setup__
  1. A File system is created. Its not accessible yet. It needs to be exported to a mount target.
  2. A _MOUNT TARGET_ created. This is an IP address or a host name - acts as a network path to reach the filesystem.
     * default 2 mount targets per AD for each tenancy.
     * A _MOUNT TARGET_ can export 100 fielsystems.
     * This _MOUNT TARGET_ is localized to an AD, its IP is coming from a AD local subnet.
     * It requires 3 IPs , one primary, one for HA and a third one.
       * The two IP addresses are not shown to the user, so if the user manually grabs an ip for use on a resource, that can lead to issues because the user might inadvertedly grab a IP used by the mount target.
     * Best practice is to create the _MOUNT TARGETS_ in their own subnet and use seclists.
  3. The FS is exported to (associated with) the _MOUNT TARGET_.
     * An _EXPORT_ associates the _MOUNT TARGET_ with an __EXPORT PATH__.
     * An _EXPORT PATH_ is defined when exporting a filesystem and provides a unique path under the _MOUNT TARGET_ to access the filesystem.
     * The _EXPORT_ is added to the _EXPORT SET_ of the _MOUNT TARGET_, which is list of exports and the paths.
* __Summary__
  * Mount Target is an IP that maintains an _EXPORT SET_.
  * Each _EXPORT_ in the _EXPORT SET_ has a unique(within the _MOUNT TARGET_) path called the _EXPORT PATH_ and a corresponding File System.
  * There should be overlaps in the path. `/example/logs` and `/example/` are not allowed, because within the mount target, its ambiguous to which FS we are reffering to. `/example/logs` and `/example/data/` are fine.

* Security
  * FSS security is layered
    * IAM 
      * Controls the resource definitions - like creating mount targets and filesystems, permissions to change these definitions.
    * SecLists
      * Controls access at _MOUNT TARGET_ level, and encompasses all FS exported through that _MOUNT TARGET_
      * SecLists need to open open ports `2048-2050 and 111 for TCP` and `2048 and 111 for UDP` __even within the same subnet__ by creating __STATEFUL RULES__
    * Export Options
      * Provides security on an individual FS export
      * Access can be granular, RO,RW,WO.
      * permissions are attached by updating a specific FS export. can be done only using the API or CLI. 
    * NFS v3 Unix permissions