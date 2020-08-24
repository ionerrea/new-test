::: {#notebook .border-box-sizing tabindex="-1"}
::: {#notebook-container .container}
::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
SnTe example[¶](#SnTe-example){.anchor-link} {#SnTe-example}
============================================

In this tutorial we are going to study the thermoelectric transition in
SnTe. To speedup the calculations, we will use a force-field that
correctly reproduces the physics of ferroelectric transition in Fcc
lattice.

We will replicate the calculations performed in the paper by [Bianco et.
al. Phys. Rev. B 96,
014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111).

The force field can be downloaded and installed from
[here](https://github.com/mesonepigreco/F3ToyModel).

Initialization[¶](#Initialization){.anchor-link} {#Initialization}
------------------------------------------------

As always, we need to initialize the working space. This time we will
initialize first the force field. This force field needs the harmonic
dynamical matrix to be initialized, and the higher parameters. We will
initialize it in order to reproduce the same as in [Bianco et. al. Phys.
Rev. B 96,
014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111).
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[ \]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    # Import the cellconstructor stuff
    import cellconstructor as CC
    import cellconstructor.Phonons

    # Import the modules of the force field
    import fforces as ff
    import fforces.Calculator

    # Import the modules to run the sscha
    import sscha, sscha.Ensemble, sscha.SchaMinimizer
    import sscha.Relax, sscha.Utilities

    # Load the dynamical matrix for the force field
    ff_dyn = CC.Phonons.Phonons("ffield_dynq", 3)

    # Setup the forcefield with the correct parameters
    ff_calculator = ff.Calculator.ToyModelCalculator(ff_dyn)
    ff_calculator.type_cal = "pbtex"
    ff_calculator.p3 = 0.036475
    ff_calculator.p4 = -0.022
    ff_calculator.p4x = -0.014
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We initialized a force field, for a detailed explanations of the
parameters, refer to the [Bianco et. al. Phys. Rev. B 96,
014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111)
paper. Now ff_calculator behaves like any
[ASE](https://wiki.fysik.dtu.dk/ase/) calculator, and can be used to
compute forces and energies for our SSCHA. Note: this force field is not
able to compute stress, as it is defined only at fixed volume, so we
cannot use it for a variable cell relaxation.

Now it is the time to initialize the SSCHA. We can start from the
harmonic dynamical matrix we got for the force field. Remember, SSCHA
dynamical matrices must be positive definite. Since we are studying a
system that has a spontaneous symmetry breaking at low temperature, the
harmonic dynamical matrices will have imaginary phonons. We must enforce
phonons to be positive definite to start a SSCHA.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[2\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    # Initialization of the SSCHA matrix
    dyn_sscha = ff_dyn.Copy()
    dyn_sscha.ForcePositiveDefinite()

    # Apply also the ASR and the symmetry group
    dyn_sscha.Symmetrize()
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We must now prepare the sscha ensemble for the minimization. We will
start with a \$T= 0K\$ simulation
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[3\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    ensemble = sscha.Ensemble.Ensemble(dyn_sscha, T0 = 0, supercell = dyn_sscha.GetSupercell())
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We can now proceed with the sscha minimization. Since we start from a
dynamical matrix that is very far from the correct result, it is
convenient to use a safer minimization scheme. We will use the fourth
root minimization, in which, instead of optimizing the dynamical matrix
itself, we will optimize its fourth root. Then the dynamical matrix
\$\\Phi\$ will be obtained as:

\$\$ \\Phi = \\left(\\sqrt\[4\]{\\Phi}\\right)\^4 \$\$

This constrains \$\\Phi\$ to be positive definite during the
minimization. Moreover, this minimization is more stable than the
standard one. If you want further details, please look at [Monacelli et.
al. Phys. Rev. B 98,
024106](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.98.024106).

We also change the Kong-Liu effective sample size threshold. This is a
value tha decrease during the minimization, as the parameters gets far
away from the starting point. It estimates how the original ensemble is
good to describe the new parameters. If this value goes below a give
threshold, the minimization is stopped, and a new ensemble is extracted.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[4\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    minim = sscha.SchaMinimizer.SSCHA_Minimizer(ensemble)

    # Lets setup the minimization on the fourth root
    minim.root_representation = "root4" # Other possibilities are 'normal' and 'sqrt'

    # To work correctly with the root4, we must deactivate the preconditioning on the dynamical matrix 
    minim.precond_dyn = False

    # Now we setup the minimization parameters
    # Since we are quite far from the correct solution, we will use a small optimization step
    minim.min_step_dyn = 1 # If the minimization ends with few steps (less than 10), decrease it, if it takes too much, increase it

    # We decrease the Kong-Liu effective sample size below which the population is stopped
    minim.kong_liu_ratio = 0.2 # Default 0.5 
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We can setup the automatic relaxation, to avoid the need to restart the
frequencies at each iteration. We will also setup a custom function to
save the frequencies at each iteration, to see how they evolves. This is
very usefull to understand if the algorithm is converged or not.

We will use ensembles of 1000 configurations for each population, and a
maximum of 20 populations.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[5\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    relax = sscha.Relax.SSCHA(minim, 
                              ase_calculator = ff_calculator,
                              N_configs = 1000, 
                              max_pop = 20)

    # Setup the custom function to print the frequencies at each step of the minimization
    io_func = sscha.Utilities.IOInfo()
    io_func.SetupSaving("frequencies.dat") # The file that will contain the frequencies is frequencies.dat

    # Now tell relax to call the function to save the frequencies after each iteration
    # CFP stands for Custom Function Post (Post = after the minimization step)
    relax.setup_custom_functions(custom_function_post = io_func.CFP_SaveFrequencies)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We are ready to start the SSCHA minimization. This may take few minutes,
depending on how powerfull is your PC. If you do not want to run this on
your machine, skip it and pass to the following .
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[ \]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    relax.relax()

    # Save the final dynamical matrix
    relax.minim.dyn.save_qe("final_sscha_T0_")
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
Plotting the results[¶](#Plotting-the-results){.anchor-link} {#Plotting-the-results}
------------------------------------------------------------

We can plot the evolution of the frequencies, as well as the Free energy
and the gradient to see if the minimization ended correctly.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[7\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    # Import Matplotlib to plot
    import numpy as np
    import matplotlib.pyplot as plt

    # Setup the interactive plotting mode
    plt.ion()

    # Lets plot the Free energy, gradient and the Kong-Liu effective sample size
    relax.minim.plot_results()
:::
:::
:::
:::

::: {.output_wrapper}
::: {.output}
::: {.output_area}
::: {.prompt}
:::

::: {.output_png .output_subarea}
![](data:image/png;base64,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%0A)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
As you can see, the free energy always decreases, and reaches a
plateaux. The gradient at the beginning increases, then it goes to zero
very fast close to convergence. To have an idea on how good is the
ensemble, check the Kong-Liu effective sample size (the last plot). We
can see that we went two times below the convergence threshold of 0.2
(200 out of 1000 configuration), this means that the we required three
calculations. We have at the end 500 good configurations out of the
original 1000. This means that the ensemble is still at its 50 % of
efficiency.

Lets have a look on how the frequencies evolve now, by loading the
frequencies.dat file that we created.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[8\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    frequencies = np.loadtxt("frequencies.dat")
    N_steps, N_modes = frequencies.shape

    #For each frequency, we plot it [we convert from Ry to cm-1]
    plt.figure(dpi = 120)
    for i_mode in range(N_modes):
        plt.plot(frequencies[:, i_mode] * CC.Units.RY_TO_CM)
    plt.xlabel("Steps")
    plt.ylabel("Frequencies [cm-1]")
    plt.title("Evolution of the frequencies")
    plt.tight_layout()
:::
:::
:::
:::

::: {.output_wrapper}
::: {.output}
::: {.output_area}
::: {.prompt}
:::

::: {.output_png .output_subarea}
![](data:image/png;base64,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%0A)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
While at the beginning the frequencies was changing a lot, in the last
population they changed more smoothly. This is a good sign that they are
converged. One frequency has a very small value. The cusps in the
frequencies is the point in which we changed the ensemble. In the
minimization we provided, this happened once sligtly above step 100, in
correspondance with the change of the ensemble into the last one. We
reached the final results with only 3 ensembles (3000 configurations
energy/forces calculations).
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
The instability[¶](#The-instability){.anchor-link} {#The-instability}
==================================================

From the frequencies, we can see that we have one SSCHA frequency that
is very low, below 10 cm-1. This is probabily a sign of instability. We
remark: the sscha frequencies are not the real frequencies observed in
experiments, but rather are linked to the average displacements of atoms
along that mode: In particular the average displacements of atoms can be
computed from sscha frequencies as (includes both thermal and quantum
fluctuations): \$\$ \\sigma\_\\mu = \\sqrt{ \\frac{1 +
2n\_\\mu}{2\\omega\_\\mu}} \$\$ where \$n\_\\mu\$ is the boson
occupation number and \$\\omega\_\\mu\$ is the SSCHA frequency.

It is clear, that \$\\omega\_\\mu\$ will always be positive, even if we
have an instability. Since we have a very small mode in the SCHA
frequencies, it means that associated to that mode we have huge
fluctuations. This can indicate an instability. However, to test this we
need to compute the free energy curvature along this mode. This can be
obtained in one shot thanks to the theory developed in [Bianco et. al.
Phys. Rev. B 96,
014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111).

First of all, we generate a new ensemble with more configurations. To
compute the hessian we will use an ensemble of 10000 configurations.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[ \]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    # We reload the final result (no need to rerun the sscha minimization)
    dyn_sscha_final = CC.Phonons.Phonons("final_sscha_T0_", 3)

    # We reset the ensemble
    ensemble = sscha.Ensemble.Ensemble(dyn_sscha_final, T0 = 0, supercell = dyn_sscha_final.GetSupercell())

    # We need a bigger ensemble to properly compute the hessian
    # Here we will use 10000 configurations
    ensemble.generate(10000)

    # We now compute forces and energies using the force field calculator
    ensemble.get_energy_forces(ff_calculator, compute_stress = False)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
Now we can proceed to compute the Free energy hessian. We can choose if
neglect or not in the calculation four phonon scattering process at
higher order. Four phonon scattering processes require a huge memory
allocation for big systems, that scales as \$(3 \\cdot N)\^4\$ with
\$N\$ the number of atoms in the supercell. Moreover, it requires also
more configurations to converge.

In almost all the systems we studied up to now, we found this four
phonon scattering at high order to be neglectable. We remark, that the
SSCHA minimization already includes four phonon scattering at the lowest
order perturbation theory, thus neglecting this term only affects
combinations of one or more four phonon scattering with two three phonon
scatterings (high order dyagrams). For more details, see [Bianco et. al.
Phys. Rev. B 96,
014111](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.96.014111).
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[10\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    dyn_hessian = ensemble.get_free_energy_hessian(include_v4 = False) # We neglect high-order four phonon scattering

    # We can save it
    dyn_hessian.save_qe("hessian")
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We can print the eigenmodes of the free energy hessian to check if there
is an instability:
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[12\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    w_hessian, pols_hessian = dyn_hessian.DiagonalizeSupercell()

    # Print all the frequency converting them into cm-1 (They are in Ry)
    print("\n".join(["{:16.4f} cm-1".format(w * CC.Units.RY_TO_CM) for w in w_hessian]))
:::
:::
:::
:::

::: {.output_wrapper}
::: {.output}
::: {.output_area}
::: {.prompt}
:::

::: {.output_subarea .output_stream .output_stdout .output_text}
            -18.4862 cm-1
            -18.4862 cm-1
            -18.4862 cm-1
              0.0000 cm-1
              0.0000 cm-1
              0.0000 cm-1
             23.3091 cm-1
             23.3091 cm-1
             23.3091 cm-1
             23.3091 cm-1
             23.3091 cm-1
             23.3091 cm-1
             25.7898 cm-1
             25.7898 cm-1
             25.7898 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             48.9018 cm-1
             52.5769 cm-1
             52.5769 cm-1
             52.5769 cm-1
             52.5769 cm-1
             52.5769 cm-1
             52.5769 cm-1
             64.6604 cm-1
             64.6604 cm-1
             64.6604 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             67.9293 cm-1
             80.5268 cm-1
             80.5268 cm-1
             80.5268 cm-1
             80.5268 cm-1
            102.3613 cm-1
            102.3613 cm-1
            102.3613 cm-1
            102.3613 cm-1
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
Yes we have an immaginary phonons! We found an instability! You can
check what happens if you include the fourth order.

The phase transition[¶](#The-phase-transition){.anchor-link} {#The-phase-transition}
============================================================

Up to now we studied the system at \$T = 0K\$ and we found that there is
an instability. However, we can repeat the minimization at many
temperatures, and track the phonon frequency to see which is the
temperature at which the system becomes stable.

We can exploit the fact that our sscha package is a python library, and
write a small script to automatize the calculation.

We will simulate the temperatures up to room temperature (300 K) with
steps of 50 K. Note, this will perform all the steps above 6 times, so
it may take some minutes, depending on the PC (on a i3 from 2015, with
one core, it took 2 hours).
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[ \]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    # Define the temperatures, from 50 to 300 K, 6 temperatures
    temperatures = np.linspace(50, 300, 6)

    lowest_hessian_mode = []
    lowest_sscha_mode = []

    # Perform a simulation at each temperature
    t_old = 0
    for T in temperatures:
        # Load the starting dynamical matrix
        dyn = CC.Phonons.Phonons("final_sscha_T{}_".format(int(t_old)), 3)
        
        # Prepare the ensemble
        ensemble = sscha.Ensemble.Ensemble(dyn, T0 = T, supercell = dyn.GetSupercell())
        
        # Prepare the minimizer 
        minim = sscha.SchaMinimizer.SSCHA_Minimizer(ensemble)
        minim.min_step_dyn = 0.002
        minim.kong_liu_ratio = 0.5
        #minim.root_representation = "root4"
        #minim.precond_dyn = False
        
        # Prepare the relaxer (through many population)
        relax = sscha.Relax.SSCHA(minim, ase_calculator = ff_calculator, N_configs=1000, max_pop=5)
        
        # Relax
        relax.relax()
        
        # Save the dynamical matrix
        relax.minim.dyn.save_qe("final_sscha_T{}_".format(int(T)))
        
        # Recompute the ensemble for the hessian calculation
        ensemble = sscha.Ensemble.Ensemble(relax.minim.dyn, T0 = T, supercell = dyn.GetSupercell())
        ensemble.generate(5000)
        ensemble.get_energy_forces(ff_calculator, compute_stress = False)
        
        # Get the free energy hessian
        dyn_hessian = ensemble.get_free_energy_hessian(include_v4 = False)
        dyn_hessian.save_qe("hessian_T{}_".format(int(T)))

        # Get the lowest frequencies for the sscha and the free energy hessian
        w_sscha, pols_sscha = relax.minim.dyn.DiagonalizeSupercell()
        # Get the structure in the supercell
        superstructure = relax.minim.dyn.structure.generate_supercell(relax.minim.dyn.GetSupercell()) #
        
        # Discard the acoustic modes
        acoustic_modes = CC.Methods.get_translations(pols_sscha, superstructure.get_masses_array())
        w_sscha = w_sscha[~acoustic_modes]
        
        lowest_sscha_mode.append(np.min(w_sscha) * CC.Units.RY_TO_CM) # Convert from Ry to cm-1
        
        w_hessian, pols_hessian = dyn_hessian.DiagonalizeSupercell()
        # Discard the acoustic modes
        acoustic_modes = CC.Methods.get_translations(pols_hessian, superstructure.get_masses_array())
        w_hessian = w_hessian[~acoustic_modes]
        lowest_hessian_mode.append(np.min(w_hessian) * CC.Units.RY_TO_CM) # Convert from Ry to cm-1
        
        t_old = T

    # We prepare now the file to save the results
    freq_data = np.zeros( (len(temperatures), 3))
    freq_data[:, 0] = temperatures
    freq_data[:, 1] = lowest_sscha_mode
    freq_data[:, 2] = lowest_hessian_mode

    # Save results on file
    np.savetxt("hessian_vs_temperature.dat", freq_data, header = "T [K]; SSCHA mode [cm-1]; Free energy hessian [cm-1]")
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
We can now load and plot the results.
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[35\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    hessian_data = np.loadtxt("hessian_vs_temperature.dat")
    plt.figure(dpi = 120)
    plt.plot(hessian_data[:,0], hessian_data[:,1], label = "Min SCHA freq", marker = ">")
    plt.plot(hessian_data[:,0], hessian_data[:,2], label = "Free energy curvature", marker = "o")
    plt.axhline(0, 0, 1, color = "k", ls = "dotted") # Draw the zero
    plt.xlabel("Temperature [K]")
    plt.ylabel("Frequency [cm-1]")
    plt.legend()
    plt.tight_layout()
:::
:::
:::
:::

::: {.output_wrapper}
::: {.output}
::: {.output_area}
::: {.prompt}
:::

::: {.output_png .output_subarea}
![](data:image/png;base64,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AvXr18ff359mzZrx9ttvY8zVWWUSExMREaZNm8YHH3xA48aNCQgIoH379mzbti3XNnft2sXDDz9MjRo1CAwMpG3btqxYscKpzkcffYSIsGnTJkaOHEmtWrVo2LBh9vJvvvmG1q1bExAQQNOmTfnoo494/fXXqVTpat9gx44dueOOO1weV5MmTejVq1eBx79ixQo6depE1apVCQkJoX379sTHx2cvr1evHsOGDcu1XmRkJN26dct+vnbtWkSEhQsX8uqrrxIeHk5QUBCbN29GRJg3b57LfYsIX3/9NQAHDhzg6aef5sYbbyQwMJDrrruOvn37cujQIafXrX///gBERUUhItmvo/1v6m9/+1uufeU8joJe/yNHjjBkyBDCwsLw9/cnIiKC2NjYAl/PkqY9w0pVFJlX4PgOKwE+sAEOb4HM9Nz1fPygXturPb/hd0ClyqUerlJlbtN7sHe17UnOqQFtz/eugk3TIHpUqYV17tw5Tp92Ts5r1qzp9Hz16tXMnz+fZ599lho1amR/7f3kk08yb948hg4dyh//+Ef279/P9OnT2bFjBxs3bsxOCmNjY3niiSe49957mTJlChcvXmTGjBlERkayffv2fL9Gv3jxIp06dSIpKYkRI0ZQv359Nm3axMsvv8yJEyd45513nOp/+umnXLx4kaeffhpjDFOnTuWhhx4iMTExO55ff/2VyMhIGjRowCuvvEKVKlWIj4/n/vvv5/PPP+f+++932uaIESMICwvjzTffJDXVup371q1buffee6lXrx7jxo0jIyODv/71r9SuXdtp3YEDB/L000+zZ88ebrrppuzy77//nv3797tMCh199NFHPPXUU9x6662MGTOG6tWrs337dr7++mv69u2b77p5eeuttwgICGDUqFGkpqbStm1bbrjhBhYsWMBjjz3mVDc+Pp7rrrsuO6nesmULW7ZsYcCAAYSHh3PgwAFmzJjB1q1bSUhIIDAwkC5duvDss8/y/vvv88Ybb3DjjTcC0Lx58yLF6+r1P378OO3bt6dSpUo8//zzXHfddXz11VcMHTqUCxcu8NxzzxVpX8XCGKOPYngALQGTkJBglPIIWVnGJO005vsZxsT1M2ZiPWPeDHHxqGbMzEhjVr1mzP/WGJN+oawjV6rI9u3bZ/bt23ftG7py2ZipTazzw+V543D+vN3Uql/CZs+ebbCy8FwPu4yMDAMYX19fs2fPHqf1161bZwATHx/vVL58+XKn8nPnzpmQkBDz9NNPO9U7duyYy/Kc3nzzTRMcHGwSExOdyl966SVTqVIlc/ToUWOMMXv37jWAqVWrlvn999+z6y1evNgAZuXKldll0dHR5vbbbzfp6enZZZmZmaZdu3amRYsW2WUffvihAUx0dLTJzMx02n9MTIwJDg42x48fzy7bs2eP8fX1Nb6+vtllZ8+eNf7+/ua1115zWv+ZZ54xVatWNZcuXcrz2M+ePWuCgoJMhw4dTFpamtOyrKys7N/Dw8PNk08+mWv9jh07mrvvvjv7+Zo1awxgmjVrZlJTU53qjho1yvj7+zu9dmlpaSYkJMQMHz48u8xVvBs3bjSAiYuLyy777LPPDGA2btzoVNf+NzV+/Phc28l5HPm9/oMHDzbh4eHmzJkzTuWPPPKICQ0NzfV6OSrMeZ2QkGA/H1oaN3M47RlWqjxJPuR8m+OLJ13Xu66pw22Oo6BKjdKNU6mysPIVSPq1cHXTfnceI5wnAxdOwqzOEODGzCl1boGYos3+8P7772f33OWla9euuXr1Fi5cSI0aNejatatTz3K7du0IDAxk3bp19OnTh1WrVpGSkkL//v2d6vn5+dG2bVvWrVuX774XLlxI586dqVatmtP63bt355133mHjxo1OPaT9+/enWrVq2c+joqIA2L9/PwCnTp1iw4YNTJo0iZSUFKd99ezZk3HjxnHixAnCwsKyy4cPH46Pz9WRoBkZGfzf//0fffv2pU6dqxf5Nm/enHvuuYfVq1dnl4WGhnLfffcRFxeX3Qt85coVFixYwEMPPURgYN7TQ65atYqLFy8yZswY/P39nZZdyxjfIUOG5Brz3bdvX95++22WLl3K4MGDAVi5ciUpKSlOr69jvJcvX+b8+fPcdNNNVK1alW3btmUPjyhOOV//rKwslixZwsCBA8nKynL6u+jRoweLFi1ix44dtG/fvthjKQxNhpXyZhdPOyS/GyD5oOt6Ves6z/hQrV6phqmUR0j6FQ5tKpltn0gome260K5duwIvoMs52wTA3r17OXv2LLVq1XK5zsmTJ7PrAXTq1MllvRo18v/wvHfvXnbt2lXgfuxyDrkIDQ0FrHHHjvGMGTOGMWPG5LlNx2Q45/EnJSWRnp5O06ZNc63btGlTp2QYYNCgQSxevJjvvvuODh06sGrVKk6fPs3AgQNd7t9u3759QPHP7uGqPe+44w6aNm1KfHx8djIcHx9PWFgY0dHR2fUuXbrExIkTiY2N5dixY07jtl2NFS+JeJOSkjh//jwzZsxgxowZLtfJ+XdRmjQZ9kYeNMWPctO1tl36eesGF/bkN69/wAHVrB7fxp2tJLhmM73QTak6txS+btrv7iW4YRHu9wyXIFe9l1lZWdStW5dPP/3U5Tr2sbNZWVkAxMXFuUxo/fzyf88yxtCzZ0/+8pe/uFyes8fa19c3z+04xjN69Gini8sc5Uy+8uu9LYyYmBhq1arF3Llz6dChA3PnziU8PJwuXbpc03bt8uolzszMdFme1/H07duXqVOnkpycTEBAAMuXL2fw4MFOr+kzzzzD3LlzefHFF7nrrrsICQlBRHj00UezX9uixOpOvPb9DB48mMcff9zlOrfddluBsZQUTYa9iQdO8aMKqahtdyUdjvx4Nfk9+hNkXcldr1KglVjbL3qrexv4uP4Ho1SF5c6whMwM+HsL2ywSOS+ecyQQXAuGr/f4998mTZrw7bffEhUVlesr/Jz1AMLCwujatavb+2ncuDEXL17MM3F1lz2eypUrF3mbderUoXLlyiQmJuZa5qrMz8+Pfv36ERcXx4QJE/jyyy959tlnnb76zy/WhIQEp1kUcgoNDeX333/PVX7o0CFuvvnmAo7mqr59+zJhwgSWLFlCtWrVuHDhAv369XOqY7+RhuOFi5cuXcrVK5xX0uvr60vVqlVzxZuWllbo3tw6deoQFBREVlZWsf1dFCedWs1beOEUP8rGnbbLyoSj26zE+dMHYPINENsLvp0Kv225mgiLL9RrB51GweDl8MohGLTUSqrDW2sirNS18vWDdsPJPxHGWt52uMcnwgB9+vQhIyODCRMm5FqWkZGRnRzFxMRQtWpVJkyYwJUruT98nzqV/1jqPn36sHHjRr755ptcy5KTk11uMz9169YlMjKSmTNncuLECbfjASu57dq1K0uWLHHaxn//+99cQyTsBg4cyJkzZxgxYgSXLl3Ks0fTUY8ePQgKCmLixImkpzvP1uM4PKFJkyZ8//33ZGRc/Z+9dOlSjh8/XuA+HN1yyy20aNGC+Ph44uPjqVevHh07dnSq4+vr67RvgH/84x+5yuxzAbtK0u0fpBx98MEHhepZBqhUqRIPPvggCxYsYNeuXbmWF6YNS5L2DHsLD53iRxVCYdtuZke4cML6etaVsAir17dRJ7ihAwSElFTESimwPlwe2WqdnwjO56/tebMeEPli2cTnprvvvpsnn3yS8ePHs23bNrp160alSpXYu3cvCxcuZMaMGTzwwANUr16d6dOnM3ToUFq3bk2/fv2oWbMmhw4dYsWKFXTu3Jlp06bluZ/Ro0ezbNkyYmJiGDp0KK1ateLChQv8+uuvLFq0iKNHj1K9unu3aZ85cyZRUVFERETw1FNP0ahRI06cOMHmzZs5ceKEy3mJcxo7diyRkZF06NCBkSNHkpGRwfTp07nllltISMg9JKZt27a0aNGChQsXcsstt3DrrbcWuI/Q0FDeffddRo4cSbt27ejXrx/Vq1fn559/5vLly/z73/8GYNiwYSxdupSYmBgefvhhEhMTiYuLczk2uCB9+/Zl/PjxVK5cmZEjR+bq4b3vvvuYPXs2VatWpXnz5nz33XesX78+e2y2XatWrfDx8WHSpEmcOXMGf39/unXrRs2aNRk2bBjPPfccjz76KHfffTfbt2/nm2++KXD8uKOpU6eyYcMG2rVrx1NPPUWLFi04e/YsP/30Exs2bNAxw6oAmRnW1+u53oxd2PSulVD5VLKNEZWrY0Udn4uPw7LC/KSQ9XzyWIYb+3LxU3yufRs5j8Pt16Cwr4XDa5CVCT/MLFzbnf6v8/PQhg4zPnSyvopVSpUeXz/oN8/qZPhxljVrhF1wLatHOPJFr+gVtvvwww9p27Yts2bN4tVXX8XPz4+GDRsyaNAg7rzzzux6gwYNol69ekyePJkpU6aQkZFBeHg4UVFRDBo0KN99BAcHs3HjRiZMmMCiRYuIjY2lWrVq3HjjjYwfP57g4GC3446IiGDr1q289dZb/Pvf/yY5OZnatWvTqlUr3njjjUJto127dqxYsYKXX36Z119/nQYNGjBx4kR27NjhcqgEWL3Dr776aoEXzjkaMWIEderUYcqUKYwfPx4/Pz9atGjhNIa6V69eTJ06lWnTpvHnP/+ZNm3a8NVXX/H8888Xej92ffv25a233iI1NdXlPMbTp0/Hz8+POXPmkJaWRlRUFGvXrs01/jk8PJwZM2YwZcoUnnzySTIzM9m4cSORkZGMHDmSgwcPMnv2bL766iuio6NZs2ZN9swfhVG3bl1+/PFHxo4dy+LFi0lKSuK6664jIiKCyZOLNrNKcZGc3eSqaESkJZCQkJBAy5Yti3fjBzbCJ/cV7zaVZ2rYCW591EqCQ28o62iU8jr26bgaN25cvBvWC5fLrfvuu499+/a5vLveu+++y8svv8zhw4cJDw8vg+gUFO683rlzp30WjwhjzE53tq89w94gNdm9+pWqgG8lMLb5p/P8mZW7TJWtdk/BzfcXXE8pVbp8/aBR4XvBlGdKS0tzmq93z549rFq1yuWtkY0xfPzxx3Tt2lUT4XJOk2FvEBhacB1Hjy24tjdtk18C7cbP4tiGPZ7sbWa5uQ3ciDevbbuxjZw/T+62Ln4rLHfbWimlVKFcuXKFJk2aMHjwYBo1asSBAwf44IMPCAwM5KWXXsqud+HCBZYtW8batWvZvXt3rttHq/JHk2Fv0OBOCKpV+Cl+GtyZT51CEIdxvuraZGbAT7NLr+2UUkq55OvrS/fu3YmLiyMpKQl/f386duzIxIkTs6dEA+sGEQMGDCA0NJQ33niDe++9twyjVqVBk2FvYJ/iZ13u6XCcec8UPxWGtp1SSnkEESE2NrbAek2bNs017Zgq33SeYW8R+SdrCh/AhomySQAAD3FJREFUNp2BA9tzL5rip0LRtlNKKaU8libD3sI+xU+X13NPsRVcyyrvN097Fj2Rtp1SpUp79ZQqX0r6nNZhEt7E18+6oUbkizrFj7fRtlOqVPj4+HD58mWMMXneXlYp5T2MMWRmZlK5cuUS24cmw95Ip/jxXtp2SpUof39/UlNTOXnyJLVr19aEWCkvZozh5MmTZGZm4u/vX2L70WRYKaVUuREWFkZ6ejpnz57l3Llz+Pr6akKslBey9whnZmYSGBhIWFhYie1Lk2GllFLlho+PDw0aNODEiROkp6eTlZVV1iEppYpARKhcuTL+/v6EhYXh41Nyl7lpMqyUUqpc8fHxoW7dumUdhlLKS+hsEkoppZRSqsLSZFgppZRSSlVYmgwrpZRSSqkKS5NhpZRSSilVYWkyrJRSSimlKixNhpVSSimlVIWlU6sVn8oAiYmJZR2HUkoppVSFci35lxhjijGUiktE7ge+KOs4lFJKKaUqsAhjzE53VtBkuJiISDUgGvgNuFwKu2yClXz/AdhXCvtTxUfbzntp23knbTfvpW3nvUq77Srbfu42xqS5s6IOkygmxphzwJeltT8Rsf+6z91PQKpsadt5L20776Tt5r207byXN7WdXkCnlFJKKaUqLE2GlVJKKaVUhaXJsFJKKaWUqrA0GfZep4Cxtp/Ku2jbeS9tO++k7ea9tO28l9e0nc4moZRSSimlKiztGVZKKaWUUhWWJsNKKaWUUqrC0mRYKaWUUkpVWJoMK6WUUkqpCkuTYQ8lIp1FxOTxuDNH3Q4isklELolIkoj8U0SCyyr2ikREgkVkrIh8LSJnbe0zJI+6LWz1LtjqzhGRWi7q+YjIyyJyQETSROQXEelf4gdTwRS27UQkNo/zcI+Lutp2JUxE2orIdBHZKSIXReSwiCwQkRtd1NVzzkMUtt30fPM8ItJSRBaKyH5bnnFaRL4Vkd4u6nrlOae3Y/Z8/wR+zFGWaP9FRG4HvgF2A38G6gEvAc2AmFKKsSKrCfwVOAz8DHR2VUlE6gHfAueAV4FgrHa6RUTaGWMuO1SfALwCfIjV9n8A4kTEGGPml9BxVESFajubdGBYjrJzLupp25W80UBHYCHwC1AHeA7YJiJ3GmMSQM85D1SodrPR882z3ABUBT4BjgFVgIeBL0VkhDFmFnj5OWeM0YcHPrD+MRvgkQLqfYX1xxniUDbMtu49ZX0c5f0B+AN1bL+3sb3uQ1zUmwFcAho4lHWz1R/uUBYOXAamO5QJ1hvMb4BvWR9zeXm40XaxwIVCbE/brnTarQNQOUdZMyANmOtQpuecBz3caDc937zgAfgCO4A9DmVee87pMAkvICJVRSRXL76IhADdsd5IUhwWfQpcAPqUUogVljEm3RiTVIiqDwPLjTGHHdZdC/wP53b6A+CH9aZir2eAmVi9/ncVR9zKrbYDQER8bedcXrTtSoEx5jvj3MOEMWYvsBNo4VCs55wHcaPdAD3fPJ0xJhMrca3uUOy155wmw55vNpACpInIOhFp47DsFqyhLlsdV7C94ewAWpValCpPIhIO1CZHO9n8B+d2agVcxBr2krMeaJuWlSpY5+E52zi4912My9e2KyMiIkAYcNr2XM85L5Cz3Rzo+eaBRCRIRGqKSBMR+RPWUMxvbMu8+pzTMcOe6zKwGGsYxGngZqyxNxtFpIMxZjtQ11b3uIv1jwNRpRGoKlBB7VRDRPyNMem2uidsn5Jz1gO4voRiVHk7DkwFtmF1IPQEngFuE5HOxpgrtnradmXnMayvXv9qe67nnHfI2W6g55snexcYYfs9C1iCNe4bvPyc02TYQxljvgO+cyj6UkQWYV14MAnrDSLQtizdxSbSHJarslVQO9nrpDv8zK+eKkXGmDE5iuaLyP+wLgB5BLBf7KFtVwZE5CbgfeB7rAt8QM85j5dHu+n55tmmAYuwktU+WOOGK9uWefU5p8MkvIgxJhH4AugiIr5Aqm2Rv4vqAQ7LVdkqqJ0c66QWsp4qW+9h9Yx0cyjTtitlIlIHWIF19fojtnGMoOecR8un3fKi55sHMMbsMcasNcZ8aoy5D2u2iGW24S5efc5pMux9fsP6JBbE1a8U6rqoVxdrlglV9gpqp7O2r47sdevY3lxy1gNtU49gjEkFzgA1HIq17UqRiFQDVmJdwNPTGOP4+uo556EKaDeX9HzzWIuAtsCNePk5p8mw92mM9VXCBSABuII1LVQ2EakM3I51EZ0qY8aYo8ApcrSTTTuc22kH1sUjOa+ubu+wXJUxEamKNU/xKYdibbtSIiIBwDKsf8L3GWN2OS7Xc84zFdRu+ayn55tnsg9nqObt55wmwx4qjzu23AbcD6w2xmQZY84Ba4HHbW8WdgOxvr5YWCrBqsJYDNwnIvXtBSJyN9Y/Bcd2+gLIwLpgxF5PgJHAUZzHkasSJiIBOc4tuzew5sX82qFM264U2IaIxWNNv/SoMeb7PKrqOedBCtNuer55JhGp7aLMDxiENaTB/qHGa885vYDOc8WLSCrWH8VJrNkkhmNNaP2KQ73XbHU2iMgsrDn6/oKVMH+NKnEi8hzWV372K2B72+7EA/Av24eWicCjwDoR+QfWh5VRwK9Y0+cBYIw5IiLTgFG2N5sfgQewZgZ5rBBj65QbCmo7IBTYLiKfAfbbwfYA7sX6x/yFfVvadqXmXaxOgWVYV6g/7rjQGDPX9quec56lMO1WBz3fPNH/s835/C1WsloHayaQm4C/GGMu2Op57zlX2nf50EfhHsALwBascVIZWGNo5gBNXdSNBDZjfUI7CUwHqpb1MVSUB3AQ6w47rh4NHeq1BFZhza+YDMwFwlxszwcYY9tuOtZwmMfK+jjL46OgtsNKlOcAe23tlmZrjzGAn7ZdmbTZ+nzazOSoq+echzwK0256vnnmA+gHrAGSbPnIWdvz+13U9cpzTmxBKaWUUkopVeHomGGllFJKKVVhaTKslFJKKaUqLE2GlVJKKaVUhaXJsFJKKaWUqrA0GVZKKaWUUhWWJsNKKaWUUqrC0mRYKaWUUkpVWJoMK6WUUkqpCkuTYaWUUkopVWFpMqyUUkoppSosTYaVUkp5NRH5QUSM7bGoiNu402EbRkTuK+44lVKeSZNhpVSFlCPxye/Ruaxj9QQi8lcPTxB/AQYC/7AXiEiArQ3fyVlZRMbZls0UEQESbeu/XWoRK6U8QqWyDkAppcrIwBzPBwHdXZTvLp1wPN5fgY+A5WUdSB6OG2PmFqaiiLwFvAHMAp4xxhjgNDBXRHoCo0osSqWUx9FkWClVIeVMnETkTqB7YRMqbyYiPkBlY0xaRYtDRN4A3gQ+BEbaEmGlVAWmwySUUqoQRCRQRCaIyH4RSReRQ7bnlR3qZH8tLyIDRGSPiKSKyCYRaWGr87xtG2kislZE6uXYzw8istU2hvUH2/r7ROTJYohpqIjsBtKBzrblY0TkexE5a9vXf0TkDznXB3yBEQ7DRz6wLZ8vIntcxDZZRNJybiefOHxF5CUR2W07liQReV9EQorSXq6IyKvAOOBjYIQmwkop0J5hpZQqkIj4Al8BbYD/B/wPaAWMBpoA/XKs0g14GJiJ9T47BvhSRGYAQ4B/ArWBl7C+qr83x/q1gWXAPCAO6A98JCKpxpi4IsYUAzwGvA8kA0ds5S8CC4A5gD/wOPC5iNxjjFkLXMYaOvIJsB6YbVvvfwW8bHnJK45YoA/wb2Ca7RieA24TkWhjTGYR9weAiIwGJmDF/5QmwkopO02GlVKqYEOBKKCDMeY/9kJbj+g0EZlqjNnmUL8ZcKMx5qit3gWsC7v+BNxkjLlkKw8AXhSRusaY4w7r1weeNcbMsNWbBfwETBGR+caYrCLEdCPQwhiTmOPYGhpjUh3Wn4F1MdqfgLW2fc0VkVhgbzEMI8kVh4h0w0rCHzbGLHEo3wwsBf4ALMm5ITc8AtyAldAP00RYKeVIh0kopVTBHgV+BvaLSE37A/jGtrxLjvpf2xNhmy22nwvsibBDuQCNcqyfitVDCoBtTO2HQD3g1iLGtMZFIow9ERZLKFAV2Ay0zlm3mLiK41HgFPBtjmP5HqtnOuexuCvM9nOfLblXSqls2jOslFIFa4aVsJ7KY3ntHM8P53h+zvbztzzKQ3OU/+biojL7sISGwI4ixHTAVSUReRB4FbgFa5iEXaqr+sXAVRzNgFoU/ljc9SFWj/Q4ETlj73FXSinQZFgppQrDB2uYwit5LD+U43le41vzKpdSiClXcisi3bGGH3wDjASSgCvACKB3IePIa8iBbx7lrpJsH6yxw0PzWOdEIWPJy2XgIWAt8C8RSTbGfHaN21RKlROaDCulVMH2YY2tXVtK+6svIgE5eodvtP08WIwxPQykADHGmAx7oYg87aJuXklvMlDdRfkNbsSxD2gPfGuMuezGeoVmjLkkIr2ADcAnInLOGPNVSexLKeVddMywUkoVbAHQWEQG5VwgIkEiUqWY9xcIPOGwD3/gKeAo8GsxxpQJZOHwv0BEmgG9XNS9iOukdx9QW0SaO2yjAeDO3eoWAAG46OUWET8RqebGtvJkjEkGemANV1kkIlHFsV2llHfTnmGllCrYx1gXecWKyD1YF3b5AS2wpgOLAhKKcX+/AWNtiel+YABwMzDIYYqx4ohpOfAMsFJE4oG6wLPAf4HmOer+BMSIyB+xhi0kGmO2AnOBvwHLRGQ61gV4zwB7bDEXyBizSkQ+sR1zG6xhG5lYveGPYn0QKJY73xljjtuGh2yyxdzFGLO9OLatlPJO2jOslFIFMMZcwZoL+HWsWRb+jnU731bAO1wdulBcTmKN2e0AvI01G8JwY8yc4ozJGLMSa6xwA6yp3x7Fmnd4pYvqL2D1Sk8GPgOG2bZxAmu4xRVbrI9hTcu22q0jtsYLP4M1Y8ZkrDmBo7HmBf7RzW3lyxizH6uHOAv4WkRuLGAVpVQ5JjrdolJKeQ4R+QGoZIxpU9axeAvba3YR6AukG2POF2EblbCGgXQF4oHexphi6Y1WSnk27RlWSilVHnTFmpptdkEV89DGtn58sUWklPIKOmZYKaWUt3sesF9kV9Rp2HYB3R2e77imiJRSXkOHSSillAfRYRJKKVW6NBlWSimllFIVlo4ZVkoppZRSFZYmw0oppZRSqsLSZFgppZRSSlVYmgwrpZRSSqkKS5NhpZRSSilVYWkyrJRSSimlKixNhpVSSimlVIWlybBSSimllKqwNBlWSimllFIVlibDSimllFKqwtJkWCmllFJKVVj/HxTLWOTw6NeRAAAAAElFTkSuQmCC%0A)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
From the previous plot we can easily see that the phase transition
occurs at about 170 K. It is worth noting, as pointed out, that the
SSCHA frequency is always positive definite, and no divergency is
present in correspondance of the transition. Moreover, the Free energy
curvature is more noisy than the SSCHA one. Mainly because the error is
bigger on small frequencies for the fact that: \$\$ \\omega \\sim
\\sqrt{\\Phi} \$\$ But this is also due to the computation of the free
energy curvature itself, that requires the third order force constant
tensor, that requires more configurations to converge.

Be aware, if you study phase transition in charge density waves, like
[NbS2](https://pubs.acs.org/doi/abs/10.1021/acs.nanolett.9b00504) or
[TiSe2](https://arxiv.org/abs/1910.12709), or thermoelectric materials
like
[SnSe](https://journals.aps.org/prl/abstract/10.1103/PhysRevLett.122.075901)
or
[SnS](https://journals.aps.org/prb/abstract/10.1103/PhysRevB.100.214307)
usually the transition temperature depends strongly on the supercell
shape.

For the Landau theory of phase transition, since the SSCHA is a
mean-field approach, we expect that around the transition the critical
exponent of the temperature \$\$ \\omega \\sim T\^\\frac 1 2 \$\$

Thus it is usually better to plot the temperature versus the square of
the frequency:
:::
:::
:::

::: {.cell .border-box-sizing .code_cell .rendered}
::: {.input}
::: {.prompt .input_prompt}
In \[38\]:
:::

::: {.inner_cell}
::: {.input_area}
::: {.highlight .hl-ipython3}
    hessian_data = np.loadtxt("hessian_vs_temperature.dat")
    plt.figure(dpi = 120)
    plt.plot(hessian_data[:,0], np.sign(hessian_data[:,2]) * hessian_data[:,2]**2, label = "Free energy curvature", marker = "o")
    plt.axhline(0, 0, 1, color = "k", ls = "dotted") # Draw the zero
    plt.xlabel("Temperature [K]")
    plt.ylabel("$\omega^2$ [cm-2]")
    plt.legend()
    plt.tight_layout()
:::
:::
:::
:::

::: {.output_wrapper}
::: {.output}
::: {.output_area}
::: {.prompt}
:::

::: {.output_png .output_subarea}
![](data:image/png;base64,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iy66iA4dOjB//nyuv/76En3nzZtHs2bNikL1N998wzfffMOvf/1roqOj2bNnD7NmzWLdunVs2bKFkJAQLrnkEu6++25eeeUVHn/8cbp27QpAt27dzqne8r7+hw8fZsCAAfj7+3PvvffSrFkzPvroI26++WbS09O55557zuleIiL1Sb+OTWkeGsjx9JwyD88VZ4DmoUH069i0tkqrHGutXg6+gPMBu2XLFluRXbt22V27dlV4vCr++ukO22Ha4rO+Xv50h1fudzZvvvmmxTONqMyrUG5urgWsy+Wy8fHxJc5fuXKlBey8efNKtC9evLhE+8mTJ214eLidMmVKiX6JiYnltpf2xBNP2NDQUJuQkFCiferUqdbf398eOnTIWmvtzp07LWBbtGhhT5w4UdTvvffes4BdunRpUduwYcNsnz59bHZ2dlFbfn6+7d+/v42Liytq++c//2kBO2zYMJufn1/i/mPGjLGhoaH28OHDRW3x8fHW5XJZl8tV1JaSkmKDgoLso48+WuL8u+66y4aFhdmMjIwK33tKSopt3LixHTRokM3KyipxzO12F/05Ojra3nrrrWXOHzx4sB05cmTR58uXL7eA7dKli83MzCzR98EHH7RBQUElvnZZWVk2PDzc3nHHHUVt5dX7xRdfWMDOnTu3qO3//u//LGC/+OKLEn0Lv6eeeuqpMtcp/T7O9PWfNGmSjY6OtsePHy/RfvXVV9umTZuW+XoV582faxERpzmdL7Zs2VKYH863VcximibRgNTlpz1feeUVli9fXuJV2ogRI8qM6i1YsIDIyEhGjBjBsWPHil79+/cnJCSElStXAvDxxx+TlpbGddddV6JfQEAAF110UVG/iixYsIDhw4cTERFR4vxLL72UvLw8vvjiixL9r7vuOiIiIoo+Hzp0KAC7d+8G4OjRo6xevZqJEyeSlpZWdL2UlBQuv/xytm3bxpEjR0pc84477sDP76cf2dzcXD777DN++ctf0rp166L2bt26cdlll5U4t2nTplxxxRXMnTu3qC0vL4/58+fzy1/+kpCQkArf+8cff8zp06d55JFHCAoKKnGsOnN8f/Ob35SZ8z1x4kSys7NZuHBhUdvSpUtJS0srMQJdvN6cnByOHz9O9+7dCQsLK3c6ijeU/vq73W7ef/99fv7zn+N2u0t8X4wePZrU1FQ2bNhQI7WIiNQ1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%0A)
:::
:::
:::
:::
:::

::: {.cell .border-box-sizing .text_cell .rendered}
::: {.prompt .input_prompt}
:::

::: {.inner_cell}
::: {.text_cell_render .border-box-sizing .rendered_html}
As seen, the linear interpolation between 100 and 250 K is much better
in this plot, allowing for a more precise determination of the
transition temperature.
:::
:::
:::
:::
:::
