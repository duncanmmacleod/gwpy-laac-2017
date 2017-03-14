.. currentmodule:: gwpy.table
.. sectionauthor:: Duncan Macleod <duncan.macleod@ligo.org>

.. _tables:

#########################
Working with tabular data
#########################

**Primary documentation:** https://gwpy.github.io/docs/latest/table/

Most analyses of GWO data reduce the time-domain data streams into tables of events, either likely noise artefacts (in the case of detector characterisation or instrument science) or likely GW signals (data analysis).

GWpy provides the `Table` and `EventTable` classes to represent tabular data:

.. plot::
   :include-source:
   :context: reset
   :nofigs:

   >>> from gwpy.table import (Table, EventTable)

.. plot::
   :include-source: False
   :context:
   :nofigs:

   from gwpy.plotter import EventTablePlot

.. note::

   The `Table` object is just an import of :class:`astropy.table.Table`,
   included in `gwpy.table` for convenience, no real changes have been made
   to that object as part of GWpy.

**References:**

- :any:`gwpy-table`

===================
Finding event files
===================

The first problem when working with event tables is just knowing where the tables live.

A subset of event trigger generators (ETGs) are linked to `trigfind`, so you can use :meth:`trigfind.find_trigger_files` to locate files if you know the data channel, ETG name, GPS start, and GPS end of the query:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> from gwpy.time import tconvert
   >>> from trigfind import find_trigger_files
   >>> files = find_trigger_files('L1:GDS-CALIB_STRAIN', 'omicron', tconvert('Jan 4 2017'), tconvert('Jan 5 2017'))

From this you should get a (long) list of files (84).

As of March 2017, the following ETGs are understood by `trigfind`:

===========================================================  ==================================
ETG                                                          Description
===========================================================  ==================================
`DMTOmega <https://wiki.ligo.org/DetChar/OmegaC>`_           C++ implementation (GDS) of Omega pipeline search
`KleineWelle <https://wiki.ligo.org/DetChar/KleineWelle>`_   Wavelet based burst ETG
`Omega Online <https://wiki.ligo.org/DetChar/OmegaOnline>`_  Initial LIGO automated Omega pipeline (Q-transform) search
`Omicron <https://virgo.in2p3.fr/GWOLLUM/v2r2/Friends/>`_    C++ implementation (GWOLLUM) of Omega pipeline search
`PyCBC Live <https://FIXME>`_                                Online CBC analysis
===========================================================  ==================================

.. note::

   If you have an event generator that produces files in a regular manner,
   and would like it included in `trigfind`, please
   `open a ticket <https://github.com/ligovirgo/trigfind/issues/new>`_.

.. warning::

   In order to find event files, you must execute the
   :meth:`find_trigger_files` command from a machine that has access to
   those files.
   For example, the below examples reading Omicron triggers will only work
   on machines that are part of the LDAS cluster at LLO.

**References:**

- https://ldas-grid.ligo.caltech.edu/~duncan.macleod/trigfind/latest/html/

========================
Reading tables of events
========================

If you have one (or more) files, you can use :meth:`EventTable.read` to load them:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> events = EventTable.read(files, format='ligolw.sngl_burst')

The ``format`` keyword argument should be given if possible - it can be guessed from the file extension(s) if omitted - to specify the structure of the data contained in the files.
For all ``LIGO_LW`` tables the format should be given as ``ligolw.<tablename>``, see `gwpy:table-io-ligolw` for full details.

The above :meth:`~EventTable.read` should return a table that looks like::

   >>> print(events)
ifo peak_time  peak_time_ns ... chisq_dof param_one_name param_one_value
--- ---------- ------------ ... --------- -------------- ---------------
 L1 1167532167    500000000 ...       0.0          phase         -1.5602
 L1 1167532173    194274902 ...       0.0          phase         2.74404
 L1 1167532174    176757097 ...       0.0          phase         1.46188
 L1 1167532176    195434093 ...       0.0          phase         0.99987
 L1 1167532176    511229991 ...       0.0          phase         3.11095
 L1 1167532181    120666027 ...       0.0          phase        -0.71677
 L1 1167532182    162108898 ...       0.0          phase        -1.22148
...        ...          ... ...       ...            ...             ...
 L1 1167611684    618164062 ...       0.0          phase         1.01182
 L1 1167611687    205565929 ...       0.0          phase        -2.57444
 L1 1167611689    680664062 ...       0.0          phase         1.37387
 L1 1167611690    294188976 ...       0.0          phase         -2.2331
 L1 1167611693    606933116 ...       0.0          phase        -0.47045
 L1 1167611696    500000000 ...       0.0          phase          -2.001
 L1 1167611700    250304937 ...       0.0          phase         1.39252
Length = 35691 rows

**References:**

- :any:`gwpy-table-io`

================
Searching tables
================

Sorting
-------

A :class:`~astropy.table.Table` can be sorted over a number of columns via the :meth:`~astropy.table.Table.sort` method.
To sort by signal-to-noise ratio (SNR)::

   >>> events.sort('snr')

This sorts the table **in-place** (meaning your `events` objects is changed directly) by SNR in increasing order, so to print the loudest 5 events::

   >>> print(events[-5:])
   ifo peak_time  peak_time_ns ... chisq_dof param_one_name param_one_value
   --- ---------- ------------ ... --------- -------------- ---------------
    L1 1126246455    648437023 ...       0.0          phase         2.32888
    L1 1126239294    367187023 ...       0.0          phase        -2.15407
    L1 1126246921    683288097 ...       0.0          phase        -0.47281
    L1 1126246475    398437023 ...       0.0          phase        -2.00041
    L1 1126243710    883300065 ...       0.0          phase        -1.58589

Filtering
---------

A :class:`~astropy.table.Table` can be filtered by applying mathematical conditions to columns using the standard `numpy` syntax.
To filter out events except in a given frequency band, first create an index array to find which rows to keep:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> in_bucket = events['peak_frequency'] > 50
   >>> in_bucket &=  events['peak_frequency'] < 500

then apply that to ``events`` to create a new table with only those events that meet all conditions:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> bucketevents = events[in_bucket]

===============
Plotting events
===============

Scatter (*Glitchgram*)
----------------------

Similar to the the :class:`~gwpy.timeseries.TimeSeries`, the :class:`EventTable` comes with a :meth:`~EventTable.plot` method, by default making a scatter plot of the given X-, Y-, and color-axis columns::

   >>> plot = events.plot('time', 'peak_frequency', color='snr')
   >>> plot.show()

This will fail, but only because the ``LIGO_LW`` ``sngl_burst`` format doesn't directly store a ``'time'`` column.
However, we can generate one by combining the ``'peak_time'`` and ``'peak_time_ns'``) columns:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> time = events['peak_time'] + events['peak_time_ns'] * 1e-9
   >>> time.name = 'time'
   >>> events.add_column(time)

Now we can generate the figure as above:

.. plot::
   :include-source:
   :context:

   >>> plot = events.plot('time', 'peak_frequency', color='snr')
   >>> plot.show()

We can clean up the figure as we have done when plotting time-domain data:

.. plot::
   :include-source:
   :context: close-figs

   >>> plot = events.plot('time', 'peak_frequency', color='snr')
   >>> ax = plot.gca()
   >>> ax.set_xlim(1167523218, 1167609618)
   >>> ax.set_epoch(1167523218)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(30, 2000)
   >>> ax.set_ylabel('Frequency [Hz]')
   >>> plot.add_colorbar(cmap='YlGnBu', log=True, clim=[3, 100], label='Signal-to-noise ratio (SNR)')
   >>> plot.show()

Tiles
-----

Many of the burst event generators output 2-dimensional time-frequency tiles, these can be displayed by passing 4 column arguments, defining the central time and frequency, and the duration and bandwidth for each tile:

.. plot::
   :include-source:
   :context: close-figs

   >>> plot = events.plot('time', 'central_freq', 'duration', 'bandwidth', color='snr')
   >>> ax = plot.gca()
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(30, 2000)
   >>> ax.set_xlim(1167523218, 1167609618)
   >>> ax.set_epoch(1167523218)
   >>> plot.add_colorbar(cmap='YlGnBu', log=True, clim=[3, 100], label='Signal-to-noise ratio (SNR)')
   >>> plot.show()

This isn't particularly illuminating over such a long time interval, but we can zoom in around a specific time:

.. plot::
   :include-source:
   :context:

   >>> ax.set_xlim(1167559934, 1167559938)
   >>> ax.set_epoch(1167559934)
   >>> plot.refresh()

Unfortunately, the Omicron clustering means that much of the rich structure is clustered away, but we can still see a series of clusters indicating something loud in the data.

We can repeat the procedure above with the un-clustered Omicron ROOT files to see the structure of the signal on the data:

.. plot::
   :include-source:
   :context: close-figs

   >>> files = find_trigger_files('L1:GDS-CALIB_STRAIN', 'omicron', 1167559934, 1167559938, ext='root')
   >>> tiles = EventTable.read(files, treename='triggers', format='root')
   >>> duration = tiles['tend'] - tiles['tstart']
   >>> duration.name = 'duration'
   >>> bandwidth = tiles['fend'] - tiles['fstart']
   >>> bandwidth.name = 'bandwidth'
   >>> tiles.add_columns((duration, bandwidth))
   >>> plot = tiles.plot('time', 'frequency', 'duration', 'bandwidth', color='snr')
   >>> ax = plot.gca()
   >>> ax.set_xlim(1167559936.5, 1167559936.7)
   >>> ax.set_epoch(1167559936.5)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(50, 500)
   >>> ax.set_ylabel('Frequency [Hz]')
   >>> plot.add_colorbar(cmap='YlGnBu', clim=[5, 8], label='Signal-to-noise ratio (SNR)')
   >>> plot.show()

Histogram
---------

We can easily make a histogram plot of any `EventTable` by calling the :meth:`~EventTable.hist` method, passing the column over which to count:

.. plot::
   :include-source:
   :context: close-figs

   >>> hist = events.hist('peak_frequency', figsize=[12, 6], bins=200, logbins=True)
   >>> ax = hist.gca()
   >>> ax.set_xlabel('Frequency [Hz]')
   >>> ax.set_ylim(30, 2000)
   >>> ax.set_ylabel('Number of events')
   >>> hist.show()

Another table type: PyCBC
-------------------------

Other than the column names, none of the above is specific to Omicron events, so we could repeate the same prescription for any other event tables.
For example, using PyCBC Live events:

.. plot::
   :include-source:
   :context: close-figs

   >>> files = find_trigger_files('L1', 'pycbc_live', 1167523218, 1167609618)
   >>> t = EventTable.read(files, format='hdf5.pycbc_live', columns=['end_time', 'template_duration', 'new_snr'], ifo='L1', nproc=8)
   >>> plot = t.plot('end_time', 'template_duration', color='new_snr')
   >>> ax = plot.gca()
   >>> ax.set_xlim(1167523218, 1167609618)
   >>> ax.set_epoch(1167523218)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(.1, 100)
   >>> ax.set_ylabel('Template duration [s]')
   >>> plot.add_colorbar(cmap='YlGnBu', clim=[6, 10], label='$\chi^2$-weighted SNR')
   >>> plot.show()

.. note::

   The ``'new_snr'`` column is not directly stored in the PyCBC Live files,
   but is derived on-the-fly from other columns.
   This can also be done for ``'mchirp'``, see :any:`gwpy-table-io-pycbc_live` for
   more details.

**References:**

- :any:`gwpy-table-plot`

