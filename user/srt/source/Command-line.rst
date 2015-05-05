******************************************
Command-line measurements and acquisitions
******************************************

Once the system and telescope setup had been completed, it is possible to manually perform measurements and observations **through the TPB**, which might as well pave the way – as preliminary checks – to longer lasting sessions carried out via schedules. 

Raw counts readouts
===================

The **raw counts readout** (called Tpi) of the signal can be obtained with::


Tsys
====

To measure the Tsys value::



Weather parameters
==================

The **weather station** measurements can be retrieved with::

.. figure:: images/Screenshot-meteoClient.py-2.png
   :scale: 80%
   :alt: meteoClient panel
   :align: center

**Notice the top graph: the red line corresponds to wind speed = 61 km/h, at which the antenna must be stowed.** 



Manual acquisitions
===================

When performing manually commanded acquisitions, it is necessary to **select the recording device**:: 

	* **MANAGEMENT/FitsZilla** if FITS output is desired


Pointing scans
-------------- 

Command cross-scans across a previously selected target (by means of the track or sidereal commands)::


	* W = -2.7725887 \* F^2  
The results are given in the logfile, in the following sequence of lines:
|                latoff_err  lonflag  latflag 
|              latampl_err  lonFWHM  lonFWHM_err  latFWHM  latFWHM_err  *(cont.)*  
|              flux  lonflag  latflag  

|                -1 = fit did not converge)
|                -1 = fit did not converge)
   

.. note:: it is possible to **include such pointing scans using the MANAGEMENT/Point writer in schedules** as well. For example, an improved pointing can be achieved setting the first scan on a source as a /Point scan, then – in case the fit is successful – the following scans (e.g. producing FITS or MBFITS files) will hold the offsets optimising the pointing, given that no user-defined offset is updated by means of an explicit *radecOffsets*, *azelOffsets* or *lonlatOffsets* command.


Focus scans
----------- 

Command a focus scan on a previously selected target (by means of the track or sidereal commands)::


Skydips
------- 

Skydip scans are indispensable in order to characterize the atmosphere. They consist in moving the telescope along a vast span in elevation (at fixed azimuth) while sampling with a backend. Their analysis allows the user to quantify the atmospheric opacity τ. 

The jolly character is supported for the elevation arguments. Example: skydip=*,*,00:04:00 will perfom the skydip between the default values for elevation (15° and 90°). Please notice that pointing corrections are disabled.


Caveat on offsets
==================

As seen in Antenna operations, there are commands used to set (or null) user-defined offsets.  

   