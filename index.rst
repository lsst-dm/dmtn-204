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

   In this document we explore the options for allowing annotations of datasets and dimension records within butler registry. These may be associating standard flags with a dataset or a textual annotation. This document does not discuss user interfaces to the annotation system.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).


Requirements
============

As data are taken at the telescope and as those data are processed multiple times in many different ways by pipelines, it is inevitable that people and bots will want to annotate the data to comment on the data quality or any artifacts that are seen that need further analysis.

In the Butler multiple datasets can be associated with a single exposure (also known as an "observation") and the definition of an exposure is decoupled from the datasets that are part of that exposure.
It is therefore also useful to be able to comment on the exposure record as a whole (for example the weather could have turned bad and affected all datasets together) independently.

For the summit systems these requirements are discussed in :cite:`LSE-490`, :cite:`DMTN-173` and :cite:`DMTN-185`.

The core requirements for dataset annotations are:

* It shall be possible to attach textual comments (annotations) to any Butler dataset.
* It shall be possible to attach annotations to exposure records.
* It shall be possible to attach a quality flag to an exposure record or dataset to indicate whether the data are thought to be usable.
* When looking at a derived dataset, it shall be possible to examine provenance records to obtain any annotation associated with an input dataset or dimension record.
* Annotations of a collections of records or datasets shall be supported.
* Multiple annotations shall be allowed per dataset or record, and at least a timestamp and user information has to be recorded.

Quality flags do not need to take the place of dynamically calculated metrics (especially since metrics with numeric values are planned to be usable to constrain graph building, :cite:`DMTN-203`).
They can though be used to indicate if the data are so bad as to never be usable and so should never be included in a default collection, or to indicate that some follow up investigation is required.
It could be that the quality flag concept could be replaced by a system allowing arbitrary textual "tags" to be used instead.

The provenance requirement is important in that it might help to identify the problem in a dataset if it is known that an annotation or flag has already appeared on an input.
This might require that actual provenance be tracked rather than quantum graph provenance, since some tasks will filter out bad datasets before using them.

It will be common for multiple datasets or records to be annotated with the same comment or tag/quality flag.
This must be supported although from an implementation perspective, it is not clear whether this is required to be kept as a single annotation or could be represented as multiple entries internally.

Summit vs Data Facility
-----------------------

The summit is a different environment to a data facility and this can lead to some differing motivations in how annotations are handled.

At the summit the observing specialist may wish to make a comment on an observation that is actively in progress -- for example if some event happens during the integration and they want to be able to comment immediately.
The camera will publish the observation ID as soon as the observation starts but any implementation involving a Butler would not be able to work unless an additional system was monitoring the observation ID from the camera and created a stub exposure record that would be fully populated later on.

During observing there can be time gaps between exposures for various reasons.
It could be helpful for those time gaps to be displayed as part of the exposure log user interface.
Rather than storing annotations for time gaps (was it weather? was it a fault and a link to the Jira ticket?) in a completely distinct system, there are advantages to be able to store them along with exposure comments.
Such gaps could coexist with real exposure records as virtual exposures (possible tagged with an ID related to the exposure that closed the gap) but the Butler dimension record system would likely not be amenable to such a concept.

A further complication for the summit is how such annotations and flags can be added by remote observers and whether a view should be available at the USDF with minimal time lag.
It is reasonable for an instrument scientist to want to make comments the morning after observing but where should those comments reside?
If editing is allowed off the mountain should the summit instance receive those comments or should the summit only contain messages made on the summit instance and the source of truth be shifted to the data facility when the night is over?

At a data facility all the output datasets already exist and virtual datasets (as opposed to virtual dimension records) are a concept that is fully supported by the Butler.
The issue mainly becomes one of how to integrate dataset annotations and analysis bots into the campaign management system.
For an integrated butler or butler adjacent implementation, transferring records between registries would be similar to that involved in transferring other records, although the butler adjacent approach would require some thought on how integrated those plugins are with registry.

Open Questions
==============

* Should we allow commenting on any dimension record?
  That would be the more general solution although it is only really helpful for ``visit`` and that would normally inherit the properties of ``exposure`` through provenance.
* Should it be possible to constrain a graph build by either a tag or a quality flag?
* Should dataset provenance affect the graph build if a parent dataset has a particular quality flag set even though the specific input dataset does not?
* Summit users require the ability to tag specific error conditions such as vignetting or bad focus.
  These are annotations against exposure records rather than explicit datasets and is somewhat orthogonal to the limited set of proposed quality flags.
  Should Butler support tags on dimensions?
  Should the set of tags be arbitrary, curated, or restricted per telescope?
* For datasets should quality flagging or applying of tags be handled solely by using TAGGED collections for datasets?
  For dimensions records a TAGGED collection can not be used (we could though automatically move raw datasets into tagged collections as the flag changes) given that there is a need to be able to ask which exposure records have been flagged regardless of where the datasets are stored.
* Should we allow commenting on any dimension record?
  That would be the more general solution although it is only really helpful for ``visit`` and that would normally inherit the properties of ``exposure`` through provenance.
* How is user information verified?
  (this relates to the use of butler client/server :cite:`DMTN-176`)
* Can bots set a quality flag and override a human setting?
* If multiple people (or bots) disagree on a quality flag should the system keep track of all the changes to the quality flag?

Design Options
==============

Each dataset in a Butler is identified by a unique dataset ID that is now a UUID.
For raw data this UUID is predictable and is guaranteed to be the same in all Butler repositories (if ingested into the same run collection).
A dimension record is also uniquely identifiable (within that record's definition) by an integer.

Core Butler
-----------

To integrate dataset annotation directly into the Butler registry would be the most efficient solution in terms of performance and ease of distribution to all butler users.
It would also simplify any requirement to include the quality information in the graph build.

In this scenario the dataset annotations would be stored directly in the Registry database with an associated manager class.
Registry APIs would be added for retrieving and storing annotations and any dataset queries would be able to directly include quality flags in the query.

.. note::

   If a dataset is removed from registry, should the annotation always be removed?

Butler Adjacent
---------------

In this model dataset annotations are not core functionality of middleware but annotations and quality flags are stored in the same database using butler code to create what are known as "opaque" tables.
These tables can be created with foreign key relationships to datasets and also can be configured to have rows automatically removed if a dataset is deleted.
It should be possible for such a table to be included in a data query similar to the method being proposed for handling metric constraints (such as seeing) :cite:`DMTN-203`.

A general downside to this approach is whether a butler supports this optional functionality is not known in advance; queries that work in one location will fail in another because the opaque table might not even exist.
This is always the issue with a pseudo-plugin approach where certain features are present but others are not.
Schema migrations also become harder since it is not known how many of these plugins may exist in user systems.

Distinct Database
-----------------

The final approach is to use a completely distinct database that has no direct linkage into the butler database.
The butler database schema is not treated as part of the public API and direct joins between a separate table and a butler table are strongly discouraged and not guaranteed to always work.
Given this barrier to interoperability it is by definition not possible for anything in the quality flag table to affect the graph building phase (short of doing a pre-query to filter out datasets that should not be included).

Summit Prototype
================

For the current prototype implementation for an exposure log (implemented specifically to annotate and apply quality flags for dimension records), a separate PostgreSQL database stores the annotations and associates them with the exposure ID (aka observation ID).
The quality flags and annotations can be retrieved and edited using a web service with a specialized LOVE interface being designed.
The system is completely isolated from butler and does not have any visibility into the butler database tables.

The contents of this summit system could though be synced to the US Data Facility and integrated into a broader exposure history logging system which could use one of the other design approaches.
It is entirely feasible for the consolidated database to store these records at the data facility but also export them to make them visible to a Butler instance.

The existing standalone system could be modified to also be able to track dataset UUIDs to allow annotations to expand past exposure records.

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa
