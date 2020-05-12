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


.. Important::

    Not all functionality referenced in this document is in place nor has it been agreed upon as the proper implementation.
    For the time being, this document should be considered a proposed implementation.

.. _section-introduction:

Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing multiple components that must operate in consonant.
In order to achieve reliable operation it is fundamental to guarantee that all components are properly configured.
This means observers must be able to, at any point, identify what set of configurations a component is using, what sets are available to be loaded, and how to load the configuration they choose.
During commissioning and engineering runs, users will also be interested in building, validating and verifying new sets of
configurations.

The process of building new sets of configurations will vary considerably for each component.
In some cases, the configuration may be a simple set of host names and ports that the component connects to.
In other cases, the configuration may require the calculation of complex lookup tables (e.g. M1M3 and M2) or model coefficients (e.g. pointing component).
These may require on-sky time for acquiring the data, and complex software to analyse and derive the configuration products.
All of this data needs to be associated with this configuration in a way that is easily accessible and traceable.

In this tech-note we specify how components deal with these different aspects of configuration and the basics procedures required to build and update them. It also contains several examples on how users are to interact with configurations.

CSC Configuration Repositories
==============================

The Rubin Control Software Architecture :cite:`LSE-150` supports two methods for performing configuration management: through a configuration database, or using a version control system.
The only component to use a database is the camera CSCs (LSSTCam, ComCam, ATCamera), all others have adopted `git <https://git-scm.com>`__ to version control and manage configurations.

Configurations must be stored in a separate repository from that of the component source code, to allow the configuration to evolve independently of the main code base.
The location of a CSCs configuration repo is defined in the CSC's documentation, but can also be found in section :ref:`section-appendix-configuration-location`.
The configuration files themselves are often yaml files, but not exclusively.
The contents of the yaml files are defined by the schema which is found in the `schema` directory of each CSC (such as the `ATAOS schema <https://github.com/lsst-ts/ts_ataos/schema>`__).
The schema definition contains default values, however, these are used primarily for testing and development and are *not* a valid configuration.

.. Important::

    CSC configurations are (generally) grouped according to the overall system architecture.
    For instance, the ``ATCS`` control package is used to sequence the motion of multiple components, including the ATDome, ATAOS, ATDomeTrajectory and ATHexapod.

From the top-level configuration folder (e.g. `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__), the configuration package for each component CSC is stored in the configuration repository in a directory with the same name as the CSC. For example, `ATAOS <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS>`__ in `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ stores the configuration files for the `ATAOS <https://github.com/lsst-ts/ts_ataos>`__ CSC.

The first level/subdirectory inside a CSC configuration package will contain folders corresponding to each version of the configuration schema
(e.g. `ATAOS/v1 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v1>`__ and `ATAOS/v2 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v2>`__). The configuration schema only evolves if the fields (not contents) in the configuration file needs to change. The configuration files themselves depend upon the CSCs, but are often yaml files. The CSC defines the schema for its configuration, which lives with the CSC repository (examples to follow).

Finally, each version directory there is a ``_labels`` file, `like this one for the ATAOS <https://github.com/lsst-ts/ts_config_attcs/blob/develop/ATAOS/v2/_labels.yaml>`__.
This file maps a user-friendly label to each configuration file (e.g. default).
This is discussed :ref:`further on <section-configurations-available>`.


.. Verification of a new configuration, in this context, mainly involves the process of guaranteeing that the configuration has the correct schema; the input values have the correct types and respect any specified range.
.. Validating that a configuration is good for operation is a much more involved procedure and may require on-sky time.


.. _section-interface-definition:

Configuration Interactions with CSCs
====================================

For aspects involving CSC configuration, all CSCs interact with the user in the same way.

1. When a CSC transitions to the `Standby` State, the CSC presents the user with the available configurations.

    - These are published as the ``configurationsAvailable`` event.


2. When the user commands the CSC to transition from `Standby` to the `Disabled` state, the user must specify which configuration they wish to load, or the default selection will be selected.

    - This is accomplished using the ``start`` command, using the attribute ``configurationToApply``, as shown in the example in section :ref:`section-configuration-interation_non_default`.


3. Upon transitioning to the `Disabled` State, the CSC presents which configuration was loaded, and all necessary information to find the archived configuration in the future.

    - This is published as the ``configurationApplied`` event

The details of these interactions are discussed in the following sections.


.. _section-configurations-available:

configurationsAvailable
^^^^^^^^^^^^^^^^^^^^^^^

The ``configurationsAvailable`` event is a Generic event that is implemented by every CSC and is published upon entering the ``STANDBY`` state.
It contains four parameters: ``labels``, ``mapping``, ``version`` and ``url``.
The information from these parameters present users with the available set of configurations.

- ``labels`` contains a comma separated list of labels, where each label maps to a configuration file.

    - Labels are meant to simplify the loading of configurations. The name must clearly state the purpose of that configuration (e.g. default, nighttime, daytime) and not contain version numbers or dates.
    - The same label is expected point to different configuration files over time (e.g. `default`), as configurations evolve.
    - **The order of the labels is important**, as the first label in the list will be the one automatically selected by the high-level control system for any configurable CSC.

- ``mapping`` contains a comma separated list of mappings between labels and files in the repo (e.g. "default: ATDome_tuned_20200511.yaml, low-speed: ATDome_tuned_20200124.yaml". The mappings are contained in the ``_label.yaml`` file in the version directory of each configuration repo.

- ``url`` contains a URL indicating how the CSC retrieves its settings.  It will start with "file:" if it is a local clone of a git repo, or the standard URL, if a database.

- ``version`` contains the version information about the local configuration repository or database and is primarily for :ref:`traceability. Details are discussed below <section-configuration-interaction-traceability>`.


The configuration repository or database may contain any number of different configurations with different labels.
However, *all configurations need to be associated with a label to be loaded by the CSC*.

.. _section-configuration-applied:

configurationApplied
^^^^^^^^^^^^^^^^^^^^^

The ``configurationApplied`` event is a Generic event that is published by all CSCs when transitioning to the ``DISABLED`` state.
It currently contains the set of parameters in the ``configurationsAvailable`` event, except each parameter is only a single value, corresponding to what was loaded by the CSC. One additional optional parameter, ``otherInfo``, may be present if required.

The ``otherInfo`` parameter contains a comma-separated list of other specific CSC configuration events.
This may be blank if no other specific CSC events are necessary.
Since it is not possible to provide a generic way for CSCs to output detailed information about the configuration parameters they are loading, it is
recommended to create additional events which are particular to each CSC to carry that information.

Although it is not required, for clarity, we suggest that these events be preceded by `configurationApplied` followed by some description of the content, (e.g., ``configurationAppliedLUT`` or ``configurationAppliedController``).
If ``otherInfo`` is not blank, then those event(s) must be published by the CSC alongside the ``configurationApplied`` event.
The CSC is allowed to publish as many events as necessary to convey the information.

Configuration Examples
^^^^^^^^^^^^^^^^^^^^^^

The most simple (and probably most common) case is for those where the CSC has only a single recommended setting.
Other files or labels may be present, but they are generally unused.
For example, for the ATDome CSC may have:

::

  configurationsAvailable event topic contains:
    - labels: "default, original-install"
    - mapping: "default: 20200511-configuration.yaml, original: 20180317-configuration.yaml"
    - version: "v0.3.0-0-g6fbe3c7"
    - url: "file:///home/saluser/repos/ts_config_attcs/ATDome/v1"

.. Important::

    The above is an example and not the current truth.

If a user (or the atcs) transitions the ATDome from the ``STANDBY`` to ``DISABLED`` state, the `default` configuration will be loaded resulting in the following information being published.

::

  configurationApplied event topic contains:
    - label: "default"
    - mapping: "default: 20200511-configuration.yaml"
    - version: "v0.3.0-0-g6fbe3c7"
    - url: "file:///home/saluser/repos/ts_config_attcs/ATDome/v1"


In the case where CSCs may also have multiple settings that are regularly used, one of them being the preferred or default and another being secondary and so on.
In this case, the purpose of those configurations should be spelled out explicitly.
As an example, the ATAOS has a couple of available options for look-up tables. In this case, we may have something like:

::

  configurationsAvailable event topic contains:
  labels: current,constant_hex,high_degree
  mapping: "current: 20200511-configuration.yaml, constant_hex: 20200511-no-hex-configuration.yaml, high_degree: 20200511-configuration-high-degree fit.yaml"
  version: v0.3.0-0-g6fbe3c7
  url: file:///home/saluser/repos/ts_config_attcs/ATAOS/v2

Note how the ``version`` from both CSCs have the same value, this is because both configurations reside in the same repository: ``ts_config_attcs``, and therefore will have the same commit hash.


For a CSC that uses a configuration database, like the ATCamera, we may have
something like:

::

  configurationApplied event topic contains:
  labels: normal,highgain_fast,lowgain_fast,highgain_slow,lowgain_slow
  version: 1.1,1.2,2.0,3.0
  url:  sqlite:///home/camuser/config/config.db

Another possibility is where the configuration is hosted in a sql database which enables remote connection. Is this case, the url would be different, and maybe contain something like:

::

  url: mysql://10.0.100.104:3306/CONFIG


.. _section-configuration-interation:

Configuration Interaction Use-Cases
===================================

Users will interact with configurations in multiple ways.
In many cases, a user/operator will only need to change the configuration that is currently loaded, and are not concerned with the explicit contents of the configuration itself.
This section illustrates example use-cases for these types of scenarios.

Selecting a default CSC configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In most cases, the control packages contain high-level commands to enable all components under their control, and select the default configuration. An example of this is the ATCS.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    await atcs.prepare_for_onsky()

If working with an individual CSC, which as an operator would be a rare occurrence, default CSC configurations are loaded just be transitioning the CSC via:

.. code-block:: python

    from lsst.ts import salobj

    d = salobj.Domain()
    atdome = salobj.Remote(d, "ATDome", index=1)
    await atdome.start_task()

    await salobj.set_summary_state(atdome, salobj.State.ENABLED)



.. _section-configuration-interation_non_default:

Selecting a non-default CSC configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Selecting non-default configurations via control packages is also possible. A dictionary is used to send the appropriate configuration labels for each component that needs a non-default configuration.

.. code-block:: python

    from lsst.ts.observatory.control import ATCS

    atcs = ATCS()

    await atcs.start_task

    await atcs.prepare_for_onsky(settings={ATAOS: 'constant_hex'})

If working with an individual CSC, which as an operator would be a rare occurrence, default CSC configurations are loaded just be transitioning the CSC via:

.. code-block:: python

    from lsst.ts import salobj

    d = salobj.Domain()
    atdome = salobj.Remote(d, "ATDome", index=1)
    await atdome.start_task()

    await salobj.set_summary_state(atdome, salobj.State.ENABLED, configurationToApply='original-install')


Changing the default configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Changing the default configuration is a more involved endeavour because it entails making a change to the contents of the configuration repository. Because the repo is under version control, the appropriate steps must be taken. For this example let's assume we want to change the default in the ATAOS, which is found in the `ATAOS directory of the ts_config_attcs repo <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS>`__.


1. Create a ticket in JIRA where the title/description note the change being made. Let's assume it creates ticket DM-12345.

2. Clone the repo and checkout a new branch

::

    git clone git@github.com:lsst-ts/ts_config_attcs.git
    git checkout -b tickets/DM-12345

3. Open the most recent schema version (v2) and modify the contents of ``_labels.yaml``. For example, the original version may be:

::

    # Labels for recommended settings; a dict of label: config_file
    default: hex_m1_hex_202003.yaml
    constant_hex: hex_m1_202003_constant_hex.yaml

You wish to add a new configuration label called m1_hex, but then make the `constant_hex` be the default. Therefore, the file would become:

::

    # Labels for recommended settings; a dict of label: config_file
    default: hex_m1_202003_constant_hex.yaml
    hex_m1: hex_m1_hex_202003.yaml

4. Add, Commit and push the changes, with a commit message.

::

    git commit -am "Updated default configuration for ATAOS"
    git push

5. If this is a normal configuration change procedure, then create a pull-request (PR), and have it reviewed, then merged. On-the-fly changes are discouraged but sometimes a reality and are therefore discussed in the section on :ref:`section-configuration-creating-a-new`.

6. Pull (or checkout the branch) with the updated repo from where the CSC

.. _section-configuration-interaction-traceability:

Finding a previously used configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In the future, one may want to verify which configuration was being used for a given observation.
Because we often use generic labels (e.g. `default`), and file contents can change with time, creating a robust version controlled system must go beyond simply changing filenames.
For this reason, additional metadata is associated with each configuration, notably the ``url`` and ``version`` parameters in both the ``configurationsAvailable`` and ``configurationApplied`` events. These parameters are key to ensuring that each configuration is unique, and is traceable to their filename and contents.

The ``url`` parameter simply contains a URL indicating how the CSC connects to its settings (meaning a link to the repo) The ``version`` parameter is more complicated. For all CSCs (except the camera?), the ``version`` parameter is a *branch description*\ [#git_version]_ is automatically generated and populated by the CSCs. This is what is output by running the following command in a configuration repo (e.g. ``ts_config_latiss``):

.. prompt:: bash

    git describe --all --long --always --dirty --broken

.. [#git_version] The option ``--broken`` was introduced in git 2.13.7 and may be removed if required

An example output is, ``heads/develop-0-gc89ef1a``. The repository branch (or tag) name forms the first part of the branch description. It may take any form necessary to convey the appropriate information. They are individual identifiers and can change rapidly. The last 7 characters (``c89ef1a``) is the hash of the commit of the loaded configuration file. Users can find this commit by navigating to the repository on github, searching for the commit hash, then clicking on the "commits" section of the search results, as shown in :ref: `the screenshot below <fig-commit-tracing>`.

.. figure:: /_static/tracing_a_commit_on_github.jpg
    :name: fig-commit-tracing

    Using the ``version`` output in the ``configurationApplied`` event, it is possible to traceback the repo to the configuration that was loaded.


Exceptions
----------
Exception to the above go here.


.. _section-configuration-creating-a-new:

Creating a new configuration
============================

The process to derive new configuration parameters will vary considerably from component to component.
In some cases, the configuration is simple enough that a change may involve simply replacing an IP or hostname value, a routine filter swap on an instrument or updating the limits to an axis range due to some evolving condition.
On the other hand, deriving new parameters may involve generating complex LUTs that may require on sky observations and detailed data analysis.

Following is a detail of each step of the process to update the CSC configuration for CSCs written in salobj.
For other components, see the exception section below.


    #.  Create a Jira ticket to track the work being done. If details or
        discussions are needed they can done using the Jira tickets itself. It
        is also possible to link tickets which allow us to capture situations
        where the configuration change was triggered by some issue reported in
        a different ticket, for instance.
    #.  For CSCs using a git repository to store configurations, e.g. all
        Salobj CSCs, clone the configuration repository locally and create a
        "ticket branch" to work on.

        For legacy CSCs where the configuration is hosted with the main code
        base, the process is very similar. The difference is that the
        repository being cloned will be the main code base instead.

        If the CSC uses a configuration database, make sure a snapshot of the
        database is created and backed up before editing it.

    #.  Execute the work needed to derive the new configuration parameter.

        As mentioned above, in some cases, the process may be straightforward,
        consisting simply of replacing the values of a set of parameters with
        given values (e.g., swapping filters). In these cases, this step will
        be simply verifying any required work was performed and continuing to
        the next step. Jira can/should be used to track those activities.

        The Jira ticket should also be used to track the work done on those
        cases where a more involved analysis is required, e.g., in
        dome and/or on sky data acquisition, EFD queries, data processing etc.
        Any ancillary software or data product required during this process
        should be properly managed using git. When working with Telescope and
        Site components, any software required during this process should be
        stored in a git repository
        in `T&S github organization <https://github.com/orgs/lsst-ts>`__, and
        should follow the standard `T&S development workflow
        guidelines <https://tssw-developer.lsst.io>`__. This includes, but is
        not limited to, EFD queries, Jupyter notebooks, other data analysis
        routines (regardless of the programming language) and so on.
        The preferred location for storing Jupyter notebooks is the
        `ts_notebooks <https://github.com/lsst-ts/ts_notebooks>`__ repository.

        Details on how to deals with Camera and DM components will be given
        in the future.

        Any intermediate data product generated in the process should also be
        stored in the
        `git Large File Storage <https://developer.lsst.io/git/git-lfs.html>`__
        or, if size permits, with the software repository itself.

    #.  Edit/replace the configuration file(s) or add a new file(s) to host the
        new configuration in the CSC configuration directory. Ideally the name
        of the file should reflect the purpose of change, dates can also be
        used as well. Old configuration files can be kept in the repo if they
        still represent valid configurations otherwise, they should be removed.
        Note, though, that they will still remain available on previous
        versions in the git repo, enabling historical comparison.

        Alternatively, load the new configuration into the configuration
        database, if that is the case for the CSC in question.

    #.  Modify the configuration labels so that it maps to the new
        configuration (preferred) or create a new label for the new
        configuration. For Salobj CSC, this is done by editing the
        ``_labels.yaml`` file.

    #.  Commit the changes made to the configuration and push it to the
        online repository.

    #.  Testing and verification of the new configuration.

        The complexity of this step will also vary considerably from component
        to component. In some cases, it might be feasible to test the
        configuration in the actual running CSC. For instance, in the filter
        swap use-cases, one can simply pull the ticket branch in the CSC host
        for testing, and exercise the CSC to verify it is working.

        In other situations the new configuration can be tested and verified
        using simulators.

        Nevertheless, in some other more critical cases tests and verification
        are only possible/feasible with on-sky operations.

        Regardless of which case the CSC falls into, the approach should be to
        run the test and verification with the repository ticket branch. If any
        issues arises during this test the process can either continue to the
        next phase or start over. The Jira ticket created to track the work
        should reflect the results of any tests.

    #.  Create/update related documentation.


    #.  Create pull requests (PRs). Once the configuration is tested, verified
        and documented, it is ready to be officially accepted.

        PRs must be created for all repositories that where modified during
        the process, including, but not limited to, the configuration
        repository, ancillary software and documentation. The PRs will follow
        the standard review procedure. Once the they are approved, merged and
        released the new configuration becomes official and can be deployed.


During commissioning we anticipate that there may be situations where quick
configuration changes need to be implemented. In these cases, the user
should also create a Jira ticket (or work out of an existing ticket) to
document the occurrence. Then, instead of checking out the repository
locally, the user can work out of the deployed CSC configuration directly
in the host. It is important to create a branch in place to work on and,
later, commit-push to the repository and continue with the process
afterwards.

.. warning::

    Users must be aware that failing to commit-push changes done in line may
    result in loss of information. Therefore, this procedure should be reserved
    only for critical situations.



Transient labels with Jira ticket numbers may be used for developing new configurations.
They should be moved to standard type labels at the earliest opportunity.


Imagine now that during a test run, someone connects to the computer running
the ATAOS CSC and edits the configuration. The ``recommendedSettingsVersion``
would reflect that change with something like:

::

  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7-dirty

Even though it may be useful to edit configurations on the fly for testing,
the process should be avoided as much as possible. When this happen, it
prevents us from precisely identifying what configuration was used.
Alternatively, the user could create a branch on their work machine,
make the required changes, commit, push it to github and pull/check out the new
configuration in the CSC machine.


Exceptions
----------
The following require different procedures to create/modify a configuration

- :ref:`Main and Auxiliary Telescope Pointing Components <section-pointing-component>`



.. _section-appendix-configuration-location:

Appendix I: Configuration location for CSCs
===========================================

.. note:: This appendix will contain a table relating the CSC to the configuration location



.. _section-appendix-configuration-non-salObj:

Appendix II: Creating Configurations for non-salObj CSCs
========================================================

This appendix details the require procedures to produce configuration files for specific CSCs.

.. _section-pointing-component:

Pointing Component
^^^^^^^^^^^^^^^^^^

The pointing component has a configuration file that resides with the code
base which, in itself, also defines a couple different files (e.g. pointing
model). Nevertheless, the CSC is not developed to be a configurable CSC,
meaning it does not accept a ``settingsToApply`` value to switch between
different configurations and does not output the required events.

The CSC is being developed by Observatory Sciences using C++.

.. Important::

    PROCEDURE TO BE ADDED

.. _section-m2:

M2
^^

M2 cell system will read “some” configuration files (csv files basically) from
disk, get the LUT values from M2 control system by TCP/IP, and hard-code many
configuration data in code.

M2 control system (e.g. CSC) will read “some” configuration files (csv, tsv,
txt) from disk and has several of hard-coded internal configuration. There is
no document to say where are the hard-coded data and what are they.

All configurations reside with the main code base. The CSC does not send any of
the events required to tie in the configuration version and does not accept a
``settingsToApply`` value to switch between different configurations.

Telescope and Site developers are working to update the M2 controller to fix
the different issues with how it handles configuration, e.g. removing the
hard-coded values, and to make sure it follows the appropriate guidelines.

.. Important::

    PROCEDURE TO BE ADDED

.. _section-atmcs-atpneumatics:

ATMCS and ATPneumatics
^^^^^^^^^^^^^^^^^^^^^^

The ATMCS and ATPneumatics are both being developed in LabView by a
subcontract with CTIO. Both CSCs contain a couple of ``.ini`` configuration
files that are stored with the main code base. Neither CSC accepts a
``settingsToApply`` value to switch between different configurations nor
outputs the required events.

.. Important::

    PROCEDURE TO BE ADDED

.. _section-non-configurable-cscs:

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all. Examples are sparse in our current
architecture but, the from Salobj point of view, a CSC can be developed on top
of a ``BaseCSC`` which makes it a non-configurable component.

A non-configurable CSC will ignore the ``settingsToApply`` attribute of the
``start`` command, as it does not contain any true meaning to it. Likewise
these CSCs will not output any of the configuration-related events.

.. Important::

    PROCEDURE TO BE ADDED


.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
