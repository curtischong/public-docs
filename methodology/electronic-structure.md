---
description: >-
  How electronic band structures and density of states are calculated on the
  Materials Project (MP) website.
---

# Electronic Structure

## Calculation Details

A relaxed structure associated with the canonical data in the `entries` field of a material data entry is used to run both uniform and line-mode NSCF calculations with the same functional (and U if any). Currently, only GGA (PBE) and GGA+U DOS and band structures are available from the database.

We first run a static (SCF) calculation with a uniform (Monkhorst Pack or $$\Gamma$$-centered for hexagonal systems) k-point grid determined by the standard `MPStaticSet` input set in pymatgen. The charge density is extracted from this and used for the subsequent uniform and line-mode NSCF calculations. The parameters for both of these are determined by the `MPNonSCFSet` input set in pymatgen. For more details, see the band structure workflow in [atomate](https://atomate.org/).

### Line-mode Band Structure

The line-mode NSCF calculation is run with k-points chosen along high-symmetry lines within the Brillouin zone of the material. Currently, three conventions for choosing this k-path are used, and follow the methodologies by Curtarolo et al. [\[1\]](electronic-structure.md#references), Hinuma et al. [\[2\]](electronic-structure.md#references), and Munro et al. [\[3\]](electronic-structure.md#references) Code for generating the k-paths can be found within [pymatgen](https://pymatgen.org/pymatgen.symmetry.html#pymatgen.symmetry.bandstructure.HighSymmKpath).

The Setyawan-Curtarolo band structure data is displayed on the website by default with full lines for spin-up and dashed lines for spin-down. For insulators, the band gap is computed according to the band structure. The nature of the gap (direct or undirect) as well as the k-points involved in the band gap transition are displayed. The VBM and CBMs are displayed for insulators as well by purple dots. Note that the website might not show all bands included in the calculation. These can be obtained by by downloaded the band structure data from the [API](../downloading-data/using-the-api/).

### Density of States (DOS)

The DOS displayed on the website shows the total DOS, and elemental projections by default. However, total orbital and elemental orbital projections are also calculated and available from the [API](../downloading-data/using-the-api/). Please note that the DOS data and line-mode band structure may not completely agree on all derived properties such as the band-gap due to k-point grid differences. For instance, the uniform k-point grid used to calculate the DOS might not include some specific k-points along one of the high-symmetry lines, while the line-mode band structure will.

### Material Band Gap

The band gap listed for a given material is chosen from one of its calculations. The current calculation hierarchy is as follows:

**Density of States > Line-mode Band Structure > Static (SCF) > Optimization**

## Accuracy of Band Structures

**Note**: _The term 'band gap' in this section generally refers to the fundamental gap, not the optical gap. The difference between these quantities is reported to be small in semiconductors but significant in insulators._ [\[4\]](electronic-structure.md#references)

![](../.gitbook/assets/band_gaps.png)

&#x20;_Figure 1: Experimental versus computed band gaps for 237 compounds in an internal test. The computed gaps are underestimated by an average factor of 1.6, and the residual error even after accounting for this shift is significant (MAE of 0.6 eV). We thank M. Chan for her assistance in compiling this data._

Density functional theory is formulated to calculate ground state properties. Although the band structure involves excitations of electrons to unoccupied states, the Kohn-Sham energies used in solving the DFT equations are often interpreted to correspond to electron energy levels in the solid.

The correspondence between the Kohm-Sham eigenvalues computed by DFT and true electron energies is theoretically valid only for the highest occupied electron state. The Kohn-Sham energy of this state matches the first ionization energy of the material, given an exact exchange-correlation functional. However, for other energies, there is no guarantee that Kohn-Sham eigenvalues will correspond to physical observables.

Despite the lack of a rigorous theoretical basis, the DFT band structure does provide useful information. In general, band dispersions predicted by DFT are reported to match experimental observations; one small test of band dispersion accuracy found that errors ranged from 0.1 to about 0.4 eV.[\[5\]](electronic-structure.md#references) However, predicted band gaps are usually severely underestimated. Therefore, a common way to interpret DFT band structures is to apply a 'scissor' operation whereby the conduction bands are shifted in energy by a constant amount so that the band gap matches known experimental observations.

### Band gaps

In general, band gaps computed with common exchange-correlation functionals such as the LDA and GGA are severely underestimated. Typically the disagreement is reported to be around 50% in the literature. Some internal testing by the Materials Project supports these statements; typically, we find that band gaps are underestimated by about 40% (Figure 1). We additionally find that several known insulators are predicted to be metallic.

#### Origin of band gap error and improving accuracy

The errors in DFT band gaps obtained from calculations can be attributed to two sources: 1. Approximations employed to the exchange correlation functional 2. A derivative discontinuity term, originating from the true density functional being discontinuous with the total number of electrons in the system.

Of these contributions, (2) is generally regarded to be the larger and more important contribution to the error. It can be partly addressed by a variety of techniques such as the GW approximation but typically at high computational cost.

Strategies to improve band gap prediction at moderate to low computational cost now been developed by several groups, including Chan and Ceder (delta-sol),[\[6\]](electronic-structure.md#references) Heyd et al. (hybrid functionals) [\[7\]](electronic-structure.md#references), and Setyawan et al. (empirical fits) [\[8\]](electronic-structure.md#references). (These references also contain additional data regarding the accuracy of DFT band gaps.) The Materials Project may employ such methods in the future in order to more quantitatively predict band gaps. For the moment, computed band gaps should be interpreted with caution.

## Materials Displaying Unexpected 0 eV Band Gaps

It is not uncommon to encounter materials on the Materials Project (MP) that are listed with a 0 eV band gap, even though they were previously reported (or expected) to be insulating or semiconducting. This can be surprising, especially if earlier MP data or published literature reported a nonzero band gap.

This section explains **why a 0 eV band gap might appear**, how to **determine if it is limitation of DFT, or a parsing artifact**, and how to **recompute or validate the band gap** using MP’s API and `pymatgen`.

***

### Why Might a Material Show a 0 eV Band Gap?

* **Database and Parsing Updates:** The MP team periodically improves its calculation methods and data parsing. In late 2024, MP changed how band gaps are parsed and stored, leading to updates in many materials’ reported band gaps.
* **Task Type Corrections:** Sometimes, the type of calculation used to determine the band gap is corrected in the database, which can change the reported value.
* **Ambiguities in Band Edge Detection:** Automated methods for detecting the conduction band minimum (CBM) and valence band maximum (VBM) can sometimes fail, especially for materials with complex density of states (DOS) near the Fermi level.
* **Numerical Artifacts:** In rare cases, bugs or limitations in the parsing code or the underlying calculation (e.g., Fermi level placement) can result in an incorrect 0 eV gap.

***

### How to Check if a 0 eV gap is from DFT or a parsing artifact

#### 1. Look Up the Material’s Calculation Tasks

Check which calculation (task) was used to determine the band gap. The MP API provides task IDs for band structure and DOS calculations.

{% code overflow="wrap" %}
```python
from mp_api.client import MPRester

with MPRester() as mpr:
    summ_doc = mpr.materials.summary.search(material_ids=["mp-1211100"])[0]

print("Band structure task:", getattr(summ_doc.bandstructure, "latimer_munro", None)) 
print("DOS task:", getattr(summ_doc.dos, "latimer_munro", None))
```
{% endcode %}

#### 2. Recompute the Band Gap from DOS

The most robust way to check the band gap is to recompute it from the DOS:

{% code overflow="wrap" %}
```python
from mp_api.client import MPRester

with MPRester() as mpr:
    dos = mpr.materials.electronic_structure_dos.get_dos_from_task_id('mp-1776854') 
print("Band gap from DOS:", dos.get_gap()) # Output: e.g., 5.84 eV
```
{% endcode %}

#### 3. Recompute the Band Gap from Band Structure

In some cases, the band structure object may have an incorrect Fermi level. You can reconstruct it using the VBM from the DOS:

{% code overflow="wrap" %}
```python
from mp_api.client import MPRester

from pymatgen.electronic_structure.bandstructure import BandStructure

with MPRester() as mpr:
    band_struct = mpr.materials.electronic_structure_bandstructure.get_bandstructure_from_task_id("mp-1776854")
    dos =  mpr.materials.electronic_structure_dos.get_dos_from_task_id("mp-1776854")
 
cbm, vbm = dos.get_cbm_vbm()

bs_corrected = BandStructure( [k.frac_coords for k in bs.kpoints], band_struct.bands, band_struct.lattice_rec, vbm, band_struct.labels_dict, structure=band_struct.structure )

print("Band gap from corrected band structure:", bs_corrected.get_gap()) # Output: e.g., ~6.14 eV
```
{% endcode %}

***

### Common Causes and Solutions

| Cause                             | What to Do                                      |
| --------------------------------- | ----------------------------------------------- |
| **Physical Metal/Semimetal**      | 0 eV is correct                                 |
| **Parsing Artifact or Bug**       | Recompute from DOS or band structure (see code) |
| **Database Update/Method Change** | Check release notes/changelog                   |
| **Missing Calculations/Data**     | Data may not be available for this material     |

***

### Additional Notes

* Not all materials have band structure or DOS data available. If the API returns `None` for these, the calculation has not been performed.
* As of now, the raw VASP output files are not publicly available, but the MP team is working on making these accessible.
* For more details, see the [Materials Project changelog](https://next-gen.materialsproject.org/changelog)

***

**If you continue to see unexpected 0 eV band gaps, or if you have evidence that a material should be insulating, consider reporting the issue on the** [**Materials Project forum**](https://matsci.org/)**.**

## Citation

To cite the calculation methodology, please reference the following works:

1. A. Jain, G. Hautier, C. Moore, S.P. Ong, C.C. Fischer, T. Mueller, K.A. Persson, G. Ceder., A High-Throughput Infrastructure for Density Functional Theory Calculations, Computational Materials Science, vol. 50, 2011, pp. 2295-2310. [DOI:10.1016/j.commatsci.2011.02.023](https://dx.doi.org/10.1016/j.commatsci.2011.02.023)

## Authors

1. Anubhav Jain
2. Shyue Ping Ong
3. Geoffroy Hautier
4. Charles Moore
5. Jason Munro

## References

\[1]: W. Setyawan, S. Curtarolo, High-throughput electronic band structure calculations: Challenges and tools, Computational Materials Science 2010, 49, 299-312.

\[2]: Y. Hinuma, P. Giovanni, Y. Kumagai, F. Oba, I. Tanaka, Band structure diagram paths based on crystallography Computational Materials Science 2017, 128, 140-184.

\[3]: J.M. Munro, K. Latimer, M.K. Horton, S. Dwaraknath, K.A. Persson, An improved symmetry-based approach to reciprocal space path selection in band structure calculations, npj Computarional Materials 2020, 6, 112.

\[4]: E.N. Brothers, A.F. Izmaylov, J.O. Normand, V. Barone, G.E. Scuseria, Accurate solid-state band gaps via screened hybrid electronic structure calculations., The Journal of Chemical Physics. 129 (2008)

\[5]: R. Godby, M. Schluter, L.J. Sham, Self-energy operators and exchange-correlation potentials in semiconductors, Physical Review B. 37 (1988).

\[6]: M. Chan, G. Ceder, Efficient Band Gap Predictions for Solids, Physical Review Letters 19 (2010)

\[7]: J. Heyd, J.E. Peralta, G.E. Scuseria, R.L. Martin, Energy band gaps and lattice parameters evaluated with the Heyd-Scuseria-Ernzerhof screened hybrid functional, Journal of Chemical Physics 123 (2005)

\[8]: W. Setyawan, R.M. Gaume, S. Lam, R. Feigelson, S. Curtarolo, High-throughput combinatorial database of electronic band structures for inorganic scintillator materials., ACS Combinatorial Science. (2011).
