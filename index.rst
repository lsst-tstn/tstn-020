..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

    This manual shows users how to change, add, or revert configurations of all CSCs.
    This document is intended for operators of the Vera Rubin observatory control system but also contains information pertinent to developers.
    It consolidates information about how to handle CSC configuration and ancillary data.
    
    After reading this document users should know what to expect when interacting with system component's configuration, how to select a configuration for a component, and be able to create and load new configurations.

.. _section-introduction:

Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing multiple components that must operate in consonant.
In order to achieve reliable operation it is fundamental to guarantee that observers must be able to, at any point, identify what set of configurations a component is using, what sets are available to be loaded, and how to load the configuration they choose.
During commissioning and engineering runs, users will also be interested in building, validating and verifying new sets of configurations.

.. Important::

    Most CSCs handle configuration in the same manner, however, there are a few exception to the rule, notably the Camera CSCs.
    These special cases are addressed in `Appendix I: Creating Configurations for non-salObj CSCs`_.

Although the configuration data structures (e.g. schemas) vary considerably between components, the process to update them is standardized.
Being CSC specific, the schemas exist within the CSC code repository as a `config_schema.py` file, as is seen here for the `ATDome CSC <https://github.com/lsst-ts/ts_ATDome/tree/develop/python/lsst/ts/ATDome>`_.
In some cases, the configuration may be a simple set of host name and port that the component connects to.
In other cases, the configuration may require the calculation of complex lookup tables (e.g. M1M3 and M2) or model coefficients (e.g. pointing component).
These may require on-sky time for acquiring the data, and complex software to analyze and derive the configuration products.
All of this data needs to be associated with this configuration in a way that is easily accessible and traceable.

In this tech-note we discuss the basic organization of configurations and the operational procedures involved in handling CSC configuration.
The document begins with the most simple case of bringing a CSC up with a specific configuration, and eventually details how to update and test newly defined configurations.


Configuration Interface Summary
===============================

The standard interface for CSCs is defined in :lse:`209` and is the official reference/requirements document.
This section provides a very brief overview of the key topics related to configuration that are described in further detail in the sections below.

Configuration-related interactions with CSCs happens via two main events (``configurationsAvailable`` and ``configurationsApplied``) and a single parameter as part of the ``start`` command (``configurationOverride``).
When in the STANDBY state, the ``configurationsAvailable`` event is published which includes a list of non-default (custom) configurations as well as information required to track available configurations (both default and custom) to a specific git commit.
Under normal operations, site-specific default configurations are automatically loaded.
However, using the ``configurationOverride`` parameter (explained in detail in `Selecting an Override Configuration_` section), it is possible to specify one of the non-default configurations that is listed in the ``configurationsAvailable`` event.
Lastly, the ``configurationsApplied`` event is published when the CSC transitions from the STANDBY state to the DISABLED state (which is on the way to the ENABLED state).
The event contains the information regarding which configuration(s) have been loaded and are in active use by the CSC.

``configurationsAvailable`` Event Description
---------------------------------------------

The ``configurationsAvailable`` event is a Generic event defined in :lse:`209` that is implemented by every configurable CSC.
It contains the following parameters: 

- ``overrides`` - A list of files that contain configuration parameters that will override the default values.
- ``version`` - Contains the version information about the local configuration repository or database.
  For configurations stored in git repositories, the `branch description <branch-description_>`_ is returned.  
- ``url`` - A URL indicating how the CSC connects to its settings.
  It will start with "file:" if it is a clone of a git repo, or the standard URL, if a database.
- ``schemaVersion`` - Indicates the schema version in use (e.g. v3), which is a folder in the configuration repository.

Further information regarding this event is found in the `Discovering Available Override Configurations`_ use-case.


``configurationsApplied`` Event Description
-------------------------------------------

The ``configurationApplied`` event is a Generic event that is implemented by every configurable CSC.
It contains the following parameters: 

- ``configurations`` - A list of configuration files that have been loaded.
- ``version`` - Contains the version information about the loaded configuration repository (the SHA) or database.
  For configurations stored in git repositories the `branch description <branch-description_>`_  is used. 
- ``url`` - A URL indicating how the CSC connected to its settings.
  It will start with "file:" if it is a clone of a git repo, or the standard URL, if a database.
- ``schemaVersion`` - Indicates the schema version selected (e.g. v3), which is a folder in the configuration repository.
- ``otherInfo`` - Contains a list of events used to publish the entire set of configuration parameters loaded by the CSC, and any other pertinent information.
  The CSC is allowed to publish as many events as necessary to convey the necessary information.

Further details and explanation regarding this event is found in the `Finding and Using a Previously Used Configuration`_ section.

Configuration Files and Repositories
====================================

It is important to have the ability to modify configurations without re-deploying components, therefore, configuration files are stored in their own repositories and kept separated from the code.
The configuration repository associated with each CSC is found in the configuration column of the `Master CSC Table`_.
Each of these configuration repositories are organized as follows:

- A single configuration repository may host configurations for multiple CSCs.
- CSC configurations are stored in folders with the CSC name.
- Each CSC folder contains sub-folders corresponding to the versions of the CSC configuration (e.g. ``v1``, ``v2``).
- A folder for a given version contains up to three different **types** of configuration files, all of which are yaml files.
  The three types of configuration files are detailed below and are listed in the order of which they are read:

    #. Initial Configuration: ``_init.yaml``.
        - This **required** file contains all values that are expected to be common to all sites and/or be relatively static in operations.
          This file may contain a complete set of parameters, but is only required to do so if no site-specific configuration file exists.

    #. Site-specific Configuration: ``_summit.yaml``, ``_ncsa.yaml``, ``_base.yaml`` etc.
        - This **optional** file contains contain site specific configuration parameters such as IP addresses and ports.
          Many CSCs have site specific files.
          SalObj determines which site-specific file should be loaded automatically by parsing the ``LSST_DDS_PARTITION_PREFIX`` environment variable
          Between this file and the ``_init.yaml`` file, **the configuration must be fully defined**

    #. Configuration overrides: ``filename.yaml``
        - These **optional** files, referred to as configuration overrides, are only to be used when the values declared in the previous files require changes.
          These files are loaded manually by the users as is demonstrated in the `Selecting an Override Configuration`_ section.

- If a value is specified in more than one of these files, the most recently seen value is used.
  This means that values in the site-specific (``_<site>.yaml``) file override values in the initial file (``_init.yaml``).
  Also, values in the override file (``filename.yaml``) override values populated in the ``_init.yaml`` and ``_<site>.yaml`` files.

Configuration Files for Unit Tests
----------------------------------

Unit or integration tests requiring specific information shall utilize an override file that is specific to the test.
Unit tests must utilize configuration files stored in the ``tests/data/config`` directory, as is done for the `ATDome CSC <https://github.com/lsst-ts/ts_ATDome/tree/develop/tests/data/config>`_.
See the `Salobj documentation <https://ts-salobj.lsst.io>`__ for more details.

.. _section-continuous-monitoring:

Monitoring the Configuration Repository
---------------------------------------

CSCs are required (as per :lse:`209`) to publish ``configurationsAvailable``  event when they transition to ``STANDBY`` state.
However, while in ``STANDBY`` state it is possible for someone to update the available configuration, which would make the information out of sync.
Therefore, while in ``STANDBY`` state, CSCs continuously monitor the configuration repository and update/publish new topics whenever changes are detected.

This monitoring only happens while the CSC is in ``STANDBY`` and should not interfere with any other state.
For instance, when transitioning from ``DISABLE`` to ``STANDBY``, the CSC will not start monitoring until the transition is completed and the command acknowledged.


.. _Master CSC Table: https://ts-xml.lsst.io/#master-csc-table.

.. _section-configuration-interaction:

Starting CSCs with Existing Configurations
==========================================

Users will interact with configurations in multiple ways.
In many cases, a user/operator will only need to change the configuration that is currently loaded and are not concerned with the contents of the configuration itself.
In other cases, the user/operator will need to make a change to file, then immediately reload it.
This section illustrates example use-cases for these types of scenarios.

.. note:: 

    When interacting with CSCs from high-level classes, the standard (PEP8) style guide is adopted.
    This results in command parameters using ``snake_case`` whereas the parameters of CSC commands at the state-machine level use ``camelCase`` to match the XML which is designed to be compliant with DDS naming restrictions.


Selecting the Default Configuration
-----------------------------------

In most cases, users will want to load the default configuration.
The default configuration consists of parameters in the ``_init.yaml`` file and subsequently the ``_<site>.yaml``, if it is present.
These files are loaded automatically when performing state transitions using salobj or any higher-level software.

In most cases, the control packages contain high-level commands to enable all components under their control.
An example of this is the ATCS.
The following example enables all ATCS controlled components using their default configurations.

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

If working with an individual CSC, which should be a special case, default CSC configurations are loaded by directly transitioning the CSC via:

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


If these types of tasks are performed from the LOVE interface, then the same result occurs where the defaults are loaded automatically.

.. TODO: Add example on how to launch script from LOVE interface


Discovering Available Override Configurations
---------------------------------------------

It is also possible to override the defaults using one of the available override configurations.
Determining the overrides available to the CSC can be done in multiple ways, including querying the EFD via Chronograf or through the CSC summary information interface on LOVE (not yet implemented).
Another way to determine which available override configurations are available to the CSC is by using a Jupyter notebook.
From a notebook, observers, developers and power users can easily interact with the system through Python.

In order to know which configurations are available for a specific CSC, users can read the ``configurationsAvailable`` event.
This is done using a ``salobj.Remote`` class.

.. code-block:: python

    from lsst.ts import salobj

    domain = salobj.Domain()

    atdome = salobj.Remote(domain, "ATDome")

    await atdome.start_task()

    config_available = await atdome.evt_configurationsAvailable.aget(timeout=5.)

    # This will print the available filenames.
    print(config_available.configurations)

    # This will print the git hash of the loaded configuration repository
    print(config_available.version)

Often, this can also be accomplished using a high-level class (aka a control package) that is designed to interact with a group of CSCs.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    config_available = await atcs.rem.atdome.evt_configurationsAvailable.aget(timeout=5.)

    # This will print the available filenames.
    print(config_available.configurations)

    # This will print the git hash of the loaded configuration repository
    print(config_available.version)



.. TODO: Add example of how to get this information from the EFD and LOVE.


.. _section-configuration-interaction_non_default:

Selecting an Override Configuration
-----------------------------------

Selecting non-default configurations via high-level control packages is also possible.
These are generally used for circumstances where customization is required, or a fallback from standard functionality is necessary.
For example, if the look-up tables in the default configuration for the ATAOS are causing problems, then we can use this procedure to override the defaults by specifying a configuration file that contains the values from the previous look-up table.

When enabling components using the ATCS class, a dictionary is used to provide the appropriate configuration override files for each component that needs a non-default configuration.
This example assumes the component of interest is already in the ``STANDBY`` state.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    # ATAOS must be in STANDBY state for this to work. All other CSCs will
    # use their default configurations
    await atcs.enable(override={'ATAOS': 'summit_constant_hex.yaml'})

.. From a Jupyter notebook, users can also launch a script by doing the following:

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("auxtel/enable_atcs.py", config={"ATAOS": "summit_constant_hex.yaml"})

    # Wait for script to execute
    await script.done()

.. note::
    
    "Specifying any file ``_`` prefix, such as ``_init.yaml``, via the ``configurationOverride`` parameter will cause the ``START`` command to be rejected.


.. note::
    
   Examples of how to do this using the LOVE interface will be added soon.

.. TODO: Add example on how to launch script from LOVE interface

If working with an individual CSC, which should be a special case, the ``salobj.Remotes`` class may be more appropriate:

.. code-block:: python

    from lsst.ts import salobj

    d = salobj.Domain()

    atdome = salobj.Remote(d, "ATDome")

    await atdome.start_task()

    await salobj.set_summary_state(
    atdome, salobj.State.ENABLED, override="simple_algorithm.yaml"
    )

.. And to launch a ``Script`` from Jupyter:

.. .. code-block:: python

    from lsst.ts.observatory.control import ScriptQueue

    # index = 1 is the MT Queue and index = 2 the AT
    queue = ScriptQueue(index=2)

    await queue.start_task

    script = await queue.add("set_summary_state", config={"data": [("ATDome", "ENABLED", "simple_algorithm.yaml]})

    # Wait for script to execute
    await script.done()

.. Or from the LOVE interface:

.. TODO: Add example on how to launch script from LOVE interface

Modifying or Creating a New Configuration
=========================================

The process to derive new configuration parameters varies considerably from component to component.
In some cases, the configuration is simple enough that a change may involve simply replacing an IP or hostname value, a routine filter swap on an instrument or updating the limits to an axis range due to some evolving condition.
On the other hand, deriving new parameters may involve generating complex LUTs that may require on sky observations and detailed data analysis.

Following is a detail of each step of the process to generate a new configuration and update it for CSCs written in salobj.
For other components, see the exception section below.


#.  Create a Jira ticket to track the work being done (e.g. DM-12345).
    If details or discussions are needed they can done using the Jira ticket itself.
    Then clone the configuration repository and create a new branch corresponding to the Jira ticket number.

    .. prompt:: bash

        git clone git@github.com:lsst-ts/ts_config_attcs.git
        git checkout -b tickets/DM-12345

#.  Execute the work needed to derive the new configuration parameter(s).

    As mentioned above, in some cases, the process may be straightforward, consisting simply of replacing the values of a set of parameters with given values (e.g., swapping filters).
    In these cases, this step will be simply verifying any required work was performed and continuing to the next step.
    Jira should be used to track those activities.

    The Jira ticket should also be used to track the work done on those cases where a more involved analysis is required, e.g., in-dome and/or on-sky data acquisition, EFD queries, data processing etc.
    Any ancillary software or data product required during this process should be properly managed using git.
    When working with Telescope and Site components, any software required during this process should be stored in a git repository in `T&S GitHub organization <https://github.com/orgs/lsst-ts>`__, and should follow the standard `T&S development workflow guidelines <https://tssw-developer.lsst.io>`__.
    This includes, but is not limited to, EFD queries, Jupyter notebooks, other data analysis routines (regardless of the programming language) and so on.
    The preferred location for storing Jupyter notebooks is the `ts_notebooks <https://github.com/lsst-ts/ts_notebooks>`__ repository.
    If the procedure to generate the new configuration requires detailed explanation, a tech-note in tstn repository can be created and linked to the ticket.

    ..    Details on how to deals with Camera and DM components will be given in the
    ..    future.

    Any intermediate data product(s) generated in the process should also be stored in the `git Large File Storage <https://developer.lsst.io/git/git-lfs.html>`__  or, if size permits, with the software repository itself.

#.  Edit/Add/Replace the configuration file(s) in the CSC's configuration directory.

    - If editing the ``_init.yaml`` or a ``_<site>.yaml`` file, the filename must remain unchanged.
    - If editing or adding a configuration override file, the name of the file should reflect the purpose of change.
      Including the date as part of the name is also recommended.
      Old configuration files can be kept in the repo if they still represent valid configurations. Otherwise, they should be removed.
      Note, though, that they will still remain available on previous commits in the git repo, enabling historical comparison.

#.  Fill out the required metadata at the top of the file detailing where any auxiliary data may be stored, the Jira ticket number used to create the file, and the reason for creating the configuration, such as in :download:`this example <_static/ATSpectrograph_example_config.yaml>`.

#.  If you have an environment to do so, such as the standard T&S development container, run the unit tests in the package locally.

#.  Add, commit and push the changes, with a commit message.

    .. prompt:: bash

        git commit -am "Add new LUTs for ATAOS (file 20200512-configuration.yaml) based on data taken on 20200512. Check DM-12345 for more information."
        git push

#. Verify the continuous integration tests pass. If they don't, fix the issue and repeat the previous step.

#.  Test the new configuration on the CSC.
    If this requires in-dome or on-sky testing, then create an annotated alpha release tag.
    Then make sure the test is properly documented in a technote and/or Jira ticket.
    To make the configuration available on a running CSC check `On-the-Fly Configuration Changes`_.

#.  Create pull request(s) (PRs) to have the files reviewed

    You must create PRs for all repositories that were modified during the process, including, but not limited to, the configuration repository, ancillary software and documentation.

    Once the PRs are reviewed and approved, the files can be merged and subsequently tagged.
    The new configuration then becomes official and will be deployed as part of the standard deployment process.



On-the-Fly Configuration Changes
================================

During the process of creating a new configuration (see also `Modifying or Creating a New Configuration`_) or during a commissioning/engineering run, it may be necessary to make a new configuration available to a running CSC. 
Normally, new configurations are only made available when rebuilding/re-deploying the component and associated configuration repository, but this is not always feasible.
Therefore, in cases where on-the-fly configuration changes are required, the following procedure should be followed to make a new configuration available to a running CSC:

#.  If the configuration is not already created and pushed to GitHub, follow steps 1 to 8 in `Modifying or Creating a New Configuration`_.
#.  Create an annotated tag alpha tag following `semantic versioning`_.
    The tag must be created to ensure the heritage is not lost in a forced commit to the branch

    .. prompt:: bash

        git tag -a v1.4.0.alpha.1 -m "Updated focus values based on on-sky tests"

#.  Make sure the CSC in in ``STANDBY`` state, which can be accomplished using the following command.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.STANDBY)

#.  Login to the where the CSC is running.
    The procedure will vary depending on how the CSC is deployed.
    Most Telescope and Site components are deployed on containers using Kubernetes (k8s).
    For CSCs that are not running on a container, you should be able to login to the host machine with ``ssh`` and continue with the procedure (go to step 3).
    A provisional list of IPs can be found in `confluence <https://confluence.lsstcorp.org/x/qw6SBg>`_.
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
#.  Once in the cloned configuration package, update the git repository and checkout the tag with the new configuration:

    .. prompt:: bash

      git fetch --all
      git checkout tags/v1.4.0.alpha.1

    You should see the new tag be pulled and git will tell you that you've changed tags/branches.

#.  Now re-enable the component to load the new configuration.

    If the ``_init.yaml`` or ``_<site>.yaml`` file was modified then use the following:

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED)

    If an override configuration was modified/added, then you must specify it using the ``override`` keyword

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED, override='summit_constant_hex')


The ``version`` attribute in the ``configurationsAvailable`` event would reflect that change with something like:

::

  version: heads/tags/v1.4.0.alpha.1-g79e2257

Note that it would be possible to track the configuration in the future by using the commit hash (``g79e2257``).

.. _semantic versioning: https://semver.org/.

.. _section-in-line-config:

In-line Changes to Loaded Configurations
----------------------------------------

During commissioning, we anticipate that there will be situations where quick configuration changes need to be implemented and tested.
In these cases, working out of a local branch and going over the `On-the-Fly Configuration Changes`_ process may result in the loss of on-sky time.
To ensure the work/changes is tracked it is still recommended that the user create a Jira ticket (or work out of an existing ticket) to document the occurrence.
Then, instead of checking out the repository locally, the user can work out of the deployed CSC configuration directly in the host.

.. warning::

    Users cannot push changes from inside a component and therefore this method will result in a loss of information and traceability.
    Therefore, this procedure should be reserved only for critical situations.

To do this, perform the following procedure:

#.  Verify (or transition) the CSC in in ``STANDBY`` state.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.STANDBY)

#.  Login to the where the CSC is running.
    The procedure will vary depending on how the CSC is deployed.
    For containerized components, you can find details on how to do that in the `deployment documentation <https://tstn-019.lsst.io>`_.
#.  Once inside the CSC host, go to the location where the configuration is installed.
    This information can be found in the CSC documentation or in the `deployment documentation`_.
#.  Create a local branch to work on that corresponds to the Jira ticket mentioned above.

    .. prompt:: bash

      git checkout tickets/DM-12345

#.  Use the available text editors (``vim`` and ``emacs`` are usually made available) to edit the configurations.
#.  Once the configurations are edited and saved, re-enable the component.

    .. code-block:: python

        await salobj.set_summary_state(ataos, salobj.State.ENABLED)


Transient filenames with Jira ticket numbers may be used for developing new configurations.
They should be moved to a more purpose-oriented filename at the earliest opportunity.

As stated in the warning above, these changes cannot be pushed from inside a component and therefore the changes made will result in a loss of information and traceability.
When you connect to the computer running a CSC and edit the configuration directly, the ``version`` parameter reflects that change with something like:

::

  version: heads/tickets/DM-12345-0-g79e2257-dirty

When this happens, it prevents us from precisely identifying what configuration was used.
In this case, the preferred solution is to follow the `On-the-Fly Configuration Changes`_ process to ensure traceability is not lost, at the expense of a few extra minutes.

.. _section-configuration-interaction-traceability:

Finding and Using a Previously Used Configuration
=================================================

In the future, one may want to verify which configuration was being used for a given observation and possibly load the exact same configuration.
Because we often use generic filenames (e.g. `simple_algorithm.yaml`), and file contents can change with time, creating a robust version controlled system must go beyond changing filenames.
For this reason, additional metadata is associated with each configuration, notably the ``url`` and ``version`` parameters in both the ``configurationsAvailable`` and ``configurationApplied`` events.
These parameters are key to ensuring that each configuration is unique, and is traceable to their filename and contents.

The ``url`` parameter contains a URL indicating how the CSC connects to its configuration repository.

.. _branch-description:

The ``version`` parameter is a bit more complicated.
For all CSCs (except possibly the cameras), the ``version`` parameter is a *branch description*\ [#git_version]_ which is automatically generated and populated by the CSCs.
It can be obtained by running the following git command on the command line.

.. prompt:: bash

    git describe --all --long --always --dirty --broken

.. [#git_version] The option ``--broken`` was introduced in git 2.13.7


When running the command in a configuration repository (e.g. ``ts_config_latiss``) the output is, ``heads/develop-0-gc89ef1a``.
The repository branch (or tag) name forms the first part of the branch description.
This first part (``heads/develop-0``) contains individual identifiers that may change.
It may take any form necessary to convey the appropriate information.
The last 7 characters (``c89ef1a``) is the hash of the commit of repository, so all configuration files in that repo correspond to the same hash.
Users can find this commit by navigating to the repository on github, searching for the commit hash, then
clicking on the "commits" section of the search results, as shown in :ref:`the screenshot below <fig-commit-tracing>`.

.. figure:: /_static/tracing_a_commit_on_github.jpg
    :name: fig-commit-tracing

    Using the ``version`` output in the ``configurationApplied`` event, it is possible to trace configuration that was loaded back to a specific commit to the configuration repository.

Once we have identified the hash of the commit file we want to reload, we can do that without having to make any changes to the currently deployed ts_config package.
If we simply want to use the default site-specific configuration for a given CSC, we can specify the commit hash with a preceding colon (``:``) as follows:

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    # ATAOS must be in STANDBY state for this to work. All other CSCs will
    # use their default configurations
    await atcs.enable(configurationOverride={'ATAOS': ':c89ef1a'})

If we also want to specify an override file then we insert the filename before the colon (``:``) as shown below:

.. code-block:: python

    await atcs.enable(configurationOverride={'ATAOS': 'simple_algorithm.yaml:c89ef1a'})


Exceptions
----------

The following require different procedures to create/modify a configuration:

- :ref:`Main and Auxiliary Telescope Pointing Components <section-pointing-component>`
- :ref:`ATMCS and ATPneumatics <section-atmcs-atpneumatics>`
- Camera CSCs

CSC Developer Information
=========================

This section contains information that is primarily of interest to CSC developers.
However, if issues are being encountered when creating new configurations the information may be pertinent.

Rules Regarding Configuration Definitions and Usage
---------------------------------------------------
#.  No default values shall be contained in the configuration schema definition in the CSC repository.

#.  In the configuration repository for the given CSC (e.g `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`_ for the ATDome) there shall be an ``_init.yaml`` file that specifies values that are expected to be common to all sites and/or be relatively static in operations (we intentionally use "_init" instead of "_default").

    - See this :download:`example _init.yaml <_static/_init.yaml>` for the ATSpectrograph CSC.
    - This file is the first configuration file loaded by the CSC.
    - Providing the ``_init.yaml`` file (or any file with a ``_`` prefix) to the ``configurationOverride`` parameter in the ``start`` command (or ``override`` in the ``set_summary_state`` command) will return an error.

#.  Also in the configuration repository for the given CSC, when applicable, are the files corresponding to each site where the CSC is used (e.g. ``_summit.yaml, _ncsa.yaml, _base.yaml``).
    These files contain site-specific configuration parameters such as IP addresses and ports.
    However, if no site-specific parameters exist for the CSC, then the use of this file is not required.
    Items in the ``_<site>.yaml`` file will override values that may have been declared in the ``_init.yaml`` file
    SalObj determines which site-specific file should be loaded automatically by parsing the ``LSST_DDS_PARTITION_PREFIX`` environment variable

    - See this :download:`example _summit.yaml <_static/_summit.yaml>` for the ATSpectrograph CSC.
    - This file is the second configuration file to get loaded by the CSC and will override any previously declared values.
    - Providing the ``_<site>.yaml`` file (or any file with a ``_`` prefix) to the ``configurationOverride`` parameter in the ``start`` command must result in rejecting the command.
    - The combination of the ``_<site>.yaml`` and ``_init.yaml`` files **must fully populate all configuration parameters**.

#.  The override configuration files, if specified using the `configurationOverride` parameter in the ``start`` command, is the third file loaded and will override the values set by the previously configuration files.

    - These files are not expected to be required as part of regular operations and are meant to be used when a non-standard configuration is required.
    - See this :download:`configuration parameter override example file <_static/ATSpectrograph_example_config.yaml>` for the ATSpectrograph CSC.d
    - If an override configuration file is also site-specific, then a prefix should be added indicating which site it belongs with (e.g. ``summit_reduced_stage_travel.yaml``)

#.  Filenames will be checked against a list of forbidden values by the continuous integration tests associated with the configuration repository. 
    Examples include: ``default.yaml``, ``site.yaml``, ``init.yaml``

#.  If a CSC receives a ``start`` command with an empty ``configurationOverride`` parameter, it will load the values in ``_init.yaml`` then the site-specific file (e.g. ``_summit.yaml``, if it exists).

#.  If a CSC receives a ``start`` command with a ``configurationOverride`` parameter equal to a valid filename, it loads the values in ``_init.yaml``, then the site-specific file (e.g. ``_summit.yaml``) if it exists, and lastly the override file.
    An invalid filename will return as a failed command with an appropriate error message saying the file was not readable and no state transition will occur.

#.  The configuration repository shall not contain configurations used for unit testing.
    Configurations needed for unit testing shall be added to the ``test`` directory in the CSC repository and use the override feature in CSCs (see `Salobj documentation <https://ts-salobj.lsst.io>`__).

#.  All configuration files shall have header metadata fields explaining that they are loading basic values from ``_init.yaml``, as shown in the :download:`example configuration file <_static/ATSpectrograph_example_config.yaml>` mentioned above.


Required Unit and Continuous Integration (CI) Testing
-----------------------------------------------------

Due to the dependence of the configuration files on the defined schema, which are located in different repositories, CI tests are required to ensure there is no breakage when making modifications in either repository.
The verification of a configuration requires that the files are syntactically correct and that all fields are populated with correctly formatted values.
This verification is what is performed in the following tests.
The validation of a configuration requires that the input values are indeed the correct values required by the user.
Validation is out of scope for CI tests.

The following CI tests are required on all configuration repos (e.g. ``ts_config_attcs``):

    #. Verify that if site-specific configuration files exist, then they exist for all sites, and the site names are valid
    #. Verify that ``_init.yaml`` + ``_<site>.yaml`` results in a complete configuration.
       This is performed for each site-specific file.
       If there are no site-specific files, then the ``_init.yaml`` file will be verified to be complete.
    #. Verify that ``_init.yaml`` + ``_<site>.yaml`` + ``<override>.yaml`` is valid for all combinations of site and override files.
    #. Verify that new and/or updated configurations have updated metadata
    #. Verify that no filesnames match the list of forbidden names

The following CI tests are required on all configurable CSC repos (e.g. ``ts_ATDome``):

    #. Verify that no defaults are set in the schema.
    #. Verify that all configuration files in the configuration repository (e.g. ``ts_config_attcs``) match the current schema.


Appendix I: Creating Configurations for non-salObj CSCs
=======================================================

This appendix explains how to produce configuration files for CSCs that do not fully follow the standard above.

.. _section-pointing-component:

Pointing Component
------------------

The pointing component has a configuration file that resides with the code base which, in itself, also defines a couple different files (e.g. pointing model).
Nevertheless, the CSC is not developed to be a configurable CSC, meaning it does not accept a ``configurationOverride`` value to switch between different configurations and does not output the required events.

The CSC is being developed by Observatory Sciences using C++.

.. Important::

    PROCEDURE TO BE ADDED/LINKED


.. _section-atmcs-atpneumatics:

ATMCS and ATPneumatics
----------------------

The ATMCS and ATPneumatics are both being developed in LabVIEW under a subcontract with CTIO.
Both CSCs contain a couple of low-level ``.ini`` configuration files that are stored with the main code base, but the CSCs are not classified as configurable CSCs.
Neither CSC accepts a ``configurationOverride`` value to switch between different configurations, nor outputs the configuration specific events.

.. _section-non-configurable-cscs:

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all.
Examples are sparse in our current architecture but, the from Salobj point of view, a CSC can be developed on top of a ``BaseCSC`` which makes it a non-configurable component.
Non-configurable CSCs will have no data in the configuration column of the `Master CSC Table`_.

A non-configurable CSC will ignore the ``configurationOverride`` parameter of the ``start`` command, as it has no meaning.
Likewise these CSCs will not output any of the configuration-related events.

Appendix II: System Requirements
================================

This document is derived from requirements spread accross multiple resources.
Many come from :lse:`209`.
The following are requirements from other system level requirement documents that are also applicable.

.. _section-lse-60:

LSE-60
------

Requirement TLS-REQ-0065, in section 2.8.1.3 from the Telescope & Site Subsystem Requirements :lse:`60` states that:

    The Telescope and Site shall publish telemetry using the Observatory specified protocol (Document-2233) containing time stamped structures of all command-response pairs and all technical data streams including hardware health, and status information.
    The telemetry shall include all required information (metadata) needed for the scientific analysis of the survey data as well as, at a minimum, the following:
    Changes in the internal state of the system, Health and status of operating systems, and Temperature, rate, pressure, loads, status, and conditions at all sensed system components.

This is a broad requirement specifying that components must publish operational status information.

.. _section-lse-62:

LSE-62
------

The LSST Observatory Control System Requirements Document :lse:`62` contains three requirements regarding system configuration:

Requirement OCS-REQ-0045 in section 3.4.4 (Subsystem Latest Configuration) states that:

        Specification: The Configuration Database shall manage the latest configuration for each subsystem, for the different observing modes.

        Discussion: The Configuration Database maintains also the latest configuration utilized during operations that can be utilized for rapid restoration of service in case of failure.

Requirement OCS-REQ-0069 in section 3.4.4.1 (Subsystem Parameters) state that:

    Specification: The Configuration Database shall manage the subsystem parameters for the different observing modes.

Requirement OCS-REQ-0070 in section 3.4.4.2 (Subsystem History) state that:

    Specification: The Configuration Database shall manage subsystem history for the different observing modes.

See furthermore details about the adopted definition of "configuration database" in the context of the control software architecture and more details about the proposed implementation.

.. _section-lse-150:

LSE-150
-------

Section 2.4 of the LSST Control Software Architecture :lse:`150` describes how to perform configuration management.
The document provides two valid alternatives for managing configuration in the LSST system; through a configuration database or version control system.

For a configuration database, any solution is acceptable as long as the technology allows versioning of the database.

For version control systems the adopted solution is `git <https://git-scm.com>`__.
The document also specifies that configurations must be stored in a separate repository from that of the component source code, to allow the configuration to evolve independently of the main code base.
The configuration for different components can be stored individually or in groups of components to facilitate maintainance.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
