## Compute Insances on OCI
* [Compute Insances on OCI](#compute-insances-on-oci)
  * [Shapes](#shapes)
  * [Images](#images)
  * [Boot Volumes](#boot-volumes)
  * [Instance Metadata](#instance-metadata)
  * [Instance Configuration and Pools](#instance-configuration-and-pools)
  * [Autoscaling](#autoscaling)
### Shapes
Various shapes available across multiple platforms. 

* X5 & X7
  * On the x86 side we have the X5 and X7 platforms. The X7 has 52 OCPUs vs 36 on the X5. 
  * The big difference between the X5 and X7 is that on X5, network maxes out at 10Gbps, vs on X7 its 2x 25Gbps or better. 
  * The standard shapes have block storage that can be attached, and the Dense IO shapes have attached NVMe 
* AMD EPYC
  * Offers better price performance ratio
  * 3x more Vnics than intel at a small reduction in the performace.
  * Available in BM and VM.
* HPC shape also available
  * 36 core (higher clocked) cpu
  * __100Gbps RDMA__ network connectivity.
  * only BM

* Bare Metal
  * Get direct access to the actual hardware. Single tenant model.
  * No performance overhead
  * All devops and automation are available.
  * Usecases
    * Performance intensive workloads
    * Non Virtualized workloads
    * Bring your own license
    * Run your own hypervisor

* Virtual Machine
  * A hypervisor runs on the same hardware as the bare metal. 
  * Just BM divided in to smaller VMs using the hypervisor.
  * Standard shapes have block storage and dense io shapes have attached NVMe.
  * Generally lower network bandwidth and # of vnics.

### Images

* Linux - various flavors available - OEL, Ubuntu, CentOS.
  * user `opc` created for RHEL flavors.
  * user `ubuntu` created for Ubuntu
  * the default users have sudo privileges.
  * can customize provisioning using cloud-init
  * default firewall rules set to allow traffic on port 22
* Custom Images  
  * Create an image out of the boot volume of an instance.
  * Includes all changes made like software installed and configuration updated.
  * data on attached block volumes are not touched.
  * Max size is 300 Gb.
  * Windows images cannot be exported out of the tenancy
* Import / Export Images
  * Uses the Object Storage, but no charges for the storage used.
  * Useful for sharing custom images across regions and tenancies.
  * Max of 25 custom images per compartment
  * Instance has to shutdown for a few minutes while the image is being created
* Bring your own image
  * You can bring your own images in to OCI
  * Useful in migration process
  * Bring your own hypervisor as well. KVM XEN etc.
  * Older legacy OS images can be used on the emulated mode.

### Boot Volumes

These are block storage volumes provisioned and attached to an instance at the time of instance creation.

* They hold the OS for the instance.
* Encrypted by default, like any other block volume.
* They can be deleted when the instance is terminated.
* Can be used to scale an instance
  * Boot volume from, say, a VM is detached after stopping the VM
  * A new BM instance is created, specifying the detached boot volume.
  * You now effectively have a scaled up (VM->BM) instance. 
  * Everything on the older instance is still available on the new instance as well.
* 46/50GB is default for Linux, and 250 GB is the default for Windows, and 32TB is the max
  * A custom boot volume also requires the OS partition to be enlarged, for the OS to use it.
* They can be backed up and cloned. and there is no downtime when backing up or cloning.
* The backups and clones reside in object storage and the customer pays for the storage.
* The backups are crash consitent (all files are backed up at the same time)
* Backups can be incremental or full.
* Backups can be in a separate compartment from the boot volume.
* They cannot be detached from an instance when they are running.

### Instance Metadata

The instance metadata includes details like the 

### Instance Configuration and Pools
 
An instance configuration encapsulates the configuration for an instance and new instances can be launched from this config and managed as a pool.

* The config includes the base image, shape, metadata, network config and block volume attachments.
* An instance pool can be creatd with the instance config 
* The pool can then be scaled.
  * If you stop an instance of a pool of size 5, the instance is restarted so that there are still 5 instances in the pool.
* When a pool is deleted, all resources - instances, boot volumes, and attached block volumes are deleted.
* An instance config used by at least one pool cannot be deleted.
* If an instance config is updated, then existing instances do not change.
  * New instances will have the updated config
  * If existing instances are terminated, then new ones are created with updated config to maintain the instance count of the pool
* The instance config and the pool need to be in the same conpartment
* If a pool is scaled down, then the oldest instance is terminated first.

### Autoscaling

 Autoscaling uses an instance pool, and scales the pool out or in based on some performance metric.

* Relies on the metrics gathered by the monitoring service.
* Metrics are aggregated to 1min windows and averaged across the instances in the pool
* When 3 consecutive evaluations hit the threshold, an auto scaling event is triggered.
* There is a cool down period after an autoscaling event. 
  * Metrics are evaluated during the cooldown as well.
  * After cool down the instance pool is adjusted again if needed. 