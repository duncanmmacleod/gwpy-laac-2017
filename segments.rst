.. currentmodule:: gwpy.segments

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
``DMT-ANALYSIS_READY:1``         <IFO> Ready for Analysis (Science &
                                 Calibrated) from h(t) DQ flags
``DMT-GRD_ISC_LOCK_NOMINAL:1``   Guardian indicates IFO is in nominal lock
                                 state
``DMT-ETMY_ESD_DAC_OVERFLOW:1``  ETMY ESD DAC overflow
``DMT-OMC_DCPD_ADC_OVERFLOW:1``  DCPD ADC overflow
``DMT-PRE_LOCKLOSS_4:1``
``ODC-INJECTION_BURST:1``        BURST Injection according to CAL ODC monitor.
``ODC-INJECTION_CBC:1``          CBC Injection according to CAL ODC monitor.
``ODC-INJECTION_DETCHAR:1``      DETCHAR Injection according to CAL ODC
                                 monitor.
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
   <DataQualityFlag('L1:DMT-ANALYSIS_READY:1',
                    known=[[1126224017 ... 1126310417)],
                    active=[[1126233814 ... 1126238906)
                            [1126245453 ... 1126245963)
                            [1126256591 ... 1126264691)
                            [1126301023 ... 1126301382)
                            [1126301535 ... 1126302050)
                            [1126302126 ... 1126304365)
                            [1126304448 ... 1126310417)],
                    description=u'L1 Ready for Analysis (Science & Calibrated) from h(t) DQ flags')>

**References:**

- :any:`gwpy-segments-dqsegdb`

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

- :any:`gwpy-segments-plot`

============================================
Generating segments from data (``guardian``)
============================================

A `DataQualityFlag` can be generated from a dataset by applying a mathematical operation that returns `True` or `False`.
For example, you can use data from the ``guardian`` records in order to generate segments for when the HAM6 seismic isolation system was in the fully isolating mode::

   >>> from gwpy.timeseries import TimeSeries
   >>> data = TimeSeries.get('L1:GRD-SEI_HAM6_STATE_N', 'Jan 4 2017', 'Jan 5 2017')
   >>> isolated = data == 200 * data.unit

.. plot::
   :include-source: False
   :context:
   :nofigs:

   >>> from gwpy.timeseries import TimeSeries
   >>> data = TimeSeries.get('L1:GRD-SEI_HAM6_STATE_N', 'Jan 4 2017', 'Jan 5 2017', frametype='L1_R', nproc=8)
   >>> isolated = data == 200 * data.unit
   >>> flag = isolated.to_dqflag()

The result should be::

   >>> print(isolated)
   FIXME

Which can then be converted into a `DataQualityFlag` via the `~gwpy.timeseries.StateTimeSeries.to_dqflag` method of the resulting `~gwpy.timeseries.StateTimeSeries`:

   >>> flag = isolated.to_dqflag(name='L1:DCH-SEI_HAM6_ISOLATED:1')
   >>> print(flag)
   <DataQualityFlag('L1:DCH-SEI_HAM6_ISOLATED:1',
                    known=[[1167523218.0 ... 1167609618.0)],
                    active=[[1167523218.0 ... 1167609618.0)],
                    description=None)>

###################################
Putting it all together for DetChar
###################################

As an example of putting everything together, we can reproduce a study correlating transient noise in an optical lever to glitches in *h(t)* at LIGO-Hanford.

First when looking at `the summary pages <https://ldas-jobs.ligo-wa.caltech.edu/~detchar/summary/day/20170225/lock/glitches/>`_ we notice a series of loud(ish) glitch events at low frequency:

.. plot::
   :include-source:
   :context: close-figs

   >>> from trigfind import find_trigger_files
   >>> from gwpy.table import EventTable
   >>> files = find_trigger_files('H1:GDS-CALIB_STRAIN', 'Omicron', 1172034018, 1172037618)
   >>> events = EventTable.read(files, format='ligolw.sngl_burst', columns=['peak_time', 'peak_time_ns', 'peak_frequency', 'snr'])
   >>> time = events['peak_time'] + events['peak_time_ns'] * 1e-9
   >>> time.name = 'time'
   >>> events.add_column(time)
   >>> plot = events.plot('time', 'peak_frequency', color='snr')
   >>> ax = plot.gca()
   >>> ax.set_xlim(1172034018, 1172037618)
   >>> ax.set_epoch(1172034018)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(4, 100)
   >>> ax.set_ylabel('Frequency [Hz]')
   >>> plot.add_colorbar(clim=[3, 100], label='Signal-to-noise ratio (SNR)', log=True, cmap='YlGnBu')
   >>> plot.show()

Here we see a series of glitches ar around 10 Hz.
DetChar found correlation with glitches in the optical lever readout for the ETMY optic, so we load those data to see what's up:

.. plot::
   :include-source:
   :context: close-figs

   >>> data = TimeSeries.get('H1:SUS-ETMY_L3_OPLEV_SUM_OUT_DQ', 1172034018, 1172037618)
   >>> plot = data.plot(figsize=[12, 4])
   >>> plot.show()

We can calculate a spectrogram to see the frequency content of the noise:

.. plot::
   :include-source:
   :context: close-figs

   >>> specgram = data.spectrogram(10, 4, 2)
   >>> plot = specgram.plot()
   >>> ax = plot.gca()
   >>> ax.set_xlim(1172034018, 1172037618)
   >>> ax.set_epoch(1172034018)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(.25, 128)
   >>> ax.set_ylabel('Frequency [Hz]')
   >>> plot.add_colorbar(log=True, label=r'ASD [counts/\rtHz]')
   >>> plot.show()

In order to isolate the noise in the time-domain, we can calculate a 1-Hz RMS in the 10-50 Hz band:

.. plot::
   :include-source:
   :context: close-figs

   >>> blrms = data.bandpass(10, 50).rms(1.)
   >>> plot = blrms.plot(figsize=[12, 4])
   >>> plot.show()

It looks like a 30-count threshold would allow us to generate veto segments for this noise, so we apply that and update our plot:

.. plot::
   :include-source:
   :context:

   >>> plot.gca().axhline(30, color='orange', linestyle='--')
   >>> plot.refresh()

.. plot::
   :include-source:
   :context:

   >>> veto = (blrms >= 30 * blrms.unit).to_dqflag(name='Veto')
   >>> plot.add_state_segments(veto)
   >>> plot.refresh()

Finally, we can apply these new veto segments to our original ``events`` table to see the impact:

.. plot::
   :include-source:
   :context: close-figs
   :nofigs:

   >>> vetoed = events['time'].in_segmentlist(veto.active)
   >>> clean = events[~vetoed]

.. plot::
   :include-source:
   :context:

   >>> plot = clean.plot('time', 'peak_frequency', color='snr')
   >>> ax = plot.gca()
   >>> ax.set_xlim(1172034018, 1172037618)
   >>> ax.set_epoch(1172034018)
   >>> ax.set_yscale('log')
   >>> ax.set_ylim(4, 100)
   >>> ax.set_ylabel('Frequency [Hz]')
   >>> plot.add_colorbar(clim=[3, 100], label='Signal-to-noise ratio (SNR)', log=True, cmap='YlGnBu')
   >>> plot.show()

