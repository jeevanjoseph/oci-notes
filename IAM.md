# Identity and Access Management

## Overview

The IAM system in OCI provides all Identity, Authorization and Access management for OCI. Every cloud service provided by OCI is abstracted as a **Resource** or a **resource group**. So a compute instance is a resource identified by it's **OCID**. Identities are abstracted as **Principals**. These principals are grouped using **Groups**. **Policies** define what a **Group** can do, and they are attached to **Compartments** which dictate the scope of the rules contained within the policy.

## Principals

Principals are any idetity on which IAM can act on. There are two types in OCI
* Users
  * standard identities, usually a user or a service account. 
  * has passwords, email addresss and features like API keys and Auth Tokens
  * The default administrator is created when the tenancy is created. This user sets up other users.
  * The notion of **Users** in OCI enforce the principal of least access. 
    * By default, a user has no privileges. For the user to be able to do something :
    * The user needs to be placed in one (or more) **Groups**.
    * The **Group** should have at least one **Policy** that gives the users in the group some privileges.
* Instance Principals
  * Are identities that represent OCI resources.
    * eg: A compute intance that is allowed to make calls to the Object Store. Identity performing the operation is not a user, but a resource.

## Authentication 

* Username / Password
  * Typical user name and password,
* API signing key 
  * This is an RSA key in PEM format
  * Used with CLI or SDKs
  * The key is used to sign the requests made to the OCI API.
    * The request is honored only of the request signature can be verified.
  * Recommeded method of interaction
* Auth Token
  * OCI generated token strings that enable 3rd parties that do not support request signing to authenticate with OCI.

## Authorization

* Groups are authorized to perform tasks by creating policies that enable them to do so.
* Principle of least privilege -> groups do not have any polices by default.
* Users cannot be granted any authorization. Only groups can be authorized through policies
  * A user who does nor belong to any group does not have any permissions.
* The Compartment where a policy is attached dictates who can manage the policy

## Policies

* Syntax
  * `Allow` *subject* `to` *verb* *resource type* `in` *location* `where` *condition*
  * eg: `Allow Administrators to manage all-resources in tenancy`
  * Policies **always** grant privileges, since by default there are no privileges granted.
  * The subject is always a group or dynamic group since polices cannot be attached to Principals 

* Verb
  * **Inspect** - Ability to list the resource. The user can discover the existence of a resource, but cannot get any real details 
  * **Read** - Can *Inspect*, but also get the details of the resource including it's metadata.
  * **Use** - *Read* + Ability to work with the resource. The actions permissible varies by service.
  * **Manage** - full privileges
  * These gives pre defined **permissions** to the subject.
  * Permissions are defined by the service. They are not directly usable by the user.
  * Instead they are mapped to the 4 verbs above. 
  * When using the API (in any form - console, CLI , SDK, HTTP) each API operation will require the user to hav a permission
    * The permissions are mapped to the verbs
    * Users are grouped in to Groups
    * Groups are given Verbs through policies.
    * When *sam.fisher* calls a `LIST VOLUMES` API call, it will succeed only if
      * *sam.fisher* is in a **Group** (perhaps more than one)
      * atleast one of those groups have been granted atleast the `INSPECT` privilege through a **Policy**.
      * The Volumes service has defined a permission called, say, `VOLUME_INSPECT`.
      * The `VOLUME_INSPECT` permission is mapped to the `INSPECT` verb by the service.
      * This mapping causes users in groups that have poilices that grant them atleast the INSPECT verb to be able to to make the API call.
      * *sam.fisher* belongs to a **Group** that has been given access to the verb INSPECT, and VOLUME_INSPECT (the permission the volume service requires users to have) is mapped to the INSPECT verb,  therfore *sam.fisher* has access to the VOLUME_INSPECT permission, and can use the API to list volumes.

* Resource Type
  * Identifies a resource type or family.
    * all-resources - keyword that implies that inclues every resource in the tenancy, including resources that will exist in the future
    * instance-family - Compute resources: instances, images, instance configurations, auto scaling groups
    * database-family - DB systems, db nodes, db homes
    * virtual-network-family - VCN, Subnets, Seclists etc,
    * object-family - Buckets, Objects
    * volume-family - Volumes, volume attachments, volume backupss
    * 

