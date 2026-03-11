.. image:: https://img.shields.io/badge/dmtn--167-lsst.io-brightgreen.svg
   :target: https://dmtn-167.lsst.io/
.. image:: https://github.com/lsst-dm/dmtn-167/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-167/actions/

##############################################################
Policies and Conventions for Organizing Gen3 Data Repositories
##############################################################

DMTN-167
========

While the Gen3 Butler provides some intrinsic structure to its data repositories, considerably more is left to convention (often encoded in higher-level packages, like obs_base).  This document will be - at least at first - a proposal for how to organize data repositories in detail, focusing on collection naming conventions, filesystem locations, and developer workflows.  The immediate focus will be the environment at NCSA, but it is hoped that much of this will hold for the IDF and USDF as well.
After consultation with other stakeholders and ultimately RFC, at least some of the content here should probably be moved to the DM Developer Guide.

**Links:**

- Publication URL: https://dmtn-167.lsst.io/
- Alternative editions: https://dmtn-167.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-167
- Build system: https://github.com/lsst-dm/dmtn-167/actions/

Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.12 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-167
   cd dmtn-167
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-167.lsst.io/ whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-167.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
