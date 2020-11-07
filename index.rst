..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

While the Gen3 Butler provides some intrinsic structure to its data repositories, considerably more is left to convention (often encoded in higher-level packages, like obs_base).  This document will be - at least at first - a proposal from Science Pipelines leadership on how to organize data repositories in detail, focusing on collection naming conventions, filesystem locations, and developer workflows.  The immediate focus will be the environment at NCSA, but it is hoped that much of this will hold for the IDF and USDF as well.

After consultation with other stakeholders and ultimately RFC, at least some of the content here should probably be moved to the DM Developer Guide.


Collections
===========

The Gen3 Butler organizes datasets primarily via *collections*, which are groups of datasets defined strictly in the database (while some collections may relate directly to filesystem or other storage locations, this should usually be considered an implementation detail).

There are several types of collections, which are quite different in how they how they are written, but almost identical to readers.
A full description of the types of collections can be found in the `daf_butler documentation`_, but a brief summary here should be sufficient for the proposals in this document:
- A ``RUN`` collection (or just "a run") is the only type of collection that is intrinsic to a dataset - a dataset is added to ``RUN`` collection when it is first added to a repository, and remains in that ``RUN`` collection until it is deleted entirely.
- A ``CHAINED`` collection is simply an ordered list of other collections to search.  These can be nested arbitrary (but cycles are of course not permitted), and may be redefined after creation.  A single-element ``CHAINED`` collection can thus be used as a sort of symbolic link, providing a stable name for a conceptual group of datasets whose definition may change over time.
- A ``TAGGED`` collection is an explict list of individual datasets.
- A ``CALIBRATION`` collection is an explict list of (dataset, validity range) tuples, where the validity range is a timespan.

A dataset is always in exactly one ``RUN`` collection, but may be in any number of ``TAGGED`` and/or ``CALIBRATION`` collections as well.

It is also important to note that collections are always groups of *datasets*, not data IDs (again, see the `daf_butler documentation`_ for more information
on these concepts).
So, for example, a collection can contain *raws* or *coadds*, but not *visits* or *tracts*.

In addition, collections are constrained to hold only one dataset with a particular combination of dataset type (e.g. "calexp") and data ID, with the exception of ``CALIBRATION`` collections, for which there may only be one dataset with a particular dataset type and data ID at any point in time.

.. _daf_butler documentation: https://pipelines.lsst.io/v/weekly/modules/lsst.daf.butler/organizing.html

.. _collections-table:

.. table:: Overview of collection naming conventions.

   +--------------------------------+-------------+---------------------------------------------------------------------+
   |          Name Pattern          |    Type     |                             Description                             |
   +================================+=============+=====================================================================+
   | <instrument>/defaults          | CHAINED     | Recommended raw, calibration, and auxilliary data for <instrument>. |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | <instrument>/raw/good          | CHAINED     | Recommended raw data for <instrument>.                              |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | <instrument>/raw/good/<ticket> | TAGGED      | Raw data curated to have no problems on <ticket>.                   |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | <instrument>/raw/all           | RUN         | Where all raw data are originally ingested.                         |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | <instrument>/calib             | CHAINED     | Recommended calibrations for <instrument>.                          |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | <instrument>/calib/<ticket>    | CALIBRATION | Calibrations certified on <ticket>.                                 |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | refcats                        | RUN         | All reference catalogs (distinguished by dataset type).             |
   +--------------------------------+-------------+---------------------------------------------------------------------+
   | skymaps                        | RUN         | All skymap definition datasets (distinguished by data ID).          |
   +--------------------------------+-------------+---------------------------------------------------------------------+

Raw, master calibration, and auxilliary data
--------------------------------------------

Raw and calibration data associated with a particular instrument is organized into collections that start with the "short" instrument name, e.g. "HSC" or "LSSTCam-imSim", followed by a slash.
The naming conventions for these collections are codified by the `lsst.obs.base.Instrument`_ class's ``make*Name`` methods.
The highest-level collections are always defined as ``CHAINED`` collection "pointers" to other versioned collections.

In the case of raw data, we propose three levels of collections:

 - ``<instrument>/raw/all``: the ``RUN`` collection into which all raws for that instrument are originally ingest.
 - ``<instrument>/raw/good/<ticket>``: a ``TAGGED`` collection containing a curated subset of all raws that do not contain problems (e.g. tracking issues, airplanes, etc.), named according to the ticket (e.g. ``DM-98765``) on which the curation work was done.
 - ``<instrument>/raw/good``: a ``CHAINED`` collection that points to the current-best ``*/good/<ticket>`` collection for this instrument.

The collections for master calibrations follow a similar pattern, but because master calibration datasets are produced by our own pipelines, not ingested, [#calibs-not-ingested]_ there is no single ``RUN`` collection that holds these all of these datasets directly.
As described further in :ref:`collections-calibration-production`, each processing run generates a new ``RUN`` collection.

Certifying these calibration datasets - marking them as acceptable for use in calibrating observations taken in a certain temporal validity range - involves adding them to a ``CALIBRATION`` collection.
These should have names of the form ``<instrument>/calib/<ticket>``, and we will use single-element ``CHAINED`` collections of the form ``<instrument>/calib`` as pointers to the current best set of calibrations for each instrument.

.. note::

   ``CALIBRATION`` collections that are not candidates for broad use (e.g. because they represent experimental work on a development branch) should instead start with ``u/<user>``, as described in :ref:`collections-developer-processing-outputs`.

We also have other auxilliary datasets that are not instrument-specific, namely reference catalogs and skymap definitions.
These are stored in ``refcats`` and ``skymaps`` ``RUN`` collections, respectively; we do not expect to need to use ``CHAINED`` collections for indirection here, because the dataset types and data IDs of these datasets already uniquely identify them.
In particular:

 - The name of a reference catalog (e.g. ``ps1_pv3_3pi_20170110``) is used directly as its dataset type name (note that this was not the case in Gen2, where the reference catalog name was part of its data ID).

 - All skymaps must have a globally unique name in Gen3, which is used as part of the data ID for any dataset that is defined on tracts.  The skymap definition datasets (e.g. ``deepCoadd_skyMap``) also include this globally unique name in their data IDs.  The existence of different skymap definition datasets for different coadd types (``goodSeeingCoadd_skyMap``, etc.) is something of a relic of Gen2, but one we do not plan to remove until Gen2 is fully removed.  The new globally-unique skymap data ID names are both necessary and sufficient for Gen3, and in the future we may not store skymap definitions in datasets at all, since they must be at least partially defined in the database as well.

Finally, for convenience, we will define per-instrument ``CHAINED`` collections that aggregate the recommended raws (``<instrument>/raw/good``), recommended calibrations (``<instrument>/calib``), and all auxiliary collections (``refcats`` and ``skymaps``), with names of the form ``<instrument>/defaults``.

.. _lsst.obs.base.Instrument: https://pipelines.lsst.io/v/weekly/py-api/lsst.obs.base.Instrument.html#lsst.obs.base.Instrument

.. [#calibs-not-ingested] In Gen2, master calibration datasets *were* ingested, because the data repository in which they were produced was entirely different from the special calibration repository where they were put after certification.  Gen3 data repositories are larger, with Gen3 collections corresponding more closely to Gen2 repositories.  So certifying a master calibration in Gen3 just involves adding it to a new collection, not ingesting it into a new data repository.

HSC-Only Collections
^^^^^^^^^^^^^^^^^^^^^^^

Our HSC processing uses bright object masks produced by external code.
By analogy with raw and calibration data, these will be stored in a ``HSC/masks/S18A`` ``RUN`` collection, with a ``HSC/masks`` single-element ``CHAINED`` collection pointer to the current best version.
``S18A`` refers to the HSC internal release in which these masks were first used.
It is somewhat unlikely we will ever either add older mask versions or new masks in the same form to LSST data repositories (LSST processing is moving to a different approach, and HSC will follow suit), this gives us a clear place to put them without naming conflicts.
The top-level ``HSC/defaults`` collection will include ``HSC/masks`` as well.

Official/common processing outputs
----------------------------------

Science processing
^^^^^^^^^^^^^^^^^^

.. _collections-calibration-production:

Calibration production
^^^^^^^^^^^^^^^^^^^^^^

.. _collections-developer-processing-outputs:

Developer processing outputs
----------------------------


Filesystem locations
====================


Access controls
===============


Best practices
==============


Personal and test-package repositories
======================================


Notable Omissions
=================

Collections that represent fields of particular interest or regularly-reprocessed test datasets are not described here, because those are conceptually more groups of data IDs than groups of raws (e.g. not just raw exposures, but tracts on which to combine them as well).
As in Gen2, we will continue to record the definitions of these groups outside the data repository itself, though we may add support for in-repository storage to Gen3 in the future.
It is also worth noting that exposure or visit metadata can sometimes be used to help select some of these data IDs (e.g. ``visit.target_name='SSP-Wide``), and these selections are automatically combined with the filters of a ``<instrument/raw/good`` input collection.


.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
