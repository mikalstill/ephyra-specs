..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Initial design of Ephyra
========================

Ephyra is a simple workload migrator for OpenStack.

Problem description
===================

Imagine that you have a pool of Ironic nodes running a Hadoop workload. You would like to do Continuous Deployment of the control plane for this OpenStack deployment, while having a graceful rollback procedure for when the new version of the control plane is broken for some reason. Specifically for this thought experiment, you can assume that only Ironic is supported as a Nova hypervisor in this deployment, and that the workload running on those Ironic nodes can handle partial outages without degrading performance.

The solution we have come to is that you have two OpenStack deployments. In the initial state, the first of these deployments (which we shall call CP1) has all of the Ironic nodes registered with it, and the second deployment (CP2) has none. The Continous Deployment system detects that there is a new version of OpenStack to deploy. It installs that version onto CP2 in whatever manner it desires. Installation of OpenStack is outside the scope of Ephyra, but it is assumed to be automated.

An API call is then made to Ephyra, which manages migrating Ironic nodes from CP1 to CP2. This is done by removing the instance running on each node from Hadoop in turn, shutting down the instance on the node, de-enrolling the node from Ironic in CP1, enrolling the node with Ironic in CP2, booting an instance mapping to that nodes "role" in Hadoop, and then adding the node back to Hadoop. This process is discussed in more detail below.

It is noted that this scheme has strong similarities with Blue / Green deployments (http://martinfowler.com/bliki/BlueGreenDeployment.html) as used in the devops community.

Simplifying assumptions
=======================

* The workload running on the Ironic deployment is resiliant to individual machine failures. Specifically in this case we are assuming Hadoop, but I am sure there are other workloads for which this holds true as well.
* Only one deployment is being run at a time -- for example, if the control plane is being deployed, then no applications running above the control plane are being deployed at that time. This is required so that the application health checking doesn't return false negatives and case an unneeded deployment rollback. This assumption will be removed later.
* Ephyra does not deploy the software running on the control plane (i.e. OpenStack itself). There are plenty of OpenStack installers already, and the user should select whichever one of those they are most comfortable with.

Design goals
============

* All Ephyra operations should be able to be expressed using existing OpenStack APIs.
* Ephyra can have whatever administrative permissions are required to administer the state of Ironic in each deployment.
* Ephyra should store the minimum amount of state viable. Whereever possible, state shall be stored within the Nova and Ironic deployments and inferred by Ephyra (for example, the list of nodes running in CP1 is the list of machines enrolled in CP1's Ironic deployment, not some separate database).
* Whilst Ironic and Hadoop are assumed at this time, that assumption will be watered down over time. To that end, all interations with Hadoop should be abstracted through a plugin in infrastructure. This will assist with testing Ephyra as well.
* It is assumed that Ephyra will need to expose data for presentation in dashboards and so forth, and therefore should export metrics about its operation as completely as is reasonable.

Networking implications
=======================

Each OpenStack control plane will need to allocate IP addresses from its own pool. Thus, workloads running on top of these OpenStack deployments will need to be able to handle having workers in different IP blocks. This would be listed in "simplifying assumptions" above, but is an interesting enough constraint to get called out in its own section.

(more here)

API interface
=============

Ephyra itself shall be API driven, with the following operations being supported:

* Start a deployment
* Stop a deployment
* List current deployments
* Report status of a current deployment
* "Lock deployments" -- stop deployments from occurring until "unlocked". This is useful if you're currently updating the version of software in one of the control planes and want to ensure a migration between control planes is not attempted until your work is done.

Additionally, there is a plugin infrastructure to determine application health. That interface includes:

* Remove a node
* Add a node
* Verify a node's health
* Verify overall application health

Implementation decisions
========================

Ephyra will be written as a series of taskflow atoms built together into a linear flow. The various steps discussed above will be those atoms, with rollback being implemented after whichever number of retries makes sense per step.

Mistral was also considered, but taskflow appears to map better our needs at this time.
