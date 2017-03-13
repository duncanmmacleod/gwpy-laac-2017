.. currentmodule:: gwpy.timeseries
.. sectionauthor:: Duncan Macleod <duncan.macleod@ligo.org>

.. _data:

################################
Working with interferometer data
################################

**Primary documentation:** https://gwpy.github.io/docs/latest/timeseries/

The primary record of gravitational-wave observatory (GWO) experiments is a time stream of strain amplitude in which we hope to find gravitational wave signals.

GWpy provides the :class:`~gwpy.timeseries.TimeSeries` class to represent some time-domain data:

.. plot::
   :include-source:
   :context: reset
   :nofigs:

   >>> from gwpy.timeseries import TimeSeries

**References:**

- :all:`gwpy-timeseries`

==============
Accessing data
==============

GWpy provides methods to access real interferometer data from a number of sources:

.. autosummary::
   :nosignatures:

   TimeSeries.get
   TimeSeries.fetch
   TimeSeries.find
   TimeSeries.read
   TimeSeries.fetch_open_data

In particular :meth:`TimeSeries.get` takes in a data channel name, and query start time, and a query end time, and will try to get the data for you, either from local frames (if available) (calling to :meth:`~TimeSeries.find`) or via NDS2 (:meth:`~TimeSeries.fetch`)::

   >>> h1data = TimeSeries.get('H1:GDS-CALIB_STRAIN', 1126259446, 1126259478)

.. plot::
   :context:
   :include-source: False
   :nofigs:

   >>> h1data = TimeSeries.fetch_open_data('H1', 1126259446, 1126259478, sample_rate=16384)

.. note::

   Members of the public can use `LOSC <https://losc.ligo.org/>`_ to access released data::

   >>> h1data = TimeSeries.fetch_open_data('H1:GDS-CALIB_STRAIN', 1126259446, 1126259478)

   :meth:`TimeSeries.fetch_open_data` accesses LOSC's 4096 Hz downsampled *h(t)* by default, but you can ask for the full rate by specifying ``sample_rate=16384``

The above :meth:`~TimeSeries.get` should return a TimeSeries that looks like::

   >>> print(h1data)
   TimeSeries([  3.02234488e-19,  3.17519543e-19,  3.01329537e-19,
               ...,  -1.33666536e-19, -1.38777172e-19,
                -1.31856806e-19]
              unit: Unit(dimensionless),
              t0: 1126259446.0 s,
              dt: 6.103515625e-05 s,
              name: H1:GDS-CALIB_STRAIN,
              channel: H1:GDS-CALIB_STRAIN)

**References:**

- :all:`gwpy-timeseries-remote`
- :all:`gwpy-timeseries-ldg`
- :all:`gwpy-timeseries-io`

=============
Plotting data
=============

We can make a simple plot of these data as follows:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> plot = h1data.plot(figsize=[12, 4], color='#ee0000')

To embellish the figure with labels and titles, etc, we can just access the :class:`~matplotlib.axes.Axes` and add elements:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> ax = plot.gca()
   >>> ax.set_ylabel('Amplitude [strain]')
   >>> ax.set_title('LIGO-Hanford strain data')

and then we call :meth:`~gwpy.plotter.Plot.show` to display the figure (interactively)

.. plot::
   :include-source:
   :context:

   >>> plot.show()

Alternatively, we could call :meth:`~gwpy.plotter.Plot.save` to save the figure to a file::

   >>> plot.save('h1-strain.png')

**References:**

- :all:`gwpy-timeseries-plot`

==============================================
Generating an ASD (amplitude spectral density)
==============================================

The standard view of interferometer sensitivity is the Amplitude (or Power) Spectral Density (ASD/PSD).

To calculate the ASD of a `TimeSeries`, call the :meth:`~TimeSeries.asd` method, passing the desired length (in seconds) of the FFT, and the overlap between FFTs:

.. plot::
   :include-source:
   :context: close-figs
   :nofigs:

   >>> h1asd = h1data.asd(4, 2)

By default this calculates an ASD by averaging FFTs using `Welch's method <https://en.wikipedia.org/wiki/Welch%27s_method>`_.
The result should be something like::

   >>> print(asd)
   FrequencySeries([  2.04925641e-20,  4.36704402e-20,
                      1.84908678e-20,...,   4.07249558e-29,
                      9.91489037e-29,  6.78360457e-29]
                   unit: Unit("1 / Hz(1/2)"),
                   f0: 0.0 Hz,
                   df: 0.25 Hz,
                   epoch: 1126259446.0,
                   name: H1:GDS-CALIB_STRAIN,
                   channel: H1:GDS-CALIB_STRAIN)

Now we can :meth:`~gwpy.frequencyseries.FrequencySeries.plot` the ASD:

.. plot::
   :include-source:
   :context:

   >>> plot = h1asd.plot(color='#ee0000', label='LIGO-Hanford')
   >>> ax = plot.gca()
   >>> ax.set_xlim(8, 2048)
   >>> ax.set_ylim(5e-24, 1e-20)
   >>> ax.set_ylabel(r'ASD [strain/\rtHz]')
   >>> plot.show()

We can easily download, transform, and add the LIGO-Livingston data to this figure as well::

   >>> l1data = TimeSeries.get('L1:LDAS-STRAIN', 1126259446, 1126259478)
   >>> l1asd = l1data.asd(4, 2)
   >>> ax.plot(l1asd, color='#4ba6ff', label='LIGO-Livingston')
   >>> ax.legend()

.. plot::
   :include-source: False
   :context:

   >>> l1data = TimeSeries.fetch_open_data('L1', 1126259446, 1126259478, sample_rate=16384)
   >>> l1asd = l1data.asd(4, 2)
   >>> ax.plot(l1asd, color='#4ba6ff', label='LIGO-Livingston')
   >>> ax.legend()
   >>> plot.refresh()

==============
Filtering data
==============

As we can see from the ASDs, the time-series *h(t)* data are dominated by low-frequency noise, and spectral lines (power harmonics, and violin modes, mainly).
We can filter out the known features to try and see anything more interesting elase where.

We start by band-passing the data to remove the lowest and highest frequencies, about which we don't really care:

.. plot::
   :include-source:
   :context: close-figs
   :nofigs:

   >>> h1bp = h1data.bandpass(50, 250, filtfilt=True)
   >>> l1bp = l1data.bandpass(50, 250, filtfilt=True)

Here ``filtfilt=True`` is given to filter forward and backward, preserving the phase of each signal.
By default :meth:`~TimeSeries.bandpass` uses `IIR filtering <https://en.wikipedia.org/wiki/Infinite_impulse_response>`_, which takes a small time to settle, so we crop out the first second of the resulting data:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> h1bp = h1bp.crop(start=1126259447, end=1126259477)
   >>> l1bp = l1bp.crop(start=1126259447, end=1126259477)

We can plot these two new `TimeSeries` separately to see what we have done:

.. plot::
   :include-source:
   :context:

   >>> from gwpy.plotter import TimeSeriesPlot
   >>> plot = h1bp.plot(color='#ee0000')
   >>> plot.add_timeseries(l1bp, color='#4ba6ff', newax=True)
   >>> plot.show()

Here we can now see a feature around 15.5-seconds into the data for each interferometer.

We can further clean our data by notching out the 60 Hz power-line harmonics:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> h1clean = h1bp.notch(60).notch(120).notch(180)
   >>> l1clean = l1bp.notch(60).notch(120).notch(180)

And again make a plot to see what we have done:

.. plot::
   :include-source:
   :context:

   >>> from gwpy.plotter import TimeSeriesPlot
   >>> plot = h1clean.plot(color='#ee0000')
   >>> plot.add_timeseries(l1clean, color='#4ba6ff', newax=True)
   >>> plot.show()

We see that the feature we saw previously is now more prominent.
We can now easily zoom into see what we have found:

.. plot::
   :include-source:
   :context:

   >>> for ax in plot.axes:
   >>>     ax.set_xlim(1126259462.2, 1126259462.6)
   >>>     ax.set_epoch(1126259462.427)
   >>> plot.refresh()

We can try and overlay our two cleaned `TimeSeries` to see if the signals match between interferometers:

.. plot::
   :include-source:
   :context: close-figs

   >>> h1clean.x0 = h1clean.x0 - 0.0078 * h1clean.x0.unit
   >>> l1clean *= -1 * l1clean.unit
   >>> plot = h1clean.plot(label='LIGO-Hanford', color='#ee0000')
   >>> ax = plot.gca()
   >>> ax.plot(l1clean, label='LIGO-Livingston', color='#4ba6ff')
   >>> ax.set_xlim(1126259462.2, 1126259462.6)
   >>> ax.set_epoch(1126259462.427)
   >>> ax.set_ylabel('Amplitude [strain]')
   >>> ax.legend()
   >>> plot.show()

**References:**

- :all:`gwpy-signal-time-domain-filter`

=======================
Generating Q-transforms
=======================

One of the most popular forms of visualising a signal in time and frequency is the `Q-transform <https://en.wikipedia.org/wiki/Constant-Q_transform>`_.
We can apply this transform to each of our original `TimeSeries` to generate Q-tile spectrograms showing the signal:

.. plot::
   :include-source:
   :context: close-figs

   >>> h1q = h1data.q_transform().crop(1126259462.31, 1126259462.45)
   >>> l1q = l1data.q_transform().crop(1126259462.31, 1126259462.45)
   >>> plot = h1q.plot()
   >>> plot.add_spectrogram(l1q, newax=True)
   >>> for ax in plot.axes:
   >>>     ax.set_yscale('log')
   >>>     ax.set_ylim(20, 500)
   >>>     plot.add_colorbar(ax=ax, cmap='viridis', clim=[0, 80], label='Normalized energy')
   >>> plot.show()

For a presentation, or publication, a nice view is to show the (cleaned) data and the Q-transform spectrogram in a single figure, as follows.

First, we create a :class:`~matplotlib.figure.Figure` and :class:`~matplotlib.gridspec.GridSpec` in which to define the new axes:

.. plot::
   :include-source:
   :context: close-figs
   :nofigs:

   >>> from matplotlib import pyplot
   >>> from matplotlib.gridspec import GridSpec
   >>> plot = pyplot.figure(figsize=[12, 8], FigureClass=type(plot))
   >>> grid = GridSpec(3, 2, wspace=.1, hspace=.3, bottom=.15)

Next we create axes for the cleaned `TimeSeries` data:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> hax1 = pyplot.subplot(grid[0, 0], projection='timeseries')
   >>> hax1.plot(h1clean, color='#ee0000')
   >>> lax1 = pyplot.subplot(grid[0, 1], projection='timeseries')
   >>> lax1.plot(l1clean, color='#4ba6ff')

and do the same for the Q-transforms:

.. plot::
   :include-source:
   :context:
   :nofigs:

   >>> hax2 = pyplot.subplot(grid[1:, 0], projection='timeseries')
   >>> hax2.plot(h1q, cmap='Reds_r', vmin=0, vmax=80)
   >>> lax2 = pyplot.subplot(grid[1:, 1], projection='timeseries')
   >>> lax2.plot(l1q, cmap='Blues_r', vmin=0, vmax=80)

Finally, we clean up the figure:

.. plot::
   :include-source:
   :context:

   >>> for ax in plot.axes:
   >>>     ax.set_xlim(1126259462.31, 1126259462.45)
   >>>     ax.set_epoch(1126259462.42)
   >>> for ax in (hax1, lax1):
   >>>     ax.set_ylim(-1e-21, 1e-21)
   >>>     ax.set_xlabel('')
   >>> for ax in (hax2, lax2):
   >>>     ax.set_xlabel('Time [milliseconds]')
   >>>     ax.set_yscale('log')
   >>>     ax.set_ylim(20, 600)
   >>> for ax in (lax1, lax2):
   >>>     ax.get_yaxis().set_ticklabels([])
   >>>     ax.get_yaxis().set_ticklabels([], minor=True)
   >>>     ax.set_ylabel('')
   >>> hax1.set_ylabel('Strain')
   >>> hax1.text(1.0, 1.001, 'LIGO-Hanford', ha='right', va='bottom', transform=hax1.transAxes)
   >>> lax1.text(1.0, 1.001, 'LIGO-Livingston', ha='right', va='bottom', transform=lax1.transAxes)
   >>> plot.show()
