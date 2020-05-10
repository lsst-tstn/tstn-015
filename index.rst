..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.


:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

    This technote describes the optical collimation technique used with the Auxiliary Telescope.
    It is based on the curvature wavefront sensing technique used for the main telescope.
    Accompanying the technote is a jupyter notebook that walks the user through performing hexapod offsets.

Introduction
============
This document describes how to use the Curvature WaveFront Sensing algorithm for optical collimation during
operation of the Auxiliary Telescope.

The Auxiliary Telescope is an F/18 Ritchey-Chretien with a 1.2 meter diameter near-spherical mirror.
The primary mirror directs the light to a 218mm diameter convex secondary which directs the light to a tertiary (flat) fold mirror which rotates to direct the light into one of the two nasmyth ports.
The primary mirror is supported by 24 pneumatic actuators and 3 hardpoints, where the pressure is adjusted as a function of elevation (see `TSTN-012 <https://tstn-012.lsst.io/>`__).
The secondary mirror is mounted to a PI H-824 Hexapod which is primarily controlled from the Auxiliary Telescope Active Optics System (ATAOS) Controllable SAL Component (CSC).
Although the hexapod has 6 degrees of freedom, the mechanical mounting is believed to be sufficiently accurate such that to first-order any tilts can be mapped as a decenter of the mirror.
This means that after rough collimation (utilizing an alignment scope and laser), the alignment of the telescope consists of 4 free parameters: primary mirror pressure, hexapod X and Y decenter, and hexapod piston (defocus).

For the purposes of this document, the important parameters are:

+------------------------------------+-------------------+
| Effective Focal length             |  21.6 m           |
+------------------------------------+-------------------+
| Magnification between secondary    |  41               |
| motion and back-focal distance     |                   |
+------------------------------------+-------------------+
| Pixel pitch                        |  10um             |
+------------------------------------+-------------------+
| Plate Scale                        |  100 um/arcsec    |
|                                    |  0.1 arcsec/pixel |
+------------------------------------+-------------------+
| Secondary Obscuration Diameter     |  0.423m           |
+------------------------------------+-------------------+

The optical design (in Zemax) for the telescope and LATISS instrument can be found in `Document-27702 <https://docushare.lsst.org/docushare/dsweb/Get/Document-27702>`__ (credentials required).
The original alignment was performed by Roberto Tighe using an alignment laser with the telescope at ~30 degrees elevation, a short report on the original first light (by eye) alignment is found `in Document-35621 <https://docushare.lsst.org/docushare/dsweb/Get/Document-35621>`__.

.. 1.5Arcsec coma/mm is another interesting number that came out of the alignment

Telescope collimation and focus can be done in a variety of ways. We opted to use CWFS techniques because the main camera for LSST uses this technique during standard operation.

Implications and Limitations of the Technique
=============================================

The CWFS technique for collimation (and focus) is highly affected by seeing, particularly dome seeing.
Experience has shown that long (~30s) exposures are required to average out the effects of dome seeing to have a stable solution.
Seeing variations of ~20nm for the defocus and coma Zernikes is to be expected.

The telescope has very good optics (0.06 waves RMS) and therefore image quality is fully seeing limited.
The CWFS method measures the wavefront error (WFE) introduced by the optics and their alignment beyond the seeing limit therefore so long as the total WFE is at (or below) the diffraction limit of the system then optimizing beyond this point is of no use.
Using a diffraction limit of :math:`\lambda / 4 \sim 158 \mathrm{nm}` for a wavelength of 632nm, and assuming all the WFE comes from the defocus through spherical aberration terms (4-11), then this allows each term to have a magnitude of :math:`158 / \sqrt{8} \sim 55\mathrm{nm}`.
Therefore, we consider any focus or coma abberation magnitude below 50nm to be acceptable and not require further optimization as the PSF will be heavily seeing dominated.

One should note that the ~158 nm guideline for an acceptable amount of total WFE is not a hard truth and breaks down in certain cases.
For example, if one has 158nm of coma and all others aberrations are zero, then you will see very significant coma in the PSF!
Using the ~50 nm guideline for an acceptable amount of WFE *per zernike term* is also not a hard truth.
In exceptional seeing conditions (which we've not encountered to date), the instrumental PSF may begin to have an effect.
To date, the PSF quality is generally dominated by dome seeing and tracking error, but this is expected to improve during the commissioning period.

In correcting the collimation/focus using the hexapod it is important to understand the limits of motion.
Motion of the hexapod and how it relates to WFE correction is defined by a sensitivity matrix.
The derivation of this matrix is discussed in detail in `TSTN-016 <tstn-016.lsst.io>`__.
For the purposes of this document, a user must know how the physical limitations of the hexapod affect the correction, which is defined by the minimum motions of each axis, which is ~0.3um.
However, we have seen instabilities of ~1um of motion. Using the sensitivity matrix, 1um of motion converts to WFEs of ~4.4nm of defocus, and 0.16nm of coma in each direction.

Due to the above discussion, we strongly encourage that **no corrections are be applied for motions below 2um in Z and 50 um in X and Y.**
These values correspond to ~10nm of WFE.
The motion reduction is recommendation order to reduce unnecessary wear on the hexapod, unnecessary heating, and addition of observing overhead.

Another aspect of the CWFS solution which is not yet well understood is strong variations in astigmatism measurements.
The only mechanism to have any control on astigmatism is to adjust the pressure under the mirror, however testing has shown that small corrections are not easily achieved.
In :numref:`fig-fit-example` below there is an example of a large coefficient term for astigmatism.
These are believed to be fitting issues as they can vary strongly from image to image.
These issues are still being tracked down.


.. figure:: /_static/cwfs_fit_example.jpg
    :name: fig-fit-example

    This shows what is believed to be a false measurement in astigmatism, which will often vary significantly
    from image to image. In this case, only a focus offset should be applied as the coma values are acceptable.

LatissCWFSAlign script
======================

To collimate and focus the Auxiliary telescope, the scriptQueue script `LatissCWFSAlign <https://github.com/lsst-ts/ts_externalscripts/blob/develop/python/lsst/ts/externalscripts/auxtel/latiss_cwfs_align.py>`__
is used.
The script can be easily run from a notebook, which allows users to exercise a finer control over the process, while it is still not fully automated.
The script proceeds from a "best focus" position, which can be a rough guess is required, then pistons the hexapod by a set amount,
generally +/-0.8mm, which corresponds to +/-32.8 mm at the focal plane.
Testing has shown this to be an adequate amount, balancing the amount of data required to calculate the low order Zernikes with keeping the image cutout size small to reduce the overhead of calculating the result.

The CWFS calculation is currently performed using `the package developed by Bo Xin <https://github.com/bxin/cwfs>`__.
Note that this package (and our script) outputs Annular Zernike Noll Polynomials.
Because the AuxTel system only (normally) exercises 3 of the six degrees of freedom adjustable via the hexapod (decenter X,Y and piston/focus in Z), the script only reports these values.

Usage of the script is best described by using the example notebook and comments there-in.

Example Notebook
================

An example notebook on how to use this script can be found in the examples folder of
`ts_notebooks <https://github.com/lsst-ts/ts_notebooks/tree/develop/examples/operations>`__.
The notebook shows how to run the script from the summit, with comments contained therein on other functionality.


Where to run the notebook
^^^^^^^^^^^^^^^^^^^^^^^^^
The example notebook above is meant to be run from the summit system during the night. However, it will be possible to
run the script at the NCSA-integration-teststand in the near future. Upon this being implemented more information will
be added here.

.. note::
    Another possiblity currently being explored is creating mocks of the high level classes and required lower-level
    aspects. This would allow the scripts to be run from the LSP at NCSA.

Future Work
===========

The current version of the script performs the focus/collimation in open loop. In the future a closed-loop version will
also be created which will apply the offsets and return when all WFE terms are sufficiently small.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
