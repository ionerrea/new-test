```python
%pylab
```

    Using matplotlib backend: Qt5Agg
    Populating the interactive namespace from numpy and matplotlib


# Second order phase transitions and structural instabilities

One of the main big features of the self-consistent harmonic approximation (SCHA) is that it provides a complete theoretical framework to study second order phase-transitions for structural instabilities.
Examples are charge density wave, ferroelectric materials or structural deformation.
An example application for each one of this case is reported, carried out with the SSCHA package:
1. [Bianco et. al. Nano Lett. 2019, 19, 5, 3098-3103](https://pubs.acs.org/doi/abs/10.1021/acs.nanolett.9b00504)
2. [Aseguinolaza et. al. Phys. Rev. Lett. 122, 075901](https://journals.aps.org/prl/abstract/10.1103/PhysRevLett.122.075901)
3. [Bianco et. al. Phys. Rev. B 97, 214101](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.97.214101)

According to the Landau theory of second order phase transitions, a phase transition occurs when the free energy curvature around the high-symmetry structure on the direction of the order parameter becomes negative:
![](second_order.png)

For structural phase transitions, the order parameter is associated to phonon atomic displacements. So we just need to calculate the Free energy Hessian, as:
$$
\frac{\partial^2 F}{\partial R_a \partial R_b} 
$$

here, $a$ and $b$ encodes both atomic and cartesian coordinates.
This quantity is very hard to compute with a finite difference approach, as it would require a SSCHA calculation for all possible atomic displacements (keeping atoms fixed), and the Free energy is affected by stochastic noise. Luckily, SSCHA provides an analitical equation for the free energy hessian, derived by Raffaello Bianco in the work [Bianco et. al. Phys. Rev. B 96, 014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111).
The free energy curvature can be writte in matrical form as:
$$
\frac{\partial^2 F}{\partial {R_a}\partial {R_b}} = \Phi_{ab} + \sum_{cdef} \stackrel{(3)}{\Phi}_{acd}[1 - \Lambda\stackrel{(4)}{\Phi}]^{-1}_{cdef} \stackrel{(3)}{\Phi}_{efb}
$$

Here, $\Phi$ is the SCHA force constant matrix obtained by the auxiliary harmonic hamiltonian, $\stackrel{(3,4)}{\Phi}$ are the average of the 3rd and 4th derivative of the Born-Oppenheimer energy landscape on the SCHA density matrix, while the $\Lambda$ tensor is a function of the frequencies of the auxiliary harmonic hamiltonian.

Fortunately, this complex equation can be evaluated from the ensemble with a simple function call:
```python
ensemble.get_free_energy_hessian()
```

Lets see a practical example:


```python
# Lets import all the sscha modules
import cellconstructor as CC
import cellconstructor.Phonons
import sscha, sscha.Ensemble
```


```python
# We load the SSCHA dynamical matrix for the PbTe (the one after convergence)
dyn_sscha = CC.Phonons.Phonons("dyn_sscha", nqirr = 3)

# Now we load the ensemble
ensemble = sscha.Ensemble.Ensemble(dyn_sscha, T0 = 1000, supercell=dyn_sscha.GetSupercell())
ensemble.load("data_ensemble_final", N = 100, population = 5)

# If the sscha matrix was not the one used to compute the ensemble
# We must update the ensemble weights
# We can also use this function to simulate a different temperature.
ensemble.update_weights(dyn_sscha, T = 1000)

# ----------- COMPUTE THE FREE ENERGY HESSIAN -----------
dyn_hessian = ensemble.get_free_energy_hessian()
# -------------------------------------------------------

# We can save the free energy hessian as a dynamical matrix in quantum espresso format
dyn_hessian.save_qe("free_energy_hessian")
```


    ---------------------------------------------------------------------------

    ValueError                                Traceback (most recent call last)

    <ipython-input-6-44b7e4f23edc> in <module>()
          1 # We load the SSCHA dynamical matrix for the PbTe (the one after convergence)
    ----> 2 dyn_sscha = CC.Phonons.Phonons("dyn_sscha", nqirr = 3)
          3 
          4 # Now we load the ensemble
          5 ensemble = sscha.Ensemble.Ensemble(dyn_sscha, T0 = 1000, supercell=dyn_sscha.GetSupercell())


    /home/pione/anaconda2/lib/python2.7/site-packages/cellconstructor/Phonons.pyc in __init__(self, structure, nqirr, full_name, use_format)
         99         if (type(structure) == type("hello there!")):
        100             # Quantum espresso
    --> 101             self.LoadFromQE(structure, nqirr, full_name = full_name, use_format = use_format)
        102         elif (type(structure) == type(Structure.Structure())):
        103             # Get the structure


    /home/pione/anaconda2/lib/python2.7/site-packages/cellconstructor/Phonons.pyc in LoadFromQE(self, fildyn_prefix, nqirr, full_name, use_format)
        166 
        167             if not os.path.isfile(filepath):
    --> 168                 raise ValueError("Error, file %s does not exist." % filepath)
        169 
        170             # Load the matrix as a regular file


    ValueError: Error, file dyn_sscha1 does not exist.


This code will do the trick. We can then print the frequencies of the hessian. If an imaginary frequency is present, then the system wants to spontaneosly break the high symmetry phase.

The frequencies in the free energy hessian are temperature dependent.



```python

```
