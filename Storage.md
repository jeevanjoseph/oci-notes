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

 Group volumes together to perform backup and other tasks like clonig
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
  * Backup and clone cannot run at the same time (request will be rejected, not queued)

#### Resizing Volumes

  * Size can only be increased, not decreased
  * In place resize
    * Stop instance, detah volume, resize volume, reattach and resize the partition on the OS
  * Back up volume and use a larger volume to restore to
  * Clone a volume to a larger target volume.
