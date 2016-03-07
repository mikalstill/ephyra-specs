..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Initial design of Ephyra
========================

Ephyra is a simple ...

Problem description
===================

Imagine that you have a pool of Ironic nodes running a Hadoop workload. You would like to do Continuous Deployment of the control plane for this OpenStack deployment, while having a graceful rollback procedure for when the new version of the control plane is broken for some reason. Specifically for this thought experiment, you can assume that only Ironic is supported as a Nova hypervisor in this deployment, and that the workload running on those Ironic nodes can handle partial outages without degrading performance.

The solution we have come to is that you have two OpenStack deployments. In the initial state, the first of these deplouments (which we shall call CP1) has all of the Ironic nodes registered with it, and the second deployment (CP2) has none.

The Continous Deployment system detects that there is a new version of OpenStack to deploy. It installs that version onto CP2 in whatever manner it desires. Installation of OpenStack is outside the scope of Ephyra, but it is assumed to be automated.

An API call is then made to Ephyra, which manages migrating Ironic nodes from CP1 to CP2. This is done by marking each node in turn as in maintenance mode in Hadoop, shutting down the instance on the node, de-enrolling the node from Ironic in CP1, enrolling the node with Ironic in CP2, booting an instance mapping to that nodes "role" in Hadoop, and then informing Hadoop that the node is no longer in maintenance mode. This process is discussed in more detail below.

Design goals
============

* All Ephyra operations should be able to be expressed using existing OpenStack APIs.
* Ephyra can have whatever administrative permissions are required to administer the state of Ironic in each deployment.
* Ephyra should store the minimum amount of state viable. Whereever possible, state shall be stored within the Nova and Ironic deployments and inferred by Ephyra (for example, the list of nodes running in CP1 is the list of machines enrolled in CP1's Ironic deployment, not some separate database).
* Whilst Ironic and Hadoop are assumed at this time, that assumption will be watered down over time. To that end, all interations with Hadoop should be abstracted through a plugin in infrastructure. This will assist with testing Ephyra as well.
* It is assumed that Ephyra will need to expose data for presentation in dashboards and so forth, and therefore should export metrics about its operation as completely as is reasonable.
