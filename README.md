## Introduction:
In polyatomic molecules, algorithms such as SHAKE and RATTLE are commonly used to introduce distance constraints during MD simulations. However, if we consider molecules like carbon dioxide and water, then we can reduce the computational cost by assuming them to be completely rigid and directly integrating the equation of motion for a rigid body. I tried implementing one such integration algorithm in this project to carry out the MD simulation of 100 CO₂ molecules. The choice of CO₂ was a good test case because it allowed me to address some tricky situations, like how to handle the inertia matrix when its determinant is zero (can't be inverted directly) and how to correctly apply a thermostat when there's one missing rotational degree of freedom.

## Method:
The simulation was carried out in reduced units to ensure that the parameters appearing in the python program don't take numerical values that are too extreme to result in overflow errors. Further, the methods chosen for this simulation were selected based on their strengths in handling polyatomic molecules with both translational and rotational degrees of freedom.

### Force Field:
The combination of Lennard-Jones and electrostatic potentials was used to capture the molecular interactions between molecules. The TraPPE parameterization data[1] was used to model the non-bonded interaction between carbon dioxide molecules. To simplify the calculations, the cutoff of 10 Å (40% of box size of 25 Å) is considered for both LJ interaction and electrostatic interaction. The molecules are assumed to be rigid, so no intramolecular interaction is considered.

### Integration Algorithm:
For the translational motion of molecules, the standard leap-frog method was chosen for its simplicity and numerical stability in integrating equations of motion, as shown in Figure 1, where a, v and r denote the acceleration, velocity and position of the centre of mass of the carbon dioxide molecule. 
I implemented the mid-step implicit leap-frog algorithm for rotational dynamics using quaternions as proposed in the research note[3]. This version of the leap-frog algorithm is numerically more stable than others when integrating over long periods, thus allowing us to use larger timesteps for the simulation, as presented in the paper. Quaternions were chosen over other rotational representations (such as Euler angles) because they avoid singularities (like gimbal lock). The flow chart in Figure 1 shows the integration algorithm implemented in the project using Python. All the relevant mathematical formulations are taken from the book[6].

### Thermostat:
To control the system temperature at 300K, a Langevin thermostat is implemented for both translational and rotational motion. It was chosen over others like the Nosé-Hoover thermostat due to its simplicity, robustness in maintaining temperature in systems with large fluctuations and its ability to better control rotational dynamics in polyatomic gases[6]. This thermostat introduces a damping force and random noise, mimicking the effect of a heat bath and allowing energy exchange with the environment.
The most challenging part was properly tuning the translational and rotational damping coefficients. I tried to vary the translational coefficient first while keeping the rotational counterpart constant and monitored the time to achieve the target temperature. A damping coefficient of 4 per ps was used for the translational degrees of freedom, while a lower damping coefficient (2 per ps) was applied for rotational degrees of freedom.

### Timestep and Normalization:
The timestep in a MD simulation is primarily determined by the fastest process in the system which is rotational motion in my case. The translational motion was stable for timestep up to 5 fs. After introducing rotational motion, I observed instability at larger values of timestep like 1 fs, 2fs and 5fs.  Although the paper[3] I referenced suggested using timesteps between 2-5 fs without a thermostat, this required frequent quaternion scaling and very strict tolerances when updating the quaternions, which increased the computational cost. As a result, I used a smaller timestep of 0.5 fs for this simulation. Moreover, the quaternions were normalized every 1000 timestep to ensure numerical stability. This is essential to ensure that quaternions are only rotating the molecule and do not introduce any scaling transformations, which will violate the rigid body assumption.

### Initialization:
The system was initialized with 100 CO₂ molecules placed in a simple cubic lattice configuration with 0 linear velocity and angular momentum. A thermalization run was performed to allow the system to reach the target temperature. Energy and temperature fluctuations were monitored during this period to ensure that the system approaches equilibrium.

### Python program:
The jupyter notebook attached with the report contains all the codes required to run the simulation and reproduce the results if required. The notebook is commented properly to explain the logic behind the codes written.

## Results & Validation:

The system was initially thermalized for 2000 timesteps with a timestep of 0.5 fs, to reach the expected temperature before the production run.

The velocity distribution obtained after 2000 steps is shown, confirming that it follows the expected Maxwell-Boltzmann distribution.

After the production run of 30000 timesteps with timestep of 0.5 fs, the velocity distribution is calculated again and it follows the expected Maxwell-Boltzmann distribution, confirming that the our MD algorithm works well and produces a statistically correct distribution of equilibrium states.

The translational kinetic energy was accurately conserved over time, proving the efficiency of the velocity-verlet algorithm used for translation. The average translational KE also matches the expected value of 1.5kT (because of 3 translational degrees of freedom in carbon dioxide).

The rotational kinetic energy also seems to be fluctuating around a constant value over time, proving that the midstep leapfrog algorithm implemented using quaternions handled angular velocities and torques effectively. However, the average rotational kinetic energy is overestimated from the expected value of kT (2 rotational degrees of freedom in carbon dioxide). This can be due to following sources of error:

Normalizing quaternions every 1000 timesteps is necessary for numerical stability of the algorithm but is a completely unnatural thing to do during MD simulation. This even compromises with the time reversibility of MD simulation. This can be addressed using the approach mentioned in paper [4], where the author proposed to address the rigidity problem rigorously using constraint forces than usual quaternion rescaling.

The Langevin thermostat implemented for rotational motion requires more tuning so that both translational KE and rotational KE reaches equilibrium energies as per the equipartition theorem. Moreover, improper tuning might interfere with the natural evolution of the system. This problem is properly addressed in the paper [5], where the author proposed the use of stochastic thermostats which are appropriate in handling quaternion-based rigid body models.

Small system size simulated for a very short time of 15 picoseconds which might not be enough time to equilibrate properly.

The same cutoff was assumed for lj interaction and electrostatic interaction, which undermines the exact force acting on a particle due to long range electrostatic interactions. 

The trend in mean squared displacement vs time and radial distribution function generated after the production run is shown below. These are crucial in validating the simulation results with the experimentally known data to ensure that simulation is statistically correct. The g(r) vs r plot follows the expected behavior as observed in gases at room temperature.


The diffusion coefficient can be estimated from the slope of the mean squared displacement (MSD) over time. However, in this project, with only 100 CO₂ molecules, the system is too small to yield realistic diffusion values. Generally, simulating gases requires many more molecules and longer simulation times to produce accurate results for parameters like diffusion. Despite this, the radial distribution function g(r) and the trends in potential and kinetic energy, along with validation against Maxwell-Boltzmann predictions, show that the algorithm is performing as expected. With more computational resources and time, it could potentially provide practical values.

Conclusion:
Through this project, I tried to implement the mid-step implicit leapfrog algorithm using quaternions to model the behavior of rigid CO₂ molecules. This project taught me the complexities of simulating polyatomic gases, even simple ones like CO₂. It increased my understanding of implementing and evaluating integration algorithms to solve the equation of motion during MD simulation. This experience will help to understand and modify built-in commands like “fix rigid” in software like LAMMPS to tailor it to my usage. I understood how critical factors such as timestep size, system size, thermostat settings, and even small adjustments like quaternion scaling can significantly impact the accuracy and behavior of the simulation. The program needs a lot of improvement.


## References
[1] J.J. Potoff, and J.I. Siepmann, Vapor-liquid equilibria of mixtures containing alkanes, carbon dioxide and nitrogen, AIChE J. 47, 1676-1682 (2001).

[2] B.L. Eggimann, Y.Z.S. Sun, R.F. DeJaco, R. Singh, M. Ahsan, T.R. Josephson, and J.I. Siepmann, Assessing the quality of molecular simulations for vapor-liquid equilibria: An analysis of the TraPPE database, J. Chem. Eng. Data 65, 1330-1344 (2020).

[3] SVANBERG, B. M. RESEARCH NOTE An improved leap-frog rotational algorithm, Molecular Physics, 92(6), 1085–1088 (1997).

[4] Igor P. Omelyan; On the numerical integration of motion for rigid polyatomics: The modified quaternion approach. Comput. Phys. 1 January 1998; 12 (1): 97–103.

[5] Davidchack, Ruslan L et al. “Langevin thermostat for rigid body dynamics.” The Journal of chemical physics vol. 130, 23 (2009): 234101.

[6] Allen, M. P., & Tildesley, D. J. (2017). Computer simulation of liquids (2nd ed.). Oxford University Press.
