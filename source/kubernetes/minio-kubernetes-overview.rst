.. _minio-kubernetes:

=======================
MinIO Kubernetes Plugin
=======================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 2

Overview
--------

MinIO is a high performance distributed object storage server, designed for
large-scale private cloud infrastructure. Orchestration platforms like
Kubernetes provide perfect cloud-native environment to deploy and scale MinIO.
The :minio-git:`MinIO Kubernetes Operator </minio-operator>` brings native MinIO
support to Kubernetes. 

The :mc:`kubectl minio` plugin brings native support for deploying MinIO
tenants to Kubernetes clusters using the ``kubectl`` CLI. You can use
:mc:`kubectl minio` to deploy a MinIO tenant with little to no interaction
with ``YAML`` configuration files. 

.. image:: /images/Kubernetes-Minio.svg
   :align: center
   :width: 90%
   :class: no-scaled-link
   :alt: Kubernetes Orchestration with the MinIO Operator facilitates automated deployment of MinIO clusters.

:mc:`kubectl minio` builds its interface on top of the
MinIO Kubernetes Operator. Visit the
:minio-git:`MinIO Operator </minio-operator>` Github repository to follow
ongoing development on the Operator and Plugin.

Installation
------------

**Prerequisite**

Install the `krew <https://github.com/kubernetes-sigs/krew>`__ ``kubectl`` 
plugin manager using the `documented installation procedure
<https://krew.sigs.k8s.io/docs/user-guide/setup/install/>`__. 

Install Using ``krew``
~~~~~~~~~~~~~~~~~~~~~~

Run the following command to install :mc:`kubectl minio` using ``krew``:

.. code-block:: shell
   :class: copyable

   kubectl krew update
   kubectl krew install minio

Update Using ``krew``
~~~~~~~~~~~~~~~~~~~~~

Run the following command to update :mc:`kubectl minio`:

.. code-block:: shell
   :class: copyable

   kubectl krew upgrade

Deploy a MinIO Tenant
---------------------

The following procedure creates a MinIO tenant using the
:mc:`kubectl minio` plugin.

1) Create a Namespace for the MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the ``kubectl create namespace`` command to create a namespace for
the MinIO Tenant:

.. code-block:: shell
   :class: copyable

   kubectl create namespace minio-tenant-1

2) Configure the Persistent Volumes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a :kube-docs:`Persistent Volume (PV) <concepts/storage/volumes/>`
for each drive on each node. 

MinIO recommends using :kube-docs:`local <concepts/storage/volumes/#local>` PVs
to ensure best performance and operations. For example, given a Kubernetes
cluster with 4 Nodes with 4 locally attached drives each, create a total of 16
``local`` ``PVs``. Configure the ``nodeAffinity` for each ``PV`` to reflect
the node on which the physical drive is installed.

- Change ``persistentVolumeReclaimPolicy`` to ``Retain``
  if you want to keep data on disk after deleting the MinIO tenant or its
  associated Persistent Volume Claims (``PVC``). 

- Change ``storageClassName`` to ``default``.

Issue the ``kubectl get PV`` command to validate the created PVs:

.. code-block:: shell
   :class: copyable

   kubectl get PV

3) Initialize the MinIO Operator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:mc:`kubectl minio` requires the MinIO Operator. Use the
:mc-cmd:`kubectl minio init` command to initialize the MinIO Operator:

.. code-block:: shell
   :class: copyable

   kubectl minio init --namespace-to-watch minio-tenant-1

By default the MinIO operator deploys to the ``minio-operator`` namespace.
You can override the namespace with 

4) Create the MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`kubectl minio tenant create` command to create the MinIO
tenant. The following example creates a 4-node MinIO deployment with a
total capacity of 16Ti across 16 drives.

.. code-block:: shell
   :class: copyable

   kubectl minio tenant create    \
     --name      minio-tenant-1   \
     --servers   4                \
     --volumes   16               \
     --capacity  16Ti             \
     --namespace minio-tenant-1

The following table explains each argument specified to the command:

.. list-table::
   :header-rows: 1
   :widths: 30 70
   :width: 100%

   * - Argument
     - Description

   * - :mc-cmd-option:`~kubectl minio tenant create name`
     - The name of the MinIO Tenant which the command creates.

   * - :mc-cmd-option:`~kubectl minio tenant create servers`
     - The number of :mc:`minio` servers to deploy across the Kubernetes 
       cluster.

   * - :mc-cmd-option:`~kubectl minio tenant create volumes`
     - The number of volumes in the cluster. :mc:`kubectl minio` determines the
       number of volumes per server by dividing ``volumes`` by ``servers``.

   * - :mc-cmd-option:`~kubectl minio tenant create capacity`
     - The total capacity of the cluster. :mc:`kubectl minio` determines the 
       capacity of each volume by dividing ``capacity`` by ``volumes``.

   * - :mc-cmd-option:`~kubectl minio tenant create namespace`
     - The Kubernetes namespace in which to deploy the MinIO Tenant.

If :mc-cmd:`kubectl minio tenant create` succeeds in creating the MinIO Tenant,
the command outputs connection information to the terminal. The output includes
the :kube-docs:`Service <concepts/services-networking/service>` address for the
tenant.

5) Connect to the MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connect to the MinIO Tenant using the connection information returned by 
:mc-cmd:`kubectl minio tenant create`. The following example configures
:mc:`mc` to connect to the tenant:

.. code-block:: shell
   :class: copyable

   

MinIO Kubernetes Plugin Syntax
------------------------------

.. mc:: kubectl minio

Create a MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: tenant create
   :fullpath:

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio tenant create  \
        --names    NAME            \
        --servers  SERVERS         \
        --volumes  VOLUMES         \
        --capacity CAPACITY        \
        [OPTIONAL_FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: name
      :option:

      *Required*

      The name of the MinIO tenant which the command creates. The
      name *must* be unique in the :mc-cmd:`~mc tenant create namespace`.

   .. mc-cmd:: servers
      :option:

      *Required*

      The number of :mc:`minio` servers to deploy on the Kubernetes cluster.
      
      Ensure that the specified number of 
      :mc-cmd-option:`~kubectl minio tenant create servers` does *not*
      exceed the number of nodes in the Kubernetes cluster. MinIO strongly
      recommends sizing the cluster to have one node per MinIO server.

   .. mc-cmd:: volumes
      :option:

      *Required*

      The number of volumes in the MinIO tenant. :mc:`kubectl minio`
      generates one Persistent Volume Claim (``PVC``) for each volume.
      :mc:`kubectl minio` divides the 
      :mc-cmd-option:`~kubectl minio tenant create capacity` by the number of
      volumes to determine the amount of ``resources.requests.storage`` to
      set for each ``PVC``.
      
      :mc:`kubectl minio` determines
      the number of ``PVC`` to associate to each :mc:`minio` server by dividing
      :mc-cmd-option:`~kubectl minio tenant create volumes` by 
      :mc-cmd-option:`~kubectl minio tenant create servers`.

      :mc:`kubectl minio` also configures each ``PVC`` with node-aware
      selectors, such that the :mc:`minio` server process uses ``PVC``
      which correspond to a ``local`` Persistent Volume (``PV``) on the 
      same node running that process. This ensures that each process
      uses local disks for optimal performance.

      If the specified number of volumes exceeds the number of 
      ``PV`` available on the cluster, :mc:`kubectl minio tenant create`
      hangs and waits until the required ``PV`` exist.

   .. mc-cmd:: capacity
      :option:

      *Required*

      The total capacity of the MinIO tenant. :mc:`kubectl minio` divides
      the capacity by the number of
      :mc-cmd-option:`~kubectl minio tenant create volumes` to determine the 
      amount of ``resources.requests.storage`` to set for each
      Persistent Volume Claim (``PVC``).

      If the existing Persistent Volumes (``PV``) in the cluster cannot
      satisfy the requested storage, :mc:`kubectl minio tenant create`
      hangs and waits until the required storage exists.

   .. mc-cmd:: namespace
      :option:

      The namespace in which to create the MinIO Tenant. 

      Defaults to ``minio``.

   .. mc-cmd:: kes-config
      :option:

      The name of the Kubernetes Secret which contains the 
      MinIO Key Encryption Service (KES) configuration.

   .. mc-cmd:: output
      :option:

      Outputs the generated ``YAML`` objects to ``STDOUT`` for further
      customization. 
      
      :mc-cmd-option:`~kubectl minio tenant create output` does 
      **not** create the MinIO Tenant. Use ``kubectl apply -f <FILE>`` to
      manually create the MinIO tenant using the generated file.

Expand a MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: tenant expand
   :fullpath:

   Adds a new zone to an existing MinIO Tenant.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio tenant expand  \
        --names    NAME            \
        --servers  SERVERS         \
        --volumes  VOLUMES         \
        --capacity CAPACITY        \
        [OPTIONAL_FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: name
      :option:

      *Required*

      The name of the MinIO Tenant which the command expands.

   .. mc-cmd:: servers
      :option:

      *Required*

      The number of :mc:`minio` servers to deploy in the new MinIO Tenant zone.
      
      Ensure that the specified number of :mc-cmd-option:`~kubectl minio tenant
      create servers` does *not* exceed the number of unused nodes in the
      Kubernetes cluster. MinIO strongly recommends sizing the cluster to have
      one node per MinIO server in the new zone.

   .. mc-cmd:: volumes
      :option:

      *Required*

      The number of volumes in the new MinIO Tenant zone. :mc:`kubectl minio`
      generates one Persistent Volume Claim (``PVC``) for each volume.
      :mc:`kubectl minio` divides the 
      :mc-cmd-option:`~kubectl minio tenant create capacity` by the number of
      volumes to determine the amount of ``resources.requests.storage`` to
      set for each ``PVC``.
      
      :mc:`kubectl minio` determines
      the number of ``PVC`` to associate to each :mc:`minio` server by dividing
      :mc-cmd-option:`~kubectl minio tenant create volumes` by 
      :mc-cmd-option:`~kubectl minio tenant create servers`.

      :mc:`kubectl minio` also configures each ``PVC`` with node-aware
      selectors, such that the :mc:`minio` server process uses ``PVC``
      which correspond to a ``local`` Persistent Volume (``PV``) on the 
      same node running that process. This ensures that each process
      uses local disks for optimal performance.

      If the specified number of volumes exceeds the number of 
      ``PV`` available on the cluster, :mc:`kubectl minio tenant create`
      hangs and waits until the required ``PV`` exist.

   .. mc-cmd:: capacity
      :option:

      *Required*

      The total capacity of the new MinIO Tenant zone. :mc:`kubectl minio` divides
      the capacity by the number of
      :mc-cmd-option:`~kubectl minio tenant create volumes` to determine the 
      amount of ``resources.requests.storage`` to set for each
      Persistent Volume Claim (``PVC``).

      If the existing Persistent Volumes (``PV``) in the cluster cannot
      satisfy the requested storage, :mc:`kubectl minio tenant create`
      hangs and waits until the required storage exists.

   .. mc-cmd:: namespace
      :option:

      The namespace in which to create the new MinIO Tenant zone. 

      Defaults to ``minio``.

   .. mc-cmd:: output
      :option:

      Outputs the generated ``YAML`` objects to ``STDOUT`` for further
      customization. 
      
      :mc-cmd-option:`~kubectl minio tenant create output` does **not** create
      the new MinIO Tenant zone. Use ``kubectl apply -f <FILE>`` to manually
      create the MinIO tenant using the generated file.

Get MinIO Tenant Zones
~~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: tenant info
   :fullpath:

   Lists all existing MinIO zones in a MinIO Tenant.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio tenant info --names NAME [OPTIONAL_FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: name
      :option:

      *Required*

      The name of the MinIO Tenant for which the command returns the
      existing zones.

   .. mc-cmd:: namespace
      :option:

      The namespace in which to look for the MinIO Tenant.

      Defaults to ``minio``.

.. mc-cmd:: tenant upgrade
   :fullpath:

   Upgrades the :mc:`minio` server Docker image used by the MinIO Tenant.
   
   .. important::

      MinIO upgrades *all* :mc:`minio` server processes at once. This may
      result in a brief period of downtime if a majority (``n/2-1``) of 
      servers are offline at the same time.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio tenant upgrade --names NAME [OPTIONAL_FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: name
      :option:

      *Required*

      The name of the MinIO Tenant which the command updates.

   .. mc-cmd:: namespace
      :option:

      The namespace in which to look for the MinIO Tenant.

      Defaults to ``minio``.

Delete a MinIO Tenant
~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: tenant delete
   :fullpath:

   Deletes the MinIO Tenant.
   
   .. warning::

      If the underlying Persistent Volumes (``PV``) were created with
      a reclaim policy of ``recycle`` or ``delete``, deleting the MinIO
      Tenant results in complete loss of all objects stored on the tenant.

      Ensure you have performed all due diligence in confirming the safety of
      any data on the MinIO Tenant prior to deletion.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio tenant delete --names NAME [OPTIONAL_FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: name
      :option:

      *Required*

      The name of the MinIO Tenant to delete.

   .. mc-cmd:: namespace
      :option:

      The namespace in which to look for the MinIO Tenant.

      Defaults to ``minio``.

Create the MinIO Operator
~~~~~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: init
   :fullpath:

   Initializes the MinIO Operator. :mc:`kubectl minio` requires the operator for
   core functionality.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio init [FLAGS]

   The command supports the following arguments:

   .. mc-cmd:: image
      :option:

      The image to use for deploying the operator. 
      Defaults to the :minio-git:`latest release of the operator
      <minio/operator/releases/latest>`:

      ``minio/k8s-operator:latest``
      
      
      The current latest release is |minio-operator-release|

   .. mc-cmd:: namespace
      :option:

      The namespace into which to deploy the operator.

      Defaults to ``minio-operator``.

   .. mc-cmd:: cluster-domain
      :option:

      The domain name to use when configuring the DNS hostname of the
      operator. Defaults to ``cluster.local``.

   .. mc-cmd:: namespace-to-watch
      :option:

      The namespace which the operator watches for MinIO tenants.

      Defaults to ``""`` or *all namespaces*.

   .. mc-cmd:: image-pull-secret
      :option:

      Secret key for use with pulling the 
      :mc-cmd-option:`~kubectl minio init image`.

      The MinIO-hosted ``minio/k8s-operator`` image is *not* password protected.
      This option is only required for non-MinIO image sources which are
      password protected.

   .. mc-cmd:: output
      :option:

      Performs a dry run and outputs the generated YAML to ``STDOUT``. Use
      this option to customize the YAML and apply it manually using
      ``kubectl apply -f <FILE>``.

Delete the MinIO Operator
~~~~~~~~~~~~~~~~~~~~~~~~~

.. mc-cmd:: delete
   :fullpath:

   Deletes the MinIO Operator along with all associated resources, 
   including all MinIO Tenant instances in the
   :mc-cmd:`watched namespace <kubectl minio init namespace-to-watch>`.

   .. warning::

      If the underlying Persistent Volumes (``PV``) were created with
      a reclaim policy of ``recycle`` or ``delete``, deleting the MinIO
      Tenant results in complete loss of all objects stored on the tenant.

      Ensure you have performed all due diligence in confirming the safety of
      any data on the MinIO Tenant prior to deletion.

   The command has the following syntax:

   .. code-block:: shell
      :class: copyable

      kubectl minio delete [FLAGS]

   The command accepts the following arguments:

   .. mc-cmd:: namespace
      :option:

      The namespace of the MinIO operator to delete.

      Defaults to ``minio-operator``.