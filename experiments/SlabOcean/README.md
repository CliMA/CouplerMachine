# Slab Ocean

Analogous to the slab ocean of [Hansen et al. [67]](https://www.cesm.ucar.edu/models/atm-cam/docs/description/node48.html#hansen84), the evolution of the prognostic variable, surface sea surface temperature can be written as (W m$^{-2}$):

$$
\rho_o h_o c_o \frac{\partial T_{sfc}}{\partial t} = F_{tot} + Q = R_{SW} - R_{LW} - SH - LH + Q
$$
  
where

- R$_{SW} = F_{sfc}^d - F_{sfc}^u = \tau F_{sol} - \alpha(\tau F_{sol} )$ is the downward-pointing short-wave radiative flux
    - $F_{sol} = 1361$W m$^{–2}$ is the incoming solar radiation
    - $\tau = 0.9$ is the selected transmissivity of the atmosphere
    - $\alpha = 0.5$ is the selected albedo
    - $\rho_o =1026$ kg m$^{–3}$ is the selected density of the ocean (assumed constant)
    - $h_o= 2$ m is the selected ocean mixed-layer depth
    - $c_o = 3.93e3$ J kg$^{–1}$ K$^{–1}$is the selected specific heat capacity of ocean

- R$_{LW} = F_{sfc}^u - F_{sfc}^d = \epsilon \sigma \theta_{sfc}^4 - \epsilon F_{a}$ is the upward-pointing net long-wave flux
    - $\epsilon = 0.98$ is the selected emissivity or absorptivity of the surface
    - $\sigma = 5.67e^{-8}$ is the Stefan Boltzmann constant
    - $F_{a} =0$ = choosing no incoming longwave radiation from the atmosphere

- SHF = $cp_d g_a (T_{sfc} - T_a) = cp_d g_a (T_{sfc} - T_a)$
    - $g_a$ = aerodynamic conductance, which in the bulk formulation is the product of atmospheric near surface density, diffusicity coefficient over the ocean and norm of the surface-tangential winds ($g_a = \rho_a C_{H} |u_h_a|$)
    - $cp_d$ = specific heat capacity of the atmosphere at constant pressure (1004 J kg$^{–1}$ K$^{-1}$)
    - $C_{H}$ = bulk transfer coefficient for sensible heat

- LHR = $ g_w \lambda (q_{sat}(T_{sfc},p_{sfc}) - q_a)$
    - $g_w^{-1}= \rho_a C_{L} |u_h_a|$
    - $\lambda$ latent heat of vaporisation at 0C (2.5008e6 J/kg)
    - $p_{sfc}$ is the surface pressure
    - $q_{sat}$ is the saturation specific humidity at the surface
    - $q_a$ is the specific humidity in the near surface atmospheric layer
    - $C_{L}$ = bulk transfer coefficient for latent heat

- $Q = \kappa_s (\theta_{sfc}-\theta_h)/h_o$ is the Q-flux representing ocean mixing
    - $\kappa_o = 2$ W m$^{-1}$ K$^{-1}$ is the selected thermal conductivity for soil (Fourier's law)
    - $\theta_h = 280$ K is the selected $\theta$ at $h_o$

The simple representation of the radiation will later be substituted by the radiation module in Atmos and a better representation of surface variables will be supplied by Atmos's SurfaceFluxes.jl (estimates based on the MO similarity theory) and subsequently by the Atmos's EDFM module.

## Implementation in single stack

For this simple implementation, we assume:
-  a dry setup 
- atmosphere only governed by heat diffusion
- aerodynamic conductance (i.e. $g_a = |u| c_D$, with $c_D$ representing the drag coefficient) is constant . Using explicit time stepping the formulation is implemented as follows:

1) Coupler sends its field `LandSurfaceTemperature` and transforms it into Atmos field `auxiliary.T_sfc`
2) Atmos initializes its state according to the user-defined ICs
3) Atmos performs all its timesteps pithing the coupling cycle, with the boundary conditions at the coupled boundary setting the total normal flux to be equal to the flux coming from the land (`fluxᵀn.ρθ = - F_tot / cp_d`, so that $\partial_t \rho \theta = \nabla_z \sdot F_{tot}/cp_d$)
4) The same flux `F_tot` is calculated in the `source!` function and saved as `state.F_ρθ_accum`, which ensures that the flux will be integrated and accumulated in time, consistent with the Atmos time stepping. 
5) Coupler converts Atmos field `state.F_ρθ_accum` to the coupler field `EnergyFluxAtmos` 
6) Coupler sends its field `EnergyFluxAtmos` and transforms it into Land field `auxiliary.F_ρθ_prescribed`
5) Land performs its own initialization and physics (`G`), which is then added to the  `auxiliary.F_ρθ_prescribed` (corresponding to $F_{tot}$ above) in the `source!` function to solve:
$$T_s^{n+1} = T_s^n +\frac{F_{tot}^{n...}}{\rho_o c_o h_o}\Delta t$$
The $^{n}$ fields correspond to the previous Land timestep.
6) Coupler converts Land field `state.T_sfc` to the coupler field `LandSurfaceTemperature` 
7) Atmos `state.F_ρθ_accum` is reset to 0 and the cycle repeats.

## Implementation in LES (using MO similarity theory, SurfaceFluxes.jl)
- TODO

## Other issues
- e.g. when 3 models; need to reset after extracting, not before accumulation
- GPU friendliness
- storing whole MPIStateArrays for coupler fields is wasteful

## References:
- [Bonan 2019 book](https://www.cambridge.org/us/academic/subjects/earth-and-environmental-science/climatology-and-climate-change/climate-change-and-terrestrial-ecosystem-modeling?format=HB&isbn=9781107043787)
- https://www.cesm.ucar.edu/models/atm-cam/docs/description/node29.html
- http://www.met.reading.ac.uk/~swrhgnrj/teaching/MT23E/mt23e_notes.pdf
- https://link.springer.com/article/10.1007/s00382-018-4236-x?shared-article-renderer 

## TODO:
- update the above doc and code for ocean params
- figure out why naning out
- clean up naming convention