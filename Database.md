# Database

* DB options include VM, BareMetal, RAC and Exa Systems
* Advantages include
  * One click deployment of RAC and DataGuard
  * dynamic scaling of storage and CPU
  * automated provisioning, patching, backup and restore
* Security
  * Resource level security and audit from OCI IAM
  * DB level security from Encryption (Transparent Data Encryption), RMAN and block volume encryption

### VM DB System

* Avaialable as single instance or 2-node RAC (only 2 node is avaialble)
* Number of CPU cores cannot scale dynamically
* There is a single DB home for the entire DB system. So there can only be one version of the DB in the DB system.
* Always uses block storage (only standard shapes available, no denseIO or HighIO shapes)
  * Can start small and go up tgo 40TB
  * This enables storage to grow much more than Baremetal where storage is limited to the attached NVMe.
  * For 2 node rac, the block storage is shared between the two instances.
  * Block storage backup and restore features can be used.
* ASM on VM DB uses the triple mirroring provided by the block storage
* ASM also uses different block storage volumes for DATA and RECO 
* Storage failures are monitored by OCI
* NO DataGuard

### BareMetal DB System

* Only single node config. If node fails, a new one can be spun up from the storage backups.
  * No RAC support
* Isolation of the baremetal isntance
* X7 has 52 cores, 786GB ram, 51TB(raw) storage and 25GBps networking
  * Ideal for workloads that require raw performance and high bandwidth with lower storage than VM
* CPUs can be scaled.
  * You always pay for all cores in the baremetal server.
  * DB license cost is dymanic and you pay only for the cores you use.
* Only DenseIO shapes available.
* The DB edition is selected at creation and cannot be changed.
  * Each node can have its own DB home, which uses a different version of the DB
* Data storage capacity depends on the shape as well as the mirroring options.
* DataGuard can be used to mirror the data within the same AD or another AD.
  * Cannot cross regions
  * Primary way to scale/HA config
  * Only one standby per primary
  * Standby can be used for queries and other reads like reports. Backup can use standby in case of Active DataGuard
  * Two modes
    * Switchover - planned role reversal secondary-> primary. No DB reinstantiation. Switchover is manual.
    * Failover - Unplanned. Flashback database used to reinstantiate original primary. Failover can be automated
  * Best Practice
    * Primary, Secondary and Observer needs to be deployed to different ADs.
    * For automated failover - can be based on DB failure, or other health check conditions.
    * Can set only Maximum Performance in the console. API supports Maximum availability as well as Maximum Protection
* ASM manages the mirroring of data across local NVMe disks.
* ASM manages DATA and RECO as separate partitions on the NVMe
* Failures are monitored and customer is alerted.

### ExaData DB System

* The gold standard
* Scale both CPUs dynamically. Storage cannot be scaled
  * Scaling from 1/4 rack to 1/2 rack to full rack requires data backup and restoration on new rack 
* Complete isolation and no over provisioning
* Managed by Oracle experts
  * Oracle manages and monitors the servers, storage, networking, firmware, hypervisor etc.
* Can specify 0 cores. This provisions your exadata and stops the instance.
* The first month is always charged, then billing is by the hour.
  * New cores added are billed by the hour from the point its added.
* Exa has its own NVMe storage and Spinning disks.
  * Different from baremetal
  * If backups are on exadata, 40% for DATA and 60% for RECO
  * If backups are not on ExaData, 80% for DATA and 20% for RECO
  * After storage is allocated, it can be cahnged by reconfiguring the environment fromscrathc or by submitting and sr
* Data Guard is avaialble in ExaData using the native Oracle tools and utilities

### DB CLI

* command line tool available for VB and BareMetal isntances to perform DB realted opperations
* NOT available on ExaData
  * Backup
  * Credential management
  * Security
  * Patching
  * Recovery
  * DB Home management
  * etc.

#### Patching

* Automated Patch discovery & pre-flight checks
* On demand patching - N-1 patching (one less than the latest patch is available if it has not been applied)
* Exa systems and RAC use rolling patching. Nodes are updated one by one while the other is active.
* For single node BM systems active data guard can be configured to be leveraged by the patch service.
* 2-Step process
  * First pathc the DB system
  * Then patch the DB
* Integrated with IAM - controls who can list and apply patches

#### Backup and Restore

* Backups can be managed through the OCI console and APIs. except Exa
  * Exa requires a backup configuration file to be created.
  * For VM, automatic incremental backup once a day and discarded after 30 days.
    * Fully managed by Oracle. Kept in Oracle object store.
    * Customers cannot list, view or edit these backups as they are in Oracle buckets.
    * Backup window is managed by oracle.
    * backups run between 00:00 and 06:00 in teh timezone of the region of the DB systems.
    * Automatically reties if backup fails.
* Backups are stored in the local storage or Object Store. (Object Store recommended)
* DB can leverage service gateways to access Object Store

### ADW

* Columnar format, generates data summaries automatically, DB gathers stats transparently (part of bulk load)
* Fully Managed through the API or console
  * Provisioning, patching, scaling - all done through the API or Console
  * No tuning. No considerations for parallelism, partitioning or indexing
  * DB params are setup for data warehouse scenarios
  * Sessions, Parallelism, IO bandwidth scales with the number of CPUs
  * User have access to limited set of params like NLS settings
  * Tablespaces are managed by ATP. Users cannot create or modify tablespaces
  * Tables are compresses usign hybrid columnar compression. Cannot be changed or disabled by users.
  * Optimizer stats are gathered during data loads. User can manually trigger as well.
  * Optimizer hints are ignored by default. Users can enable hints explicitly
  * Result cache is enabled by default for all queries.

* To start, simply needs the core count and storage
  * No shapes or fixed units of scale
  * Can scale CPUs and Store while in flight
  * All exisitng tools are supported, and use standart connectivity
  * Service prompts for admin passwod , cpu count and storage. sping up takes a few minutes, and db is open 

* Scaling does not involve downtime.
  * memory, IO bandwidth and concurrency scales with CPU
  * Scaling is instant
  * DB can be closed to save on billing
  * Startup of the database is instant. 

* The data is encrypted at rest.
* Connectivity is only through cert based auth.
  * The DB generates the keys and provides a wallet or a JavaKeyStore with the certs preloaded.
  * The keys are passphrase enabled.

* Pre-defined service names
  * Provide different performance for the clients.
  * __High__
    * dedicates most resources to run the queries.
    * supports fewest number of concurrent sql executions = 3; reguardless of the number of CPUs
    * Each SQL can use all CPUs avaialable
  * __Medium__
    * dedicates more reosurces than low, but less than high.
    * SQL can use multiple CPUs 
    * number of concurrent queries scale linearly with the number of cpus.
  * __Low__
    * Least resources per query
    * max # of concurrent queries = 100x number of CPUs
    * each query can use only a single CPU
    * Use for data replication -> golden gate

#### Using the ADW
* grant `DWROLE` to users ; No need to specify table spaces for users (ADW does not give access to table spaces)
* Loading data
  * Traditional - good for low volume data
    * SQL*Loader - works from a client machine where data resides
    * ETL scripts - scripts that connect to the database and use DML to insert data
  * Object Storage - best for large volumes of data
    * Stage data in an object storage bucket and load directly
    * New PL/SQL package to directly import data from object storage
    * Also supports AWS S3, Azure and OCI-C object storage
* New `DBMS_CLOUD` Package
  * makes it easier to work with coud services
  * No need to create external tables
  * can define and manage credentials to a cloud service with `create_credential`
  * `copy_data` procudure can point to an object store bucket and directly import data.
    * leveages the credentials created with `create_credential` to auth with the service.
  * `user_load_operations` and `dba_load_operations` tables can help troubleshoot data load issues.
  * `dbms_cloud.create_external_table` can point to an object store bucket and directly query it as an external table.

* Built in SQL worksheet and notebook
  * Alternative to SQL Developer
  * based on apache zeppelin
