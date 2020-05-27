..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This manual shows users how to change, add, or revert configurations of CSCs.

.. note::

    This document is intended for operators of the Vera Rubin observatory control system.
    It consolidates information about how to handle CSC configuration and ancillary data.
    After reading this document users should know what to expect when interacting with system component's configuration, how to select a configuration for a component and, be able to create and load new configurations.


.. warning::

    The procedures in this documents are still under development. The functionality referenced here is not in place nor has it been agreed upon as the proper implementation. **For the time being, this document should be considered a proposed implementation.**

.. _section-introduction:

Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing multiple components that must operate in consonant.
In order to achieve reliable operation it is fundamental to guarantee that observers must be able to, at any point, identify what set of configurations a component is using, what sets are available to be loaded, and how to load the configuration they choose.
During commissioning and engineering runs, users will also be interested in building, validating and verifying new sets of configurations.

The process of building new or modifying configurations will vary considerably for each component.
In some cases, the configuration may be a simple set of host name and port that the component connects to.
In other cases, the configuration may require the calculation of complex lookup tables (e.g. M1M3 and M2) or model coefficients (e.g. pointing component).
These may require on-sky time for acquiring the data, and complex software to analyze and derive the configuration products.
All of this data needs to be associated with this configuration in a way that is easily accessible and traceable.

In this tech-note we discuss the basic operational procedures involved in handling CSC configuration.
The document begins with the most simple case of bringing a CSC up with a specific configuration, and eventually details how to updating and testing newly defined configurations.

For more technical details about how CSCs handle configuration, see `tstn-017 <https://tstn-017.lsst.io>`__.

.. _section-configuration-interation:

Configuration Interaction Use-Cases
===================================

Users will interact with configurations in multiple ways.
In many cases, a user/operator will only need to change the configuration that is currently loaded and are not concerned with the contents of the configuration itself.
This section illustrates example use-cases for these types of scenarios.

Discovering available configurations
------------------------------------

The easiest way to get information from a CSC programmatically is by using a Jupyter notebook server.
From a notebook, observers, developers and power users can easily interact with the system through Python.

In order to know which configurations are available for a specific CSC, users can read the ``configurationsAvailable`` event.
This is done using a ``salobj.Remote`` class.

.. code-block:: python

    from lsst.ts import salobj

    domain = salobj.Domain()

    atdome = salobj.Remote(domain, "ATDome")

    await atdome.start_task()

    config_available = await atdome.evt_configurationsAvailable.aget(timeout=5.)

    # This will print the available labels.
    print(config_available.labels)

    # This will print the mapping between label and configuration files.
    print(config_available.mapping)

Often, this can also be accomplished using a high-level class that is designed to interact with a group of CSCs.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    config_available = await atcs.rem.atdome.evt_configurationsAvailable.aget(timeout=5.)

    # This will print the available labels.
    print(config_available.labels)

    # This will print the mapping between label and configuration files.
    print(config_available.mapping)

It is also possible to check this information by querying the EFD or through the CSC summary information interface on LOVE. Examples of how to do this using the LOVE interface will be added when the functionality is ready.

.. TODO: Add example of how to get this information from the EFD and LOVE.

Selecting default configuration
-------------------------------

In most cases, the control packages contain high-level commands to enable all components under their control, and select the default configuration in the process.
An example of this is the ATCS.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    await atcs.enable()

.. It is also possible to perform this action using a ``Script`` in the ``scriptQueue``.
.. There are different ways to launch scripts.
.. From a Jupyter notebook, the user could launch a script by doing the following:

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("auxtel/enable_atcs.py")

    # Wait for script to execute
    await script.done()

.. Another alternative would be to launch the ``Script`` from the LOVE Queue interface.

.. TODO: Add example on how to launch script from LOVE interface

If working with an individual CSC, which as an operator would be a rare occurrence, default CSC configurations are loaded simply by transitioning the CSC via:

.. code-block:: python

    from lsst.ts import salobj

    domain = salobj.Domain()

    atdome = salobj.Remote(domain, "ATDome")

    await atdome.start_task()

    # CSC needs to be in STANDBY state for this to work
    await salobj.set_summary_state(atdome, salobj.State.ENABLED)

.. Similarly, this can be accomplished by using the ``ScriptQueue``, from Jupyter;

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("set_summary_state", config={"data": [("ATDome", "ENABLED")]})

    # Wait for script to execute
    await script.done()

.. Or the LOVE interface.

.. TODO: Add example on how to launch script from LOVE interface


.. _section-configuration-interation_non_default:

Selecting a non-default configuration
-------------------------------------

Selecting non-default configurations via control packages is also possible.
A dictionary is used to override the appropriate configuration labels for each component that needs a non-default configuration.
This example assumes the component of interest is already in the ``STANDBY`` state.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    # ATAOS must be in STANDBY state for this to work. All other CSCs will
    # be configured with the default label
    await atcs.enable(configuration={'ATAOS': 'constant_hex'})

.. From a Jupyter notebook, users can also launch a script by doing the following:

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("auxtel/enable_atcs.py", config={"ATAOS": "constant_hex"})

    # Wait for script to execute
    await script.done()

.. And from the LOVE interface:

Examples of how to do this using the LOVE interface will be added when the functionality is ready.

.. TODO: Add example on how to launch script from LOVE interface

If working with an individual CSC, which as an operator would be a rare occurrence, the ``salobj.Remotes`` class may be more appropriate:

.. code-block:: python

    from lsst.ts import salobj

    d = salobj.Domain()

    atdome = salobj.Remote(d, "ATDome")

    await atdome.start_task()

    await salobj.set_summary_state(
    atdome, salobj.State.ENABLED, configurationToApply="original-install"
    )

.. And to launch a ``Script`` from Jupyter:

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("set_summary_state", config={"data": [("ATDome", "ENABLED", "original-install")]})

    # Wait for script to execute
    await script.done()

.. Or from the LOVE interface:

.. TODO: Add example on how to launch script from LOVE interface


.. _section-configuration-interation_changing_default:

Changing the default configuration
----------------------------------

Changing the default configuration is a more involved endeavor because it entails making a change to the contents of the configuration repository.
Because the repo is under version control, the appropriate steps must be taken.
For this example, let's assume we want to change the default configuration in the ATAOS, which is found in the `ATAOS directory of the ts_config_attcs repo <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS>`__.


#.  Create a JIRA ticket in where the title/description note the change being made.
    Let's assume it creates ticket DM-12345.

#.  Clone the repo and checkout a new branch

    ::

      git clone git@github.com:lsst-ts/ts_config_attcs.git
      git checkout -b tickets/DM-12345

    Note that the branch name is the word ``tickets/`` appended with the Jira ticket name.

#.  Open the most recent schema version (v2) and modify the contents of ``_labels.yaml``.
    For example, the original version may be:

    ::

        # Labels for recommended configurations; a dictionary of {label: config_file}
        default: _base.yaml
        constant_hex: hex_m1_202003_constant_hex.yaml

    Say you wish to add a new configuration label called, `m1_hex`, and then make the `constant_hex` be the default.
    Because `_base.yaml` is always assigned the label of `default`, the contents of `_base.yaml` must be updated to include the contents of `constant_hex`.
    Then, `m1_hex` and the corresponding filename needs to be appended.
    Therefore, the file would become:

    ::

        # Labels for recommended configurations; a dictionary of {label: config_file}
        default: _base.yaml
        hex_m1: hex_m1_hex_202003.yaml

#.  Add, commit and push the changes, with a commit message.

    ::

      git commit -am "Updated base configuration to use the configuration previously labeled constants_hex with a filename of hex_m1_hex_202003.yaml. Added hex_m1_hex_202003.yaml with label of hex_ml. See DM-12345 for more information."
      git push

    The commit message can add information about what changes are being made and a short description for the reason.
    It is also recommended to explicitly mention the Jira ticket for the work being done as the branch name is lost once the changes are merged to the head branch.

#.  If this is a normal configuration change procedure, then create a pull-request (PR), and have it reviewed, merged and released.

    .. TODO: Fix/Edit/Verify the example below to checkout a local version of
    .. the repo, then set it up accordingly.


#.  Once the new configuration is released it can be made available to the component, which will not automatically see the newly created configuration.
    During normal operations this involves creating a new deployable artifact and updating the deployment to use the new configuration version.

    On-the-fly changes are discouraged but sometimes a reality and are therefore discussed in :ref:`section-configuration-creating-a-new`.

#.  Once the component is re-deployed with the new configuration, bring it from ``STANDBY`` to the ``ENABLED`` state.
    No explicit specification of the configuration is necessary since the default is being selected.
    If a different label is used, the ``configuration`` parameter must be set in the command below (see :ref:`section-configuration-interation_non_default`).

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED)


.. _section-configuration-interaction-traceability:

Finding a previously used configuration
---------------------------------------

In the future, one may want to verify which configuration was being used for a given observation.
Because we often use generic labels (e.g. `default`), and file contents can change with time, creating a robust version controlled system must go beyond simply changing filenames.
For this reason, additional metadata is associated with each configuration, notably the ``url`` and ``version`` parameters in both the ``configurationsAvailable`` and ``configurationApplied`` events.
These parameters are key to ensuring that each configuration is unique, and is traceable to their filename and contents.

The ``url`` parameter simply contains a URL indicating how the CSC connects to its settings (meaning a link to the repository).
The ``version`` parameter is more complicated.
For all CSCs (except the camera?), the ``version`` parameter is a *branch description*\ [#git_version]_ which is automatically generated and populated by the CSCs.
This is what is output by running the following command in a configuration repository (e.g. ``ts_config_latiss``):

.. prompt:: bash

    git describe --all --long --always --dirty --broken

.. [#git_version] The option ``--broken`` was introduced in git 2.13.7

An example output is, ``heads/develop-0-gc89ef1a``.
The repository branch (or tag) name forms the first part of the branch description.
This first part contain individual identifiers and can change rapidly.
It may take any form necessary to convey the appropriate information.
The last 7 characters (``c89ef1a``) is the hash of the commit of the loaded configuration file.
Users can find this commit by navigating to the repository on github, searching for the commit hash, then
clicking on the "commits" section of the search results, as shown in :ref:`the screenshot below <fig-commit-tracing>`.

.. figure:: /_static/tracing_a_commit_on_github.jpg
    :name: fig-commit-tracing

    Using the ``version`` output in the ``configurationApplied`` event, it is possible to traceback the repo to the configuration that was loaded.


Exceptions
----------
TBR.

.. TODO: Complete this section

.. _section-configuration-creating-a-new:

Creating a new configuration
============================

The process to derive new configuration parameters will vary considerably from component to component.
In some cases, the configuration is simple enough that a change may involve simply replacing an IP or hostname value, a routine filter swap on an instrument or updating the limits to an axis range due to some evolving condition.
On the other hand, deriving new parameters may involve generating complex LUTs that may require on sky observations and detailed data analysis.

Following is a detail of each step of the process to generate a new configuration and update it for CSCs written in salobj.
For other components, see the exception section below.


#.  Create a Jira ticket to track the work being done (e.g. DM-12345).
    If details or discussions are needed they can done using the Jira tickets itself.

    .. prompt:: bash

        git clone git@github.com:lsst-ts/ts_config_attcs.git
        git checkout -b tickets/DM-12345


#.  Execute the work needed to derive the new configuration parameter(s).

    As mentioned above, in some cases, the process may be straightforward, consisting simply of replacing the values of a set of parameters with given values (e.g., swapping filters).
    In these cases, this step will be simply verifying any required work was performed and continuing to the next step.
    Jira should be used to track those activities.

    The Jira ticket should also be used to track the work done on those cases where a more involved analysis is required, e.g., in dome and/or on sky data acquisition, EFD queries, data processing etc.
    Any ancillary software or data product required during this process should be properly managed using git.
    When working with Telescope and Site components, any software required during this process should be stored in a git repository in `T&S GitHub organization <https://github.com/orgs/lsst-ts>`__, and should follow the standard `T&S development workflow guidelines <https://tssw-developer.lsst.io>`__.
    This includes, but is not limited to, EFD queries, Jupyter notebooks, other data analysis routines (regardless of the programming language) and so on.
    The preferred location for storing Jupyter notebooks is the `ts_notebooks <https://github.com/lsst-ts/ts_notebooks>`__ repository.

    ..    Details on how to deals with Camera and DM components will be given in the
    ..    future.

    Any intermediate data product(s) generated in the process should also be stored in the `git Large File Storage <https://developer.lsst.io/git/git-lfs.html>`__  or, if size permits, with the software repository itself.

#.  Edit/Add/Replace the configuration file(s) or add a new file(s) to host the new configuration in the CSC configuration directory.

        - Ideally the name of the file should reflect the purpose of change, dates can also be used as well.
          Old configuration files can be kept in the repo if they still represent valid configurations otherwise, they should be removed.
          Note, though, that they will still remain available on previous versions in the git repo, enabling historical comparison.

#.  Add a (commented out) description in the file detailing where any auxiliary data may be stored, the Jira ticket number used to create the file, and the reason for creating the configuration, such as in `this example <https://tstn-017.lsst.io/v/PREOPS-27/_downloads/ATSpectrograph_example_config.yaml>`__, with the appropriate information.

#.  Modify the configuration labels to either re-use a previous label to map to the new configuration (preferred) or create a new label for the new configuration.

        - For Salobj CSCs, this is done by editing the ``_labels.yaml`` file.

#.  Add, commit and push the changes, with a commit message.

    .. prompt:: bash

        git commit -am "Add new LUTs for ATAOS (file 20200512-configuration.yaml) based on data taken on 20200512. Check DM-12345 for more information."
        git push

#.  Test the new configuration on the CSC.
    If this requires in-dome or on-sky testing, make sure the test is properly documented in a technote and/or Jira ticket.
    To make the configuration available on a running CSC check :ref:`section-on-the-fly-config`.

#.  Create pull request(s) (PRs), with evidence that the  configuration is tested, verified and documented.

    PRs must be created for all repositories that where modified during the process, including, but not limited to, the configuration repository, ancillary software and documentation.

    The PRs will follow the standard review procedure.
    Once the they are approved, merged and released the new configuration becomes official and can be deployed.

.. _section-on-the-fly-config:

On-the-fly changes
------------------

During the process of creating a new configuration (:ref:`section-configuration-creating-a-new`) or during a commissioning/engineering run, it may be necessary to make a new configuration available to a running CSC for testing without rebuilding/re-deploying the component.
In these cases, the user should also create a Jira ticket (or work out of an existing ticket) to document the occurrence.

Following are the steps to make a new configuration available to a running CSC:

#.  If the configuration is not already created and pushed to GitHub, follow steps 1 to 5 in :ref:`section-configuration-creating-a-new`.
#.  Make sure the CSC in in ``STANDBY`` state, which can be accomplished using the following command.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.STANDBY)

#.  Login to the where the CSC is running.
    The procedure will vary depending on how the CSC is deployed.
    Most Telescope and Site components are deployed on containers using Kubernetes (k8s).
    For CSCs that are not running on a container, you should be able to login to the host machine with ``ssh`` and continue with the procedure (go to step 3).
    A provisory list of IPs can be found in `confluence <https://confluence.lsstcorp.org/x/qw6SBg>`.
    For details about the deployment system see the `deployment documentation <https://tstn-019.lsst.io>`_.

    The procedure to access containerized components is as follows:

    #.  Log in to the rancher service at https://rancher.ls.lsst.org.
        You will need special authorization to acquire an account on that service.

        .. warning::

            This service is responsible for managing the deployment of the entire system.
            Make sure you follow the procedure exactly.
            If you are in doubt about an operation make sure you verify it with knowledgeable personnel.

    #.  Once logged in, you will be presented with the :ref:`list of available k8s clusters <fig-rancher-page-1>`.

        .. figure:: /_static/rancher-page-1.png
          :name: fig-rancher-page-1
          :target: ../_images/rancher-page-1.png
          :alt: clusters

          List of Kubernetes clusters.
          At the time of this writing, the only cluster available was kueyen, the commissioning cluster at the base facility in Chile.

        Click on the name of the cluster where the CSC you want to modify is running.
        If it is a summit operation, the name of the cluster will be ``andes``.
        After selecting the cluster, you will be redirected to the :ref:`cluster dashboard <fig-cluster-dashboard>`.

        .. figure:: /_static/cluster-dashboard.png
           :name: fig-cluster-dashboard
           :target: ../_images/cluster-dashboard.png
           :alt: cluster dashboard

           Cluster dashboard.

    #.  On the top right corner of the :ref:`cluster dashboard <fig-cluster-dashboard>`, there is a button with ``Launch kubectl``.
        This will open an interactive session on you browser that will allow you to interact with the k8s cluster you selected.
        If you are knowledgeable about k8s you can also download the ``Kubeconfig file`` and login to the cluster from your own computer.

        .. warning::

            **Do not** download the ``Kubeconfig file`` unless you really know what you are doing.
            This file contains access and credential information that would allow users direct access to the k8s cluster.
    #.  Once you select ``Launch kubectl`` you will be redirected to a :ref:`Shell <fig-k8s-shell>` connected directly to the selected k8s cluster.

        .. figure:: /_static/k8s-shell.png
          :name: fig-k8s-shell
          :target: ../_images/k8s-shell.png
          :alt: kubectl shell

          Kubectl shell.

    #.  Use the following command to discover the container running the CSC :

        .. prompt:: bash

          kubectl get pods -n cscs

        This will list all the CSCs "pods" which are, basically, the running containers.
        The name of the CSC will be part of the pod name and should be easy to identify.

    #.  Connect to the running pod:

        .. prompt:: bash

          kubectl exec -it -n cscs <pod-name> -- /bin/bash

        Make sure to replace ``<pod-name>`` with the name of the pod for that CSC.

#.  Once inside the CSC host, go to the location where the configuration is installed.
    This information can be found in the CSC documentation or in the `deployment documentation`_.
    You should be able to use regular linux command line commands (e.g. ``ls`` and ``cd``).
#.  Once in the configuration package, update the git repository and checkout the branch with the new configuration:

    .. prompt:: bash

      git fetch --all
      git checkout -b tickets/DM-12345

#.  Once the branch is updated you can re-enable the component to load the new configuration.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED)

The ``version`` attribute in the ``configurationsAvailable`` event would reflect that change with something like:

::

  version: heads/tickets/DM-12345-0-g79e2257

Note that it would be possible to track the configuration in the future, even if the branch is removed from the repository, by using the commit hash (``g79e2257``).

.. _section-in-line-config:

In-line changes
---------------

During commissioning, we anticipate that there will be situations where quick configuration changes need to be implemented and tested.
In these cases, working out of a local branch and going over the :ref:`section-on-the-fly-config` process may result in the loss of on-sky time.
To ensure the work/changes is tracked it is still recommended that the user create a Jira ticket (or work out of an existing ticket) to document the occurrence.
Then, instead of checking out the repository locally, the user can work out of the deployed CSC configuration directly in the host.

To do this, perform the following procedure:

#.  Verify (or transition) the CSC in in ``STANDBY`` state.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.STANDBY)

#.  Login to the where the CSC is running.
    The procedure will vary depending on how the CSC is deployed.
    For containerized components, you can find details on how to do that in the `deployment documentation <https://tstn-019.lsst.io>`_.
#.  Once inside the CSC host, go to the location where the configuration is installed.
    This information can be found in the CSC documentation or in the `deployment documentation`_.
#.  Create a local branch to work on.

    .. prompt:: bash

      git checkout tickets/DM-12345

#.  Use the available text editors (``vim`` and ``emacs`` are usually made available) to edit the configurations.
#.  Once the configurations are edited and saved, re-enable the component.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED)

It is important to create a branch in place to work on and, later, commit-push to the repository and continue with the process afterwards.

.. warning::

    Users must be aware that failing to commit-push changes done in line may result in loss of information and traceability.
    Therefore, this procedure should be reserved only for critical situations.

Transient labels with Jira ticket numbers may be used for developing new configurations.
They should be moved to standard type labels at the earliest opportunity.

Note that when you connect to the computer running a CSC and edits the configuration directly, the ``version`` parameter reflect that change with something like:

::

  version: heads/tickets/DM-12345-0-g79e2257-dirty

When this happens, it prevents us from precisely identifying what configuration was used.
In this case, the preferred solution is to use :ref:`section-on-the-fly-config` to ensure traceability is not lost, at the expense of a couple extra minutes.

Exceptions
----------

The following require different procedures to create/modify a configuration

- :ref:`Main and Auxiliary Telescope Pointing Components <section-pointing-component>`
- :ref:`M2 <section-m2>`
- :ref:`ATMCS and ATPneumatics <section-atmcs-atpneumatics>`


.. _section-appendix-configuration-location:

Appendix I: Configuration location for CSCs
===========================================

.. note:: This appendix will contain a table relating the CSC to the location of its configuration repository


.. _updating-deployed-csc:

Appendix II: Updating Deployed CSCs
===================================

.. TODO: Example where you change code inside a container (scriptQueue)

.. TODO: Example where you deploy a new container (scriptQueue)


.. Important::

    Needs completing. Might be better to have this as a separate document.

.. _updating-control-packages:

Appendix III: Updating Control Packages
=======================================

TBD

.. _section-appendix-configuration-non-salObj:

Appendix IV: Creating Configurations for non-salObj CSCs
=========================================================

This appendix details the require procedures to produce configuration files for specific CSCs.

.. _section-pointing-component:

Pointing Component
------------------

The pointing component has a configuration file that resides with the code base which, in itself, also defines a couple different files (e.g. pointing model).
Nevertheless, the CSC is not developed to be a configurable CSC, meaning it does not accept a ``configurationToApply`` value to switch between different configurations and does not output the required events.

The CSC is being developed by Observatory Sciences using C++.

.. Important::

    PROCEDURE TO BE ADDED

.. _section-m2:

M2
--

.. Important::

    PROCEDURE TO BE ADDED

.. _section-atmcs-atpneumatics:

ATMCS and ATPneumatics
----------------------

.. Important::

    PROCEDURE TO BE ADDED

.. _section-non-configurable-cscs:

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all.
Examples are sparse in our current architecture but, the from Salobj point of view, a CSC can be developed on top of a ``BaseCSC`` which makes it a non-configurable component.

A non-configurable CSC will ignore the ``configurationToApply`` attribute of the ``start`` command, as it does not contain any true meaning to it.
Likewise these CSCs will not output any of the configuration-related events.

.. Important::

    LIST NON-CONFIGURABLE CSCs


.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
