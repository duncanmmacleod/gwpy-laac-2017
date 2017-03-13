.. currentmodule:: gwpy.segments
.. sectionauthor:: Duncan Macleod <duncan.macleod@ligo.org>

.. _segments:

#####################
Working with segments
#####################

**Primary documentation:** https://gwpy.github.io/docs/latest/segments/

In order to record when an observatory is taking science-quality data, or is being impacted by a known data-quality issues, time segments are generated in real-time and recorded in :any:`the segment database <segments-io-dqsegdb>`.

Each segment is associated with a :any:`flag <segments-flag>`, a unique identifier for a specific definition.
Important flags include:

===============================  ==============================================
Name                             Description
===============================  ==============================================
``DMT-ANALYSIS_READY:1``         
``DMT-GRD_ISC_LOCK_NOMINAL:1``   
``DMT-ETMY_ESD_DAC_OVERFLOW:1``  
``DMT-OMC_DCPD_ADC_OVERFLOW:1``  
``DMT-PRE_LOCKLOSS_4``           
``ODC-INJECTION_BURST:1``        
``ODC-INJECTION_CBC:1``          
``ODC-INJECTION_DETCHAR:``       
===============================  ==============================================

GWpy provides the `DataQualityFlag` class to represent a flag:

.. plot::
   :include-source:
   :context: reset
   :nofigs:

   >>> from gwpy.segments import DataQualityFlag

=====================
Querying for segments
=====================

The :meth:`DataQualityFlag.query` method can be used to download segments from any segment database:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> observing = DataQualityFlag.query('L1:DMT-ANALYSIS_READY:1', 'Sep 14 2015', 'Sep 15 2015')

The above query will return a `DataQualityFlag` that looks something like::

   >>> print(observing)
   FIXME

**References:**

- :all:`gwpy-segments-dqsegdb`

=================
Plotting segments
=================

The `DataQualityFlag` comes with a :meth:`~DataQualityFlag.plot` method, showing the :attr:`~DataQualityFlag.known` and :attr:`~DataQualityFlag.active` segments for a given flag:

.. plot::
   :include-source:
   :context:

   >>> plot = observing.plot(label='Observing')
   >>> plot.show()

**References:**

- :all:`gwpy-segments-plot`

============================================
Generating segments from data (``guardian``)
============================================

A `DataQualityFlag` can be generated from a dataset by applying a mathematical operation that returns `True` or `False`.
For example, you can use data from the ``guardian`` records in order to generate segments for when the HAM6 seismic isolation system was in the fully isolating mode:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> from gwpy.timeseries import TimeSeries
   >>> data = TimeSeries.get('L1:GRD-SEI_HAM6_STATE_N', 'Jan 4 2017', 'Jan 5 2017')
   >>> isolated = data == 200

The result should be::

   >>> print(isolated)
   FIXME

=====================================
Genearting segments from data (FIXME)
=====================================

Generating a `DataQualityFlag` from variable data is just as easy.
In this example we can generate segments during which the RMS of FIXME was above a certain threshold, which correlates strongly with glitches in *h(t)* from LIGO-Hanford.

FIXME: get info from TJ.
