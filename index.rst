..
  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   Summary of state of the art PSF estimation tools and their suitability for the LSST alert pipeline.

Overview
========

Understanding the point-spread function is important for several Alert Production tasks, including image differencing and photometry. The LSST point spread function is expected to vary in a complex way with chip position, telescope orientation and focus, and atmospheric conditions. The PSF must be fit independently for each image, and must be fit quickly enough to not form a bottleneck for Alert Production.

The requirements for PSF quality depend on downstream operations, but are in general poorly quantified. In particular:

- the :cite:`1998ApJ___503__325A` image differencing algorithm requires that the FHWM of the PSF be known to within 10% (Reiss, priv. comm. 2017), but is otherwise self-contained
- the ZOGY image differencing algorithm requires a "high quality" PSF (Reiss, priv. comm. 2017)
- the limiting factor for our current DCR correction algorithm is noise in the images to be corrected, not errors in the PSF. Therefore, the only requirement DCR correction imposes is that the PSF not have color terms (Sullivan, priv. comm. 2017; :cite:`LDM-151`)
- PSF requirements specific to LSST photometry are not available, but PSF photometry is in general only moderately sensitive to PSF models (:cite:`2006A&A___454_1029A`), so photometry is unlikely to be the limiting application.
- PSF requirements specific to LSST astrometry are not available

In addition, the Alert Pipeline PSF fitting code needs to be robust to crowded fields (Bosch, April 2017 PST Meeting)). Source crowding may be well in excess of source fill factor :math:`\Sigma n_\mathrm{eff} \sim 0.2`, where :math:`\Sigma` is the source density and :math:`n_\mathrm{eff}` is the PSF area defined in eq. 2 of :cite:`LPM-17`. The PSF fitting code may also need to support high-frequency atmospheric components (:cite:`2012MNRAS_427_2572C`), depending on the accuracy needed for image differencing.

This note reviews recent developments on PSF modeling, both in publicly available code and in the literature. Given the lack of general-purpose metrics for PSF fitting methods (new metrics will be presented in a future DMTN), the discussion is mostly qualitative. :ref:`Section 2 <psffit-general>` covers recent software for PSF modeling of large astronomical datasets. :ref:`Section 3 <psffit-dense>` covers algorithms presented for ground-based observations of crowded fields. :ref:`Section 4 <psffit-atmo>` covers algorithms designed for atmospheric variations and other high-frequency effects.

The following notation appears in discussions of an algorithm's asymptotic complexity:

- :math:`N_\mathrm{pixels}`: the number of pixels in the image
- :math:`N_\mathrm{PSF}`: the number of pixels used to represent or fit the PSF
- :math:`N_\mathrm{star}`: the number of stars used to fit the PSF (note: in general, :math:`N_\mathrm{star} N_\mathrm{PSF} \ll N_\mathrm{pixels}`)
- :math:`N_\mathrm{cells}`: the number of cells used to model PSF spatial variations

.. _psffit-general:

General-Purpose Software
========================

.. _psffit-general-psfex:

PSFEx
-----

`PSFEx <http://www.astromatic.net/software/psfex>`_ is a generic PSF fitting tool last updated in 2014. While intended to be run as a standalone command-line program, the LSST Stack's ``meas_extensions_psfex`` provides a wrapper that lets its core functionality be used as a ``PsfDeterminer``.

.. _PSFEx Manual: http://psfex.readthedocs.io/en/latest/

When run as a command-line tool, PSFEx identifies PSF template sources by filtering an input catalog. The primary criterion is to find a locus of sources with constant half-light-radius as a function of magnitude, but the user may specify cuts on SNR, arbitrary flags, image ellipticity, or presence of zero-weight pixels. Crowded fields are handled by iteratively computing a PSF model, modeling each identified source, and removing 4-σ outliers (`PSFEx Manual`_). It is not clear how high a source density is supported by this approach, but clearly there must be *some* isolated stars for the PSFEx approach to be viable.

The Stack's version does not use the above system, instead providing an independent ``StarSelector`` that uses similar but manually specified filtering criteria, and no crowded-field handling.

PSFs are fit as a sum of basis functions, which may be pixel/nonparametric (as used by ``meas_extensions_psfex``), Gauss-Laguerre, or a custom basis. The output PSF is always presented as a supersampled image; the developers concede that this approach does not handle PSF wings or diffraction spikes well.

The pixel-basis fitter constructs an image using "sinc interpolation" (essentially kernel smoothing by a Lanczos4 function) to resample the model to the pixel grid around each star. The fit is computed by minimizing the χ\ :sup:`2` of the interpolated model across all sources.

PSFEx fits PSF variations simultaneously with the PSF itself. Variability may be taken with respect to source position, source parameters, or, for multi-image fits, image metadata. While PSFEx itself requires that this input be in SExtractor and FITS header format, respectively, ``meas_extensions_psfex`` does the appropriate translation from Sources and Exposures. PSF variations are represented by a low-order polynomial whose "coefficients" are themselves PSF images. There is no support for high-frequency components of the type discussed by :cite:`2012MNRAS_427_2572C`. The algorithm requires :math:`\mathcal{O}((N_\mathrm{PSF} N_\mathrm{coeff})^3 + N_\mathrm{PSF}^2 N_\mathrm{star} N_\mathrm{coeff})` time, where :math:`N_\mathrm{coeff}` is the number of terms in the polynomial.

PSFEx supports the use of PCA to get an optimized image basis. Presumably this is in the context of PSF variation fitting, but the documentation is very vague about what the principal component analysis produces. It can also produce PSF homogenization kernels on request.

.. _psffit-general-piff:

PIFF
----

`PIFF <http://rmjarvis.github.io/Piff/html/>`_ is a new PSF fitting tool that is still under development. As most of the current version consists of placeholders (e.g., the only PSF model supported is a single elliptical Gaussian), I will discuss this program only briefly.

PIFF is designed for wide-field telescopes and multi-chip cameras. It plans to offer fitting of PSFs as a sum of basis functions, which may be pixel/nonparametric, shapelet, or Gaussian. PSF variations will be supported (only?) with respect to position, and may be interpolated using polynomials, kriging, or :ref:`PSFEnt <psffit-atmo-psfent>`. It will also allow the user to specify the optical component of the PSF while fitting for the atmospheric contribution.

It is not clear how PIFF will select PSF template sources, except that it will involve some kind of outlier rejection scheme.

.. _psffit-dense:

PSF Fitting in Dense Fields
===========================

.. _psffit-dense-iteration:

Iterative Derivation
--------------------

:cite:`2006A&A___454_1029A` present a method to fit the PSF in dense fields for WFI on the ESO 2.2m. Beginning with crude source detections, centroided positions, and aperture fluxes, they reconstruct the PSF from visible stars, use the PSF to get new source positions and fluxes, use the improved source data to construct better PSFs, and so on. Template stars are selected to have high counts and no nearby neighbors; stars that are poor fits to the PSF can be rejected.
The PSF is nonparametric, but with smoothness and centering constraints enforced at each iteration.

Each iteration of the algorithm requires :math:`\mathcal{O}(N_\mathrm{PSF} N_\mathrm{star} + N_\mathrm{PSF} N_\mathrm{smooth})` time, where :math:`N_\mathrm{smooth}` is the number of pixels in the PSF smoothing kernel. No stopping condition is presented, although the discussion suggests the algorithm converges quickly.
Spatial variation is handled by solving an independent PSF for each section of a chip, then using bilinear interpolation to find the PSF at an arbitrary position (this adds an :math:`N_\mathrm{cells}` factor to the complexity).

While :cite:`2006A&A___454_1029A` have tested their method in Baade's Window (see their Fig. 2), they have no explicit support for cases where isolated stars do not exist. Forward modeling is used only to solve for a field's astrometry and photometry once the final PSF model is available.

Their method can use saturated stars to fit PSF wings, but they admit the extended wings are not very accurate.

.. _psffit-dense-blind:

Blind Deconvolution
-------------------

:cite:`2013aoel_confE__78S` present a deconvolution-based nonparametric PSF estimator for adaptive optics observations of extremely dense, low-SNR fields. It does not require an explicit source selection. Starting from an initial PSF (which can be quite crude) and model image, they use scaled gradient projection to solve for a better image model, followed by using the refined image to improve the PSF, and so on. The algorithm is prone to overfitting, and their reconstructed PSF starts developing holes as they go to too many iterations (their Figures 3 and 4). Each iteration requires :math:`\mathcal{O}(N_\mathrm{PSF}^3 + N_\mathrm{pixels}^3)` time. The authors say a "few hundred" iterations suffice to give a good PSF model, but give no explicit stopping condition.

Using simulated data, the authors show that they can get similar completeness and :math:`\sim 15\%` worse photometry using the reconstructed PSF compared to using the true PSF. However, their background estimation method (involving an initial pass, creation of a source-subtracted smoothed image, and a second pass) does make their final catalog about 0.5 mag shallower than it would be with a perfectly estimated background. The effect may be less prominent for non-AO PSFs, which have smaller wings.

Unlike many modern PSF fitting algorithms (e.g., :cite:`2006A&A___454_1029A`, :cite:`2012MNRAS_427_2572C`), the authors do not impose any kind of smoothness constraint on their PSF. It may be worth investigating whether, with such a constraint, the deconvolved PSF would be better-behaved, or whether the solution would diverge in some other way.

.. _psffit-dense-lupton:

Forward Modeling with Image Differencing
----------------------------------------

Section 6.11.1 of :cite:`LDM-151` mentions a proposed algorithm by Lupton & Bosch to estimate PSFs in crowded LSST fields. Starting from an initial PSF (which can be quite crude, but should be narrower than the true PSF) and source list, a model image can be created, and :cite:`1998ApJ___503__325A` image differencing run on the original image and the model. The PSF is defined as the convolution of the previous PSF with the image matching kernel, and a new source list and model image are created. Each iteration requires :math:`\mathcal{O}(N_\mathrm{pixels}^2 + N_\mathrm{PSF} N_\mathrm{kernel})` time, where :math:`N_\mathrm{kernel}` is the number of pixels in the matching kernel. It is not clear how many iterations would be required or what the stopping criterion would be.

To work on the most crowded fields, this method requires a source identification algorithm that can deal with blended stars. Spatial PSF variations can be introduced using the :cite:`1998ApJ___503__325A` matching kernel.

.. _psffit-atmo:

PSF Fitting of Short Exposures
==============================

.. _psffit-atmo-psfent:

PSFEnt
------

PSFEnt :cite:`2012MNRAS_427_2572C` is a nonparametric PSF interpolation scheme to reconstruct small-scale PSF variations using maximum-entropy fitting. While it can reproduce arbitrary structure in the PSF as a function of detector position, it requires a parametric model for the PSF and (for best performance) prior knowledge of atmospheric turbulence properties.

PSFEnt models PSF variations as a grid of independent cells that are bilinearly interpolated to get the value at a specific point, much like the Stack does. The model is divided into seven "hidden" layers that are each forced to be smooth on a different spatial scale.

PSFEnt requires iterative maximization of a function whose complexity is :math:`\mathcal{O}(N_\mathrm{cells}^3 + N_\mathrm{layers} N_\mathrm{cells})`. It has been deemed too slow even for the Level 2 Data Reduction Pipeline (Bosch, priv. comm. 2017), so it is also too slow for Alert Production.

.. _psffit-atmo-cs:

Compressive Sampling
--------------------

:cite:`2014MNRAS_443__919S` proposes a method to reconstruct small-scale PSF variations from atmospheric turbulence by using properties of :math:`1/f` random fields. The PSF model must be representable as a complex field; the authors, motivated by weak lensing work, use ellipticities.

The PSF variations are assumed to be sparse in the Fourier domain, and the sparsest solution consistent with the observations can be reconstructed by several optimization algorithms, which typically have :math:`\mathcal{O}(N_\mathrm{star} \log^2 N_\mathrm{star} + N_\mathrm{star} N_\mathrm{cells}^2 \log N_\mathrm{cells})` complexity.

.. Basis Pursuit: N_\mathrm{star} \log^2 N_\mathrm{star} + N_\mathrm{star} N_\mathrm{cells}^2 \log N_\mathrm{cells}
.. TV Minimization: same as Basis Pursuit?
.. TwIST: N_\mathrm{star} N_\mathrm{cells}^2 setup, plus N_\mathrm{star} N_\mathrm{cells} per iteration; not clear how long convergence takes

:cite:`2014MNRAS_443__919S` try their method on the :cite:`2013ApJS__205___12K` data set and find it can reconstruct ellipticity errors as well as other algorithms (and somewhat better than a polynomial fit to the PSF variations). This is encouraging, although ellipticity errors are not a metric relevant to the Alert Production pipeline.

While a single complex field is far too simple a model for LSST's purposes, it may be possible to adapt their algorithm to fit an elliptical shear of a more general model. However, it is not clear how well this algorithm can handle a combination of atmospheric and optical PSF variation; while the :cite:`2013ApJS__205___12K` data does include terms for astigmatism, defocus, and coma as well as Kolmogorov turbulence, there is no theoretical basis for modeling the optics contributions using compressive sampling.

Future Work
===========

The main priority for improving LSST PSF estimation at the time of writing is robust handling of crowded fields, which :ref:`PSFEx <psffit-general-psfex>` handles poorly (Reiss, priv. comm. 2017). Early attempts to improve crowded field handling will likely involve implementing :ref:`Lupton's image differencing <psffit-dense-lupton>`. While this algorithm is appealing because of its quadratic running time (assuming a constant bound on the number of iterations) and reuse of existing code, it is untested and may fail if it cannot find suitable PSF template stars.

The best approach may involve combining the strengths of recent algorithms. For example, :ref:`blind deconvolution <psffit-dense-blind>` is appealing because it does not require source identification, but the published algorithm requires hand tuning to avoid overfitting. A regularization scheme like that used by :cite:`2006A&A___454_1029A` may make it more stable without significantly increasing its asymptotic complexity.

Another possibility is to use multiple algorithms in different contexts. For example, we may find that Lupton's algorithm performs well on all but the most crowded fields, where source identification fails catastrophically. If so, its high speed would make it the preferred algorithm for LSST data reduction, and we could fall back to a much slower algorithm like blind deconvolution in dense fields (blind deconvolution has nearly as extreme computational requirements as :ref:`PSFEnt <psffit-atmo-psfent>`, so while it's promising from a reliability standpoint it is likely to be too expensive to run on all LSST images). So long as fewer than 2% of all images require an expensive algorithm, such a strategy can comply with LSST system requirements (:cite:`LSE-29`, LSR-REQ-0025).

References
==========

.. bibliography:: dmtn045.bib

