
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote describes policy that has been adopted (RFC-741) but not fully implemented.**

While the Gen3 Butler provides some intrinsic structure to its data repositories, considerably more is left to convention (often encoded in higher-level packages, like obs_base).
This document will be - at least at first - a proposal for how to organize data repositories in detail, focusing on collection naming conventions, filesystem locations, and developer workflows.
The immediate focus will be the environment at NCSA, but it is hoped that much of this will hold for the IDF and USDF as well.
A core assumption is that there will be a very small number of large shared data repositories for all instruments at NCSA (and each other major facility), for "friendly" use by the DM team; in particular, we will have one data repository for all real (non-simulated) data.

Shared repositories insulate most users from having to worry about raw ingest, calibrations, references, and (now) even skymap definition.
In Gen3, including data from multiple instruments in the same data repository opens up processing data from those instruments together (albeit with careful control over configuration, as our usual obs-package overrides may not work), and allows them to directly share reference catalogs and skymaps.

Smaller custom data repositories may also exist (especially for CI), but we hope to ensure that nearly all development work can be performed without the need for developers to ever create their own personal repositories.
One or more large shared repositories for science users are also expected to exist, but are explicitly beyond the scope of this proposal.

Simulated data other than DESC DC2 (e.g. the NCSA "test stand") is also explicitly not covered in this proposal; the author is not sufficiently familiar with the scope and structure of such data to make a proposal.
The real-data repository should generally not contain simulations, however; this could easily lead to confusion and conflicts, while seeming to have no benefits at all.

After consultation with other stakeholders and ultimately RFC, at least some of the content here should probably be moved to the DM Developer Guide.


Collections
===========

The Gen3 Butler organizes datasets primarily via *collections*, which are groups of datasets defined strictly in the database (while some collections may relate directly to filesystem or other storage locations, this should usually be considered an implementation detail).

There are several types of collections, which are quite different in how they how they are written, but almost identical to readers.
A full description of the types of collections can be found in the `daf_butler documentation`_, but a brief summary here should be sufficient for the proposals in this document:

- A ``RUN`` collection (or just "a run") is the only type of collection that is intrinsic to a dataset - a dataset is added to ``RUN`` collection when it is first added to a repository, and remains in that ``RUN`` collection until/unless it is deleted entirely.
- A ``CHAINED`` collection is an ordered list of other collections to search.  These can be nested arbitrarily (but cycles are of course not permitted), and may be redefined after creation.  A single-element ``CHAINED`` collection can thus be used as a sort of symbolic link, providing a stable name for a conceptual group of datasets whose definition may change over time.
- A ``TAGGED`` collection is an explict list of individual datasets.
- A ``CALIBRATION`` collection is an explict list of (dataset, validity range) tuples, where the validity range is a timespan.

A dataset is always in exactly one ``RUN`` collection, but may be in any number of ``TAGGED`` and/or ``CALIBRATION`` collections as well.

It is also important to note that collections are always groups of *datasets*, not data IDs (again, see the `daf_butler documentation`_ for more information
on these concepts).
So, for example, a collection can contain *raws* or *coadds*, but not *visits* or *tracts*.

In addition, collections are constrained to hold only one dataset with a particular combination of dataset type (e.g. "calexp") and data ID, with the exception of ``CALIBRATION`` collections, for which there may only be one dataset with a particular dataset type and data ID at any point in time.

.. note::
   Butler nomenclature in both Gen2 and Gen3 uses "dataset" to refer to entities that are of order one file (e.g. "a raw" or "a coadd" is "a dataset").
   Informally, we also frequently refer to large sets of these datasets and/or their data IDs ("HSC PDR1" or "DESC DC2") as datasets, but I will avoid that usage whenever possible here.

The naming patterns for collections proposed here are summarized in :ref:`table-overview-real` and :ref:`table-overview-dc2`, with details and explanations in the following subsections.

.. _table-overview-real:

.. list-table:: Overview of collection naming conventions for real (non-simulated) data.
   :header-rows: 1

   * - Name Pattern
     - Type
     - Description
   * - <instrument>/defaults
     - CHAINED
     - Recommended raw, calibration, and auxiliary data for <instrument>.
   * - <instrument>/raw
     - CHAINED
     - Recommended raw data for <instrument>.
   * - <instrument>/raw/<ticket>
     - TAGGED
     - Raw data curated to have no problems on <ticket>.
   * - <instrument>/raw/all
     - RUN
     - Where all raw data are originally ingested.
   * - <instrument>/calib
     - CHAINED
     - Recommended calibrations for <instrument>.
   * - <instrument>/calib/<ticket>
     - CALIBRATION
     - Calibrations certified on <ticket>.
   * - <instrument>/calib/<ticket>/*
     - unspecified
     - Calibration production runs.
   * - [<instrument>/]runs/<target>/<release>/<ticket>
     - CHAINED
     - Public outputs of processing data <target> with <release> on <ticket>.
   * - [<instrument>/]runs/<target>/<release>/<ticket>/*
     - unspecified
     - Private intermediates of processing data <target> with <release> on <ticket>.
   * - refcats
     - CHAINED
     - All reference catalogs (distinguished by dataset type).
   * - refcats/<ticket>
     - RUN
     - One reference catalog, ingested and sharded on <ticket>.
   * - skymaps
     - RUN
     - All skymap definition datasets (distinguished by data ID).
   * - pretrained_models/<model>
     - RUN
     - One pretrained neural network, with package name <model>.
   * - injection/defaults
     - CHAINED
     - All required input data for synthetic source injection.
   * - injection/catalogs
     - CHAINED
     - All synthetic source injection input catalogs.
   * - injection/catalogs/<ticket>
     - RUN
     - Synthetic source injection input catalog as defined on <ticket>.
   * - u/<user>/*
     - unspecified
     - Experimental/development processing by <user>.

.. _table-overview-dc2:

.. table:: Overview of collection naming conventions for DESC DC2 data.

   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   |                   Name Pattern                    |    Type     |                                  Description                                  |
   +===================================================+=============+===============================================================================+
   | <X>.<Y>[ip]/defaults                              | CHAINED     | Recommended raw, calibration, and auxiliary data for a simulation run.        |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/raw                                   | CHAINED     | Recommended raw data for for a simulation run.                                |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/raw/<ticket>                          | TAGGED      | Raw data curated to have no problems on <ticket>.                             |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/raw/all                               | RUN         | Where all raw data are originally ingested.                                   |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/calib                                 | CHAINED     | Recommended calibrations for for a simulation run.                            |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/calib/<ticket>                        | CALIBRATION | Calibrations certified on <ticket>.                                           |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | <X>.<Y>[ip]/calib/<ticket>/*                      | unspecified | Calibration production runs.                                                  |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | [<X>.<Y>[ip]/]runs/<target>/<release>/<ticket>    | CHAINED     | Public outputs of processing data <target> with <release> on <ticket>.        |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | [<X>.<Y>[ip]/]runs/<target>/<release>/<ticket>/*  | unspecified | Private intermediates of processing data <target> with <release> on <ticket>. |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | refcats                                           | CHAINED     | All reference catalogs (distinguished by dataset type).                       |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | refcats/<ticket>                                  | RUN         | One reference catalog, ingested and sharded on <ticket>.                      |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | skymaps                                           | RUN         | All skymap definition datasets (distinguished by data ID).                    |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+
   | u/<user>/*                                        | unspecified | Experimental/development processing by <user>.                                |
   +---------------------------------------------------+-------------+-------------------------------------------------------------------------------+

.. _daf_butler documentation: https://pipelines.lsst.io/v/weekly/modules/lsst.daf.butler/organizing.html

.. _collections-per-instrument:

Per-instrument collections
--------------------------

Raw and calibration data associated with a particular real-data instrument is organized into collections that start with the "short" instrument name, e.g. "HSC" or "LSSTCam-imSim", followed by a slash.
The naming conventions for these collections are codified by the `lsst.obs.base.Instrument`_ class's ``make*Name`` methods.
The highest-level collections are always defined as ``CHAINED`` collection "pointers" to other versioned collections.

In the case of raw data (including both science observations and raw calibrations), we propose three levels of collections:

- ``<instrument>/raw/all``: the ``RUN`` collection into which all raws for that instrument are originally ingested.
- ``<instrument>/raw/<ticket>``: a ``TAGGED`` collection containing a curated subset of all raws that do not contain problems (e.g. tracking issues, airplanes, etc.), named according to the ticket (e.g. ``DM-98765``) on which the curation work was done.
- ``<instrument>/raw``: a ``CHAINED`` collection that points to the current-best ``*/<ticket>`` collection for this instrument.

The collections for master calibrations follow a similar pattern, but because master calibration datasets are produced by our own pipelines, not ingested, [#calibs-not-ingested]_ there is no single ``RUN`` collection that holds these all of these datasets directly.
As described further in :ref:`collections-calibration-production`, each processing run generates a new ``RUN`` collection.

Certifying these calibration datasets - marking them as acceptable for use in calibrating observations taken in a certain temporal validity range - involves adding them to a ``CALIBRATION`` collection.
These should have names of the form ``<instrument>/calib/<ticket>``, and we will use single-element ``CHAINED`` collections of the form ``<instrument>/calib`` as pointers to the current best set of calibrations for each instrument.

.. note::

   ``CALIBRATION`` collections that are not candidates for broad use (e.g. because they represent experimental work on a development branch) should instead start with ``u/<user>``, as described in :ref:`collections-developer-processing-outputs`.

Finally, for convenience, we will define per-instrument ``CHAINED`` collections with names of the form ``<instrument>/defaults`` that aggregate:

- the recommended raws for that instrument (``<instrument>/raw/good``),
- the recommended calibrations for that instrument (``<instrument>/calib``),
- and cross-instrument auxiliary collections (``refcats`` and ``skymaps``; see :ref:`collections-reference-catalogs` and :ref:`collections-skymap-definitions`, respectively).

.. _lsst.obs.base.Instrument: https://pipelines.lsst.io/v/weekly/py-api/lsst.obs.base.Instrument.html#lsst.obs.base.Instrument

.. [#calibs-not-ingested] In Gen2, master calibration datasets *were* ingested, because the data repository in which they were produced was entirely different from the special calibration repository where they were put after certification.  Gen3 data repositories are larger, with Gen3 collections corresponding more closely to Gen2 repositories.  So certifying a master calibration in Gen3 just involves adding it to a new collection, not ingesting it into a new data repository.

HSC-only auxiliary data
^^^^^^^^^^^^^^^^^^^^^^^

Our HSC processing uses bright object masks produced by external code.
By analogy with raw and calibration data, these will be stored in a ``HSC/masks/S18A`` ``RUN`` collection, with a ``HSC/masks`` single-element ``CHAINED`` collection pointer to the current best version.
``S18A`` refers to the HSC internal release in which these masks were first used.
While it is somewhat unlikely that we will ever add older mask versions or new masks in the same form to LSST data repositories (LSST processing is moving to a different approach to these masks, and HSC will probably follow suit), this gives us a clear place to put them without naming conflicts.
The top-level ``HSC/defaults`` collection will include ``HSC/masks`` as well.

This of course establishes a precedent for other instrument-specific auxiliary data, but we expect this to be sufficiently rare that new cases probably merit their own RFCs.

.. _collections-in-dc2:

Per-instrument collections for DESC DC2
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For DESC DC2 data repositories, a very similar structure is used, but ``<instrument>`` is replaced here by a ``<X>.<Y>[ip]`` simulation number; while DC2 data repositories may in general have multiple instruments (i.e. ImSim and PhoSim), the simulation version number is also necessary to distinguish between different raws and calibs.
It is assumed that all simulation versions utilize the same observational metadata (i.e. ``exposure`` and ``visit`` records), at least within each of ImSim and PhoSim, or that differences are sufficiently small that one simulation version's observations can be used with raws from other simulation versions.
When this is not the case, different data repositories must be used for those incompatible simulation versions.

.. _collections-reference-catalogs:

Reference catalogs
------------------

External reference catalogs reformatted and sharded by DM code are written to ``refcats/<ticket>`` ``RUN`` collections, where ``<ticket>`` is the ticket on which the reformatting and sharding work was performed.
After a reference catalog has been validated, its ``RUN`` is added to the overall ``refcats`` ``CHAINED`` collection.

Different collections for different reference catalogs are not necessary, as the name of a reference catalog (e.g. ``ps1_pv3_3pi_20170110``) is used directly as its dataset type name (note that this was not the case in Gen2, where the reference catalog name was instead part of the data ID).

.. _collections-skymap-definitions:

SkyMap definitions
------------------

All skymaps must have a globally unique name in Gen3, which is used as part of the data ID for any dataset that is defined on tracts.
The skymap definition datasets (i.e. ``lsst.skymap.BaseSkyMap`` subclass instances in Python) also include this globally unique name in their data IDs, and hence can also all go in a single ``skymaps`` collection.
This is a ``RUN`` collection that holds skymap definition datasets directly.

The existence of different skymap definition datasets for different coadd types (``goodSeeingCoadd_skyMap``, etc.) is a relic of Gen2 that will soon be removed entirely from Gen3; all skymap definition datasets will just use the ``skyMap`` dataset type.
The new globally-unique skymap data ID names are both necessary and sufficient for uniqueness in Gen3.

SkyMap registration is something we expect to be rare in Gen3 - *much* more rare than running ``makeSkyMap.py`` was in Gen2 - because we almost always use one of a few standard SkyMaps, and in Gen3 a SkyMap (a combination of a ``lsst.skymap.BaseSkyMap`` class *and* its configuration) can only be registered once.
Discrete SkyMaps, which typically cover only a small part of the sky and are *conceptually* a bit more per-user, may be less rare, but our data model currently does not treat these any differently, and until we can identify the patterns and use cases for creating new SkyMaps (even discrete ones), we propose that any new SkyMap registration in a shared repository be preceded by an RFC.

.. _collections-source-injection:

Source injection datasets
-------------------------

Input catalogs containing synthetic source parameters required for source injection are written to ``injection/catalogs/<ticket>`` ``RUN`` collections, where ``<ticket>`` is the ticket on which the catalog was created.
After a catalog has been ingested, its ``RUN`` is added to the overall ``injection/catalogs`` ``CHAINED`` collection.
All required input data for source injection, including input catalogs, are aggregated in the ``injection/defaults`` ``CHAINED`` collection.

.. _collections-shared-official-processing-outputs:

Shared/official processing outputs
----------------------------------

Processing runs overseen by production operators should produce output collections of the form ``<instrument>/runs/<target>/<release>/<ticket>``, or ``runs/<target>/<release>/<ticket>`` in the (rare) case of processing that includes science data from multiple instruments and none of them can be considered the "primary" instrument.
``<target>`` is a human-meaningful name for the set of data IDs being processed, and ``<release>`` is some kind of DM software release version, so examples of complete processing-output collection names might include ``HSC/runs/RC2/w_2020_50/DM-75643`` or ``DECam/runs/HiTS-2015/d_2021_90/DM-80000``.
These versions are intended to make it easy for users to browse collections and understand what is in them at a glance; formal provenance for software versions actually used in the processing will be automatically stored in the data repository itself.
Of course, the version in the collection name should differ as little as possible from the versions actually used to reduce confusion.

These names should always correspond to a "public" ``CHAINED`` collection that aggregates both all ``RUN`` collections that directly hold outputs and all collections used as inputs.
The organization of those "private" output ``RUN`` collections (if there is more than one) is completely at operator discretion (they may correspond to e.g. different tracts, different stages of processing, different attempts), but these collections should start with the same prefix as the umbrella ``CHAINED`` collection, followed by a slash.

In cases where one or more private ``RUN`` collections contain datasets that should not be considered part of the final public outputs (e.g. because they are superceded by datasets in other private ``RUN`` collections), a ``TAGGED`` collection can be used to screen and aggregate these.
That ``TAGGED`` collection would then be a direct child of the final public ``CHAINED`` collection, instead of any ``RUN`` collections it references.
For example, instead of the following chain involving two processing-output ``RUN`` collections (``first`` and ``second``) as well as the input (``HSC/defaults``):

.. code::

   HSC/runs/w_2021_50/DM-20000              CHAINED
     HSC/runs/w_2021_50/DM-20000/second    RUN
     HSC/runs/w_2021_50/DM-20000/first     RUN
     HSC/defaults                          CHAINED
       (nested input collections)

we would redefine the chain to include a ``TAGGED`` collection (``filtered``)
that references (at the level of individual datasets) the ``first`` and ``second`` runs, but still include the inputs directly:

.. code::

   HSC/runs/w_2021_50/DM-20000              CHAINED
     HSC/runs/w_2021_50/DM-20000/filtered  TAGGED
     HSC/defaults                          CHAINED
       (nested input collections)

.. note::

   It is not generally possible to use a ``TAGGED`` collection as the public output collection for a processing run, because putting master calibrations (which are almost always inputs, even if indirectly) in a ``TAGGED`` collection strips them of their validity ranges and does not allow datasets from different validity ranges to coexist.
   So even if a ``TAGGED`` collection is used, the public ``CHAINED`` collection would contain both that collection and the input ``CALIBRATION`` collection as children.

.. note::

   These public ``CHAINED`` collections essentially mimic Gen2's "parent link" mechanism, which provides at best approximate coarse-grained provenance information about which datasets were used as inputs when producing others.
   The Gen3 repository will eventually be extended to include fine-grained, exact provenance - essentially a serialization of the directed acyclic graph (DAG) that describes the processing.
   Whether queries against that DAG are fast enough to allow this more rigorous provenance information to be used as a type of collection (replacing some usage of ``TAGGED`` and ``CHAINED`` collections) remains to be seen, however.
   It is also worth noting that in general the full DAG does not maintain the usual collection invariant of having only one dataset with a particular dataset type and data ID (e.g. two calexps with the same data ID, from two differently-configured runs, could each contribute to different, non-conflicting coadd patches in downstream runs).

.. _collections-developer-processing-outputs:

Developer processing outputs
----------------------------

Processing initiated by DM developers that are intended primarily for personal or small-group use must start with ``u/<user>`` (e.g. ``u/jbosch``), and are strongly encouraged to start with ``u/<user>/<ticket>`` (e.g. ``u/jbosch/DM-56789``) whenever possible.
Names and structure after this prefix are at user discretion, but we strongly recommend using a combination of ``CHAINED`` collections and ``RUN`` collections to distinguish between "inputs and outputs" collections and "output only" collections, as in :ref:`collections-shared-official-processing-outputs`.
The ``pipetask`` tool will automatically take care of this if the ``--output`` option is used with or instead of the ``--output-run`` option.

.. note::
   **TODO**: It's unclear whether BPS supports this currently, but it should be easy to at least support it under the condition that the ``RUN`` collection be given explicitly as well, instead of generated automatically by appending a timestamp.


.. _collections-calibration-production:

Calibration production
----------------------

Calibration production runs intended for broad use (i.e. outputs will be at least candidates for membership in the recommended calibration collection for this instrument) should output to collections with names that start with ``<instrument>/calib/<ticket>/``.
Those produced for experimental or development purposes should start with ``u/<user>/<ticket>/``.

In either case, the ``RUN`` collections that hold output datasets directly will usually require another disambiguating term, mapping roughly to the expected validity range epoch.
Our current plan is to use our initial estimate of the start date of the validity range; this will rarely change (though if it does, the date in the ``RUN`` collection - which cannot be changed after datasets have been written - will not reflect the actual validity range start date).
We could consider using only dates (possibly with human-incremented integer suffixes, as necessary) for ``RUN`` terms while always using full (e.g. second-precision) date-times for actual validity values in order to reduce both confusion and verbosity.
Actual validity ranges are not assigned until datasets are certified (i.e. added to ``CALIBRATION`` collections), and until then, the usual dataset type + data ID constraint applies (i.e. there can only be one ``bias`` for each detector in a particular ``RUN`` collection).

As noted in :ref:`collections-per-instrument`, certified calibration products intended for broad use should go in ``CALIBRATION`` collections named *just* ``<instrument>/calib/<ticket>``.
``CALIBRATION`` collections can also of course be nested under ``u/<user>/<ticket>``, but may not always be necessary for development work, because a ``RUN`` or ``CHAINED`` collection directly containing e.g. new ``bias`` datasets can also be used as an input to a processing run that generates new ``flat`` datasets (as long as only one calibration epoch is in play).

.. note::

   While the middleware *can* use ``RUN`` collections as inputs to later CPP processing steps, the CPP team may declare in the future that this should not be done as part of official calibration production.

.. note::

   "Curated" calibration datasets that are written from a source-of-truth in an ``obs_*_data`` git repository (rather than generated directly via pipeline processing) are currently written to ``RUN`` collections with names of the form ``<instrument>/calib/curated/<calibDate>``, which are then ingested directly into an ``<instrument>/calib`` ``CALIBRATION`` collection (which clashes with our proposal earlier to make ``<instrument>/calib`` a ``CHAINED`` collection "pointer").

   The full workflow for curated calibrations is sufficiently unclear that it is unlikely that we will get this right in time for the first long-lived Gen3 repository.
   Initially, our proposal is to use ``RUN`` collections of the form ``<instrument>/calib/<ticket>/curated/<calibDate>``, and a ``CALIBRATION`` collection of the form ``<instrument>/calib/<ticket>`` (which would in general hold non-curated calibrations as well).
   This leaves room for multiple curated calibration ingests to coexist, which is necessary because they will improve over time, but we don't want to assume we can remove old ones.
   It does not provide a way to avoid duplication of curated calibration datasets that have not changed.

   Calibration collections created by converting the default Gen2 calibration repo for an instrument will use ``gen2/defaults`` instead of ``<ticket>``, i.e. ``<instrument>/calib/gen2/defaults`` for the ``CALIBRATION`` collection.

.. _collections-pretrained-models:

Pretrained models
-----------------

Pretrained neural networks for ``meas.transiNet.RbTransiNetTask`` (and similar tasks that may be developed in the future) are stored in ``pretrained_models/<model>`` ``RUN`` collections, where ``<model>`` is the model package name.
Models cannot be explicitly selected through task connections, but only implicitly by including a specific run in the execution inputs.
Therefore, there is no overall ``CHAINED`` collection, and the models are not included in ``<instrument>/defaults``.

Collections disambiguated by ticket number are not necessary, as the name of a model (e.g. ``rbResnet50-DC2``) may include any versioning information, just as refcats are labeled by conversion date.


Filesystem locations
====================

The main shared data repository for all real-data instruments at NCSA will have a public repository root of ``/repo/main``, which will be a symlink to a directory of the form ``/repo/main_<YYYYMMDD>``.
These directories will each contain a ``butler.yaml`` file that points to the appropriate database (with a one-to-one correspondance between databases or database schemas and ``main_<YYYYMMDD>`` directories).
The DC2 shared data repository will use an analogous structure with ``/repo/DC2`` and ``/repo/DC2_<YYYYMMDD>`` paths.

The default (POSIX) datastore will write datasets with templates that begin with the ``RUN`` name, resulting in e.g. the datasets of per-instrument ``RUN`` collections landing in ``/repo/main_<YYYYMMDD>/<instrument>/`` and per-user ``RUN`` collections landing in ``/repo/main_<YYYYMMDD>/u/<user>``.
Users are discouraged from inspecting these directories (as this will be at least quite different in the IDF or other future cloud-based datastores), and *strongly* discouraged from modifying them in any way other than via middleware tools.
In many cases, write access will actually be prohibited (see :ref:`access-controls`).

When migrations are necessary due to changes in the repository format (something that is *always* preceded by an RFC with explicit CCB approval), a new ``main_<YYYYMMDD>`` (or ``DC2_<YYYYMMDD>``) directory and database/schema pair will be created, and files will shared via hard links until/unless the old repository is retired.

The existing ``/datasets`` and ``/lsstdata`` paths will remain largely as-is, and may be mounted as fully read-only in any context in which only Gen3-mediated access is needed.
Nested paths within that contain fully-Gen2-managed datasets (such as processing outputs) will be converted to Gen3 via hard-link transfers to the corresponding Gen3 location under ``/repo``.
These Gen2 locations may be removed when the Gen2 middleware is fully retired.
Files under these paths that are typically symlinked into Gen2 data repositories (such as raw data) will be ingested in-place into the Gen3 data repositories; symlinks would also be possible, but are unnecessary given that Gen3 supports in-place ingest.
These paths may renamed for consistency after Gen2 retirement, as long as the Gen3 database entries are updated accordingly.

:ref:`table-existing-paths` describes the migration plan for existing dataset locations at NCSA in detail.
New locations (and access controls) are described in :ref:`table-new-paths`.

.. _table-existing-paths:

Current paths and migrations
----------------------------

.. raw:: html
   :file: _static/current-paths-and-migrations.html

.. _access-controls:

Access Controls
===============

The current Gen3 registry architecture does not allow any fine-grained access control in the repository database; we instead rely on "friendly users" being careful and respectful of this shared space.
The access control rules for most users are extremely simple: do not create, modify, or write datasets to any collection unless it starts with ``u/<user>``.
References to both shared datasets and collection, and other user's datasets and collections (via ``TAGGED`` and ``CHAINED`` collections) are allowed, but shared collections should not reference personal (``u/<user>``) collections

We may be able to add a small number of guards via database access control systems (specifically PostgreSQL's "row-level security") in the future, but we do not ever plan to make these exhaustive (the long-term plans for butler access control involve a different approach; see `DMTN-163`_).
Our focus will be limited to the most important shared collections and those easiest to accidentally modify, and the details of these guards are beyond the scope of this document.

.. warning::

   NEVER use ``psql`` or other direct-SQL clients (e.g. the Python DBAPI or SQLAlchemy) to perform write operations in the repository database.
   These can corrupt the data repository, and we have essentially no way to guard against them.

   It should not be necessary in the long term to ever use direct SQL access even for read access; the SQL schema is *not* considered a public interface - but we recognize that this may be necessary for debugging for a while.
   This can be ensured by running:

   .. code:: sql

      SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY;

   at the start of the session.

   If you have to do this (and not at the prompting of a middleware team member trying to help diagnose a problem), please also create a ticket explaining what you wanted to do that couldn't be done with butler tools, so we can address that feature gap.

We do plan to use filesystem access controls to protect shared and per-user files, and we plan to implement some checks in the Butler client itself to make it at least difficult to *accidentally* cause problems.

This proposal specifies filesystem access controls in terms of small number of groups that mostly grant permission to create subdirectories in files in various paths under ``/repo``.
How to map these to users, groups, and filesystem, directory, or file-level permissions in detail is something I'd prefer to leave to the system administrators.
All directories in ``/repo`` should always be world-readable.

In addition, all directories in ``/datasets`` and ``/lsstdata`` are expected to be read-only (from the perspective of Gen3 data repositories) and world-readable.

.. note::

   The first version of this document proposed much more extensive changes to the access controls in ``/datasets``, to enable easier access (i.e. without admin action) to shared datasets by the expert developers that actually oversee them.
   Those aspects of the proposal have been dropped because they were a distraction from (and a lower priority than) getting a shared, long-term Gen3 data repository up and available.

Access controls for directories under ``/repo`` are detailed in the table below.

.. _table-new-paths:

New paths and access controls
-----------------------------

.. raw:: html
   :file: _static/new-paths-and-access-controls.html


.. _DMTN-163: https://dmtn-163.lsst.io


Personal and test-package repositories
======================================

This proposal is primarily concerned with long-lived, shared data repositories of the sort that will exist not just at NCSA, but at the IDF, SLAC, CCIN2P3, and other major LSST data facilities.

Small repositories (typically backed by SQLite) are also expected to be common, especially for small-scale CI and local development.
These repositories should follow the same naming patterns whenever possible, but will generally not need as many levels of indirection to guard against future changes or collections, and many of the collections defined here as ``CHAINED`` or ``TAGGED`` collections can instead be safely defined directly as ``RUN`` collections instead.


Notable omissions and future work
=================================

"Collections" of data IDs
-------------------------

Collections that represent fields of particular interest or regularly-reprocessed test datasets are not described here, because those are conceptually more groups of data IDs than groups of datasets (e.g. not just raw exposures, but tracts on which to combine them as well).
As in Gen2, we will continue to record the definitions of these groups outside the data repository itself, though we may add support for in-repository storage of data IDs to Gen3 in the future.
It is also worth noting that exposure or visit metadata can sometimes be used to help select some of these data IDs (e.g. ``visit.target_name='SSP-Wide'``), and these selections are automatically combined with the selection of a ``<instrument>/raw/good`` input collection.


Naming conventions for dataset types
------------------------------------

The names for nearly all dataset types in Gen3 have been inherited directly from Gen2, and while these are sorely in need of standardization and cleanup, we have no plans to change to new names until Gen2 has been fully retired.
Naming conventions for new dataset types would be welcome before then, but are still beyond the scope of this proposal.

In the meantime, users should be aware that dataset types are *global* entities with no implicit namespacing, and hence new dataset types should be created with care.
The ``pipetask`` tool's ``--register-dataset-types`` option is a non-default option for exactly this reason: in a long-lived repository, re-executions of the same pipeline will eventually outnumber executions of new pipelines (especially new pipelines with new datasets), and hence ``--register-dataset-types`` should rarely be needed.
Passing it all the time as a matter of habit is an antipattern, because it makes it easy for a typo to result in long-lived, hard-to-clean-up garbage (dataset types can be removed, but only if there are no datasets of that type).


Provenance and Reproducibility
------------------------------

The plan for provenance in the Gen3 butler is centered around storing the directed acyclic graph (DAG) of datasets and processing "quanta" that is used to drive ``PipelineTask`` execution, after updating it with the unique identifiers of the datasets actually produced and annotating it with information about which input datasets are actually used by the (rare) ``PipelineTasks`` that may not use all predicted inputs.
While some provenance information (e.g. software versions and configurations) are currently associated directly with ``RUN`` collections (and this information, at least, may always be), and ``CHAINED`` collections provide some information about what datasets were used as inputs when creating others (see :ref:`collections-shared-official-processing-outputs`), these do not carry sufficiently detailed information about the relationships between datasets to meet our needs.

Using fine-grained provenance information to exactly reprocess a DAG will actually be quite different from starting a new run "from scratch", as it doesn't involve providing collections or data IDs as inputs - the input datasets are already fully resolved, so there is no need to search for them in collections, and the data IDs are intrinsic to those datasets.
We will also need to provide ways to *almost* exactly reprocess a DAG, of course - e.g. replacing the initial resolved datasets with new collection + data ID searches, modifying ``Task`` configuration in a way that does not change the DAG (or changes it only in a limited sense), and probably more.

All of this fine-grained provenance is not yet implemented, however, and at present the only way to guarantee reproducibility is for all input collections to have exactly the same state they had when the original run was performed.
The standard collections defined in this document are poorly suited for this role, however; we consider it more important for these to track the "current best" (or in the case of raws, recent observations) than it is for them remain immutable.
Users should thus be aware that repeated processing runs using the same input collections (and everything else held constant) are *not* intended to always produce the same results (and this is a feature, not a bug).


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
