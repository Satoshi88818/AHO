Abstract Hardware Optimizer (AHO) v9

A Multi-Physics Rocket Engine Nozzle Design & Optimization Framework

Table of Contents

Overview

Potential Uses

Architecture

Physics & Integrations

Requirements

Deployment & Installation

Usage

Configuration & Parameters

Output Files

Current Limitations

Future Enhancements

Reference Engine Benchmarks

Contributing

Overview

AHO (Abstract Hardware Optimizer) is a single-file, multi-physics framework for the preliminary design and constrained optimization of liquid-propellant rocket engine nozzles. Starting from a set of operating conditions (chamber pressure, temperature, propellant flow rate), AHO simultaneously optimizes nozzle geometry, wall thickness, regenerative cooling, film cooling, and structural margins to maximize a user-selected objective function.

Version 9 adds three major physics integrations on top of the v8 baseline:

Integration 6 — Transient thermal analysis and Low-Cycle Fatigue (LCF) life prediction using the Manson-Coffin model

Integration 7 — Supercritical heat transfer via the Bishop correlation with wall/bulk density correction

Integration 8 — 2D/axisymmetric turbulent boundary layer growth model and improved shock loss prediction

The framework is written entirely in Python, designed to be extensible to other hardware domains beyond nozzles, and produces STL geometry exports and publication-quality plots.

Potential Uses

Academic & Research

Preliminary design studies for CH₄/LOX rocket engines (Raptor-class)

Parametric trade studies: Isp vs. mass vs. fatigue life

Validation of 1D/2D thermal and fluid models against higher-fidelity CFD

Teaching tool for rocket propulsion engineering courses

Industry / Commercial Space

Rapid first-pass nozzle sizing before committing to full CAD/FEA cycles

Sensitivity analysis across operating envelope (O/F ratio, chamber pressure, coolant inlet temperature)

Design space exploration for derivative engines sharing a propellant combination

Generating baseline geometries (STL) for import into CAD tools

Mission & Systems Engineering

Rocket stage trade studies requiring consistent Isp and mass estimates

Coupling with trajectory optimizers that need parametric engine performance models

Evaluating design margins for reusability requirements (minimum LCF life)

Architecture

AHO is built around a plugin/strategy pattern using Python abstract base classes. This separates physics, geometry, evaluation, and optimization concerns cleanly, and allows new hardware domains to be registered without modifying core framework code.

┌─────────────────────────────────────────────────────┐ │ CLI / main() │ └──────────────────────┬──────────────────────────────┘ │ ┌────────▼────────┐ │ HardwareDomain │ ◄── @register_domain("nozzle") └────────┬────────┘ ┌───────────────┼───────────────────┐ │ │ │ ┌──────▼──────┐ ┌──────▼──────┐ ┌─────────▼──────────┐ │ Axioms │ │ Evaluator │ │ GeometryGenerator │ │ (SymPy) │ │ (physics) │ │ (Rao contour/STL) │ └─────────────┘ └──────┬──────┘ └────────────────────┘ │ ┌───────▼──────┐ │EvaluatorCache│ ◄── memoized dict └───────┬──────┘ │ ┌───────▼──────┐ │ Optimizer │ ◄── SLSQP via scipy └──────────────┘ 

Core Classes

ClassRoleHardwareDomainAbstract top-level interface; wires all components togetherAbstractAxiomsSymbolic (SymPy) physics relations; evaluated numerically at runtimeAbstractEvaluatorComputes all performance metrics from a parameter dictEvaluatorCacheTransparent memoization wrapper around any evaluatorAbstractGeometryGeneratorNozzle contour generation, mass calculation, STL exportAbstractSimulatorCore isentropic flow simulationAbstractOptimizerWraps scipy.optimize.minimize with domain-specific objectives and constraintsAbstractConstraintsData class holding material and design limit constantsCombustionPropertiesTemperature-dependent γ and Cp lookup tables 

Domain Registration

New hardware domains (e.g., turbopumps, injectors, tanks) can be added by subclassing HardwareDomain and decorating with @register_domain("my_domain"). No core framework code needs modification.

@register_domain("turbopump") class TurbopumpDomain(HardwareDomain): ... 

Physics & Integrations

Gas Dynamics (all versions)

Isentropic flow relations expressed as SymPy symbolic expressions and evaluated numerically

Temperature-dependent specific heat ratio γ(T) via table interpolation (2500–4000 K)

Iterative Mach-from-area-ratio solver using Brent's method (scipy.optimize.brentq)

Real Gas Effects

Peng-Robinson equation of state for compressibility factor Z (methane/LOX)

Methane critical properties: Tc = 190.6 K, Pc = 4.6 MPa, ω = 0.011

Z applied to chamber conditions and local flow stations

Nozzle Geometry (Rao Parabolic Approximation)

Optimum nozzle angles (θ_n, θ_e) interpolated from lookup tables vs. log(expansion ratio)

Bézier curve divergent section between throat-end and exit points

Circular arc throat transition radius = 0.382 × r_t

Cylindrical chamber + 45° conical convergent section

L* characteristic chamber length calculation

Regenerative Cooling (Station-by-Station Marching)

Coolant marches exit → inlet (counter-flow)

Bartz correlation for gas-side heat transfer coefficient with σ correction factor

Variable channel geometry: aspect ratio peaks near throat

Darcy-Weisbach friction pressure drop per segment

Full LCH₄ property tables (ρ, μ, k, Cp, Pr) from 111 K to 600 K across the supercritical range

Integration 1 — Film Cooling

Smooth exponential decay model: η_film = f × exp(−s / L_decay)

Applied as a heat flux reduction along the wetted arc length

Integration 2 — Near-Critical Coolant Handling

Two-phase/near-critical flagging when T_cool > 180 K

Temperature capped at 1000 K to prevent divergence

Integration 4 — Nozzle Wall Friction Loss

Turbulent flat-plate skin friction (Schlichting): cf = 0.455 / (log₁₀ Re)^2.58

Integrated over full wetted length → η_friction

Integration 5 — Chamber Heat Flux Augmentation

Multiplier applied to Bartz h_g in the chamber zone only (x < 0)

Integration 6 — Transient Thermal & LCF Life (NEW in v9)

1D Fourier conduction through throat wall: ΔT = q_max × t_throat / (2k)

Constrained thermal strain range: Δε = α × ΔT / (1 − ν)

Manson-Coffin Low-Cycle Fatigue: N_f = (Δε / 2ε_f)^(1/c)

Default material constants approximating OFHC copper alloy: ε_f = 0.3, c = −0.6

Minimum life constraint: 50 cycles (configurable via NozzleConstraints.min_life)

Integration 7 — Supercritical Heat Transfer (NEW in v9)

Switches from Dittus-Boelter to Bishop correlation when T_cool > 193 K: Nu = 0.0069 × Re^0.9 × Pr^0.66 × (ρ_wall/ρ_bulk)^0.43

Wall/bulk density correction using Peng-Robinson for coolant-side density

Integration 8 — Turbulent Boundary Layer Model (NEW in v9)

Turbulent momentum-integral method marching inlet → exit

Turbulent skin friction: cf = 0.025 × Re_θ^−0.25

Constant shape factor H = 1.4

Displacement thickness δ* = H × θ → effective exit radius → η_bl

Improved underexpansion shock loss: linear penalty proportional to (P_e − P_a)/P_a

Nozzle Efficiency Budget

η_nozzle = λ_div × η_bl × η_shock × η_friction 

where λ_div is the divergence efficiency = 0.5(1 + cos α).

Requirements

Python Version

Python 3.9 or later

Dependencies

numpy>=1.23 scipy>=1.9 sympy>=1.11 matplotlib>=3.6 

Install all dependencies via:

pip install numpy scipy sympy matplotlib 

Or using a requirements.txt:

numpy>=1.23 scipy>=1.9 sympy>=1.11 matplotlib>=3.6 pip install -r requirements.txt 

Hardware

No GPU required; all computation is CPU-bound

RAM: < 512 MB for typical single runs

Typical runtime: 5–30 seconds per optimization (depends on mode and convergence)

Deployment & Installation

Local (Development)

# 1. Clone or copy the file git clone https://github.com/your-org/aho.git cd aho # 2. Create a virtual environment (recommended) python -m venv venv source venv/bin/activate # Linux/macOS venv\Scripts\activate # Windows # 3. Install dependencies pip install -r requirements.txt # 4. Run a quick validation python AHOv9_integrated.py --domain nozzle --mode balanced --validate 

Docker

FROM python:3.11-slim WORKDIR /app COPY AHOv9_integrated.py requirements.txt ./ RUN pip install --no-cache-dir -r requirements.txt ENTRYPOINT ["python", "AHOv9_integrated.py"] docker build -t aho:v9 . docker run --rm -v $(pwd)/outputs:/app/outputs aho:v9 \ --domain nozzle --mode balanced --plot --output-dir /app/outputs 

Batch / HPC

For large parametric sweeps, wrap main() directly in a Python loop or use multiprocessing.Pool:

from AHOv9_integrated import main from multiprocessing import Pool configs = [ {"opt_mode": "balanced", "target_isp": 310.0}, {"opt_mode": "min_mass", "target_isp": 320.0}, {"opt_mode": "max_isp", "max_mass": 6.0}, ] with Pool(processes=3) as pool: pool.starmap(main, [(c.get("opt_mode", "balanced"),) for c in configs]) 

Usage

Command-Line Interface

python AHOv9_integrated.py [OPTIONS] Options: --domain {nozzle} Hardware domain to optimize (default: nozzle) --mode {balanced, Optimization objective mode (default: balanced) min_mass, max_isp} --target-isp FLOAT Minimum Isp target in s, used in min_mass mode (default: 315.0) --max-mass FLOAT Maximum mass in kg, used in max_isp mode (default: 8.0) --validate Print comparison against reference engines (sea-level) --vacuum-validate Also print vacuum reference engine comparison --plot Save Isp/area-ratio and nozzle profile plots to PNG --output-dir DIR Directory for STL and plot outputs (default: current dir) --list-domains List all registered hardware domains and exit 

Examples

# Balanced optimization with plots and validation python AHOv9_integrated.py --domain nozzle --mode balanced --plot --validate # Minimize mass while achieving at least 320 s Isp python AHOv9_integrated.py --mode min_mass --target-isp 320 # Maximize Isp subject to 6 kg mass limit, save outputs to ./results/ python AHOv9_integrated.py --mode max_isp --max-mass 6.0 --output-dir ./results # Check which domains are available python AHOv9_integrated.py --list-domains 

Programmatic API

from AHOv9_integrated import get_domain domain = get_domain("nozzle") optimizer = domain.build_optimizer(opt_mode="balanced", target_isp=315.0, max_mass=8.0) params = { "R": 400.0, "T0": 3500.0, "P0": 10e6, "Pa": 101_325.0, "m_dot": 5.0, "throat_radius": 0.05, "t_chamber": 0.006, "t_throat": 0.008, "t_divergent": 0.004, "length_factor": 0.8, "Me": 3.0, "use_real_gas": True, "OF": 3.5, "T_cool_in": 120.0, "eta_c_star": 0.97, "L_star": 1.20, "film_cooling_fraction": 0.08, "film_decay_length": 0.40, "chamber_heat_flux_multiplier": 1.25, } results = optimizer.optimize_design(params) print(f"Isp = {results['results']['Isp']:.1f} s") print(f"Life = {results['results']['life']:.0f} cycles") 

Configuration & Parameters

Key Operating Conditions

ParameterDefaultUnitsDescriptionT03500.0KChamber stagnation temperatureP010×10⁶PaChamber stagnation pressurePa101 325PaAmbient pressure (sea level)m_dot5.0kg/sTotal propellant mass flow rateR400.0J/kg·KSpecific gas constant of exhaustOF3.5—Oxidizer-to-fuel mixture ratioT_cool_in120.0KCoolant (LCH₄) inlet temperatureeta_c_star0.97—C* efficiency (combustion quality) 

Design Variables (Optimized)

VariableRangeDescriptionMe2.0 – 5.0Exit Mach numberlength_factor0.6 – 1.0Rao nozzle length fractiont_chamber3 – 12 mmChamber wall thicknesst_throat4 – 15 mmThroat wall thicknesst_divergent2 – 8 mmDivergent section wall thicknessL_star0.8 – 1.8 mCharacteristic chamber lengthfilm_cooling_fraction0 – 15%Film coolant mass fractionfilm_decay_length0.15 – 0.80 mFilm cooling decay lengthchamber_heat_flux_multiplier1.05 – 1.45Chamber Bartz augmentation factor 

Constraints Enforced

ConstraintDefault LimitNo overexpansion at sea levelP_exit ≥ P_ambientHoop stress (3 zones)σ ≤ 200 MPa / 1.5Wall temperatureT_wall ≤ 1700 K / 1.5Coolant pressure dropΔP ≤ 12 MPaMinimum LCF lifeN_f ≥ 50 cycles 

Output Files

FileDescriptionnozzle_v9_integrated.stlASCII STL of full nozzle/chamber 3D geometry (36-facet revolution)nozzle_results_v9_integrated.pngTwo-panel plot: isentropic A/A* curve + nozzle radial profile 

Console output includes all key performance metrics, optimized design variable values, and optional reference engine comparison table.

Current Limitations

Material Model Inconsistency

The NozzleConstraints class uses steel properties (ρ = 7 800 kg/m³, k = 20 W/m·K, α = 17×10⁻⁶ /K) for mass and conduction calculations, while the Manson-Coffin fatigue constants (ε_f = 0.3, c = −0.6) approximate OFHC copper. These represent two different materials and produce physically inconsistent results. A real design would use a single consistent material model (e.g., GRCop-84 or Inconel 718).

Hardcoded Reynolds Number in Friction Loss

Re_exit_approx = 1e6 in Integration 4 is a constant, making η_friction independent of actual flow conditions. This can over- or under-predict friction loss significantly at off-design points.

Coolant Pressure Assumption in Bishop Correlation

The supercritical coolant pressure is set to P_cool = P0 × 1.2 rather than being integrated from the actual Darcy-Weisbach ΔP accumulation. This decouples the Bishop correlation from the real coolant pressure state.

Throat Zone Detection

The throat zone boundary condition abs(x_mid) < 0.03 × throat_radius is extremely narrow, typically misclassifying most near-throat segments as "divergent" and applying the thinner divergent wall thickness where the thicker throat thickness should apply.

Single Propellant Combination

The thermodynamic tables and LCH₄ coolant property tables are hardcoded for methane/LOX. Running the optimizer with different propellants (e.g., RP-1/LOX, H₂/LOX) would require replacing these tables entirely.

No Injector Aerodynamics

The injector is modeled only as a mass (proportional to chamber face area) with no injection pattern, spray atomization, or mixing efficiency model.

1D Thermal Model Only

The transient thermal analysis (Integration 6) uses a simple through-wall 1D conduction approximation. Circumferential and axial heat conduction, fin effects between cooling channels, and unsteady ramp-up profiles are not captured.

EvaluatorCache Not Thread-Safe

The memoization cache uses a plain Python dict with no locking, making it unsafe for parallel multi-threaded optimization calls.

SLSQP Sensitivity to Initial Conditions

SLSQP is a local optimizer. Results depend on the initial parameter values and can converge to local optima. No multi-start or global search strategy is implemented.

STL Surface Only

The exported STL represents only the inner wetted surface revolved around the axis. Cooling channels, flanges, injector face, and structural features are absent from the geometry export.

Future Enhancements

Near-Term (Low Effort)

Consistent material database: Replace hardcoded constants with a Material dataclass (GRCop-84, Inconel 718, SX-copper) carrying all thermal, structural, and fatigue properties together

Real coolant pressure tracking: Pass accumulated ΔP into the Bishop correlation instead of using the P0 × 1.2 approximation

Throat zone fix: Replace the narrow 0.03 × r_t threshold with a proper zone boundary at the inflection point of the contour profile

Dynamic Re in friction loss: Compute Re from actual local gas properties rather than using the hardcoded 10⁶

Medium-Term

Multi-start optimization: Run SLSQP from N randomized initial points and return the global best — straightforward with multiprocessing.Pool

Differential evolution / genetic algorithm: Replace or supplement SLSQP with scipy.optimize.differential_evolution for global search

CEA integration: Replace the polynomial γ(T) tables with NASA CEA (Chemical Equilibrium with Applications) calls via rocketcea Python wrapper for accurate multi-species thermochemistry

Parametric propellant support: Abstract the combustion property and coolant tables behind a Propellant class to support RP-1/LOX and LH₂/LOX combinations

Cooling channel geometry optimization: Add channel width, height, and number of channels as additional design variables rather than using fixed heuristics

Long-Term

2D FEA thermal solver: Replace the 1D conduction model with a proper finite-element solver (e.g., FEniCS or FEATool) for accurate temperature fields and stress distributions

CFD-in-the-loop: Call OpenFOAM or SU2 as a subprocess for high-fidelity flow evaluation at promising design points identified by the 1D optimizer

Turbopump domain: Implement a TurbopumpDomain using the existing domain registry to co-optimize nozzle and turbopump in a coupled system

Solid closed STL export: Add wall thickness offset to produce a watertight solid body suitable for direct 3D printing or CNC toolpath generation

Web interface: Wrap main() with a FastAPI backend and a React frontend for browser-based parameter input and live result visualization

Uncertainty quantification: Propagate manufacturing tolerances and material property scatter through the optimizer using Monte Carlo sampling or polynomial chaos expansion

Reference Engine Benchmarks

AHO compares its optimized design against the following published reference engines at sea level:

EngineIsp (s)C* (m/s)Mass (kg)Raptor v2 (sea-level)3271 8557.5Merlin 1D (sea-level)2821 6715.1BE-4 (sea-level, est.)3101 83013.0 

Vacuum references (not directly comparable to sea-level optimization):

EngineIsp (s)C* (m/s)Mass (kg)Raptor v2 (vacuum)3551 8557.5Typical CH₄/LOX (CEA vac)3601 8709.0 

Contributing

Fork the repository and create a feature branch

Follow existing class/method naming conventions

All new physics integrations should be documented with a comment block referencing the source correlation (author, year, equation number)

Add the new integration to the docstring header in the format established by Integrations 1–8

Ensure _validate_results() still passes after any changes to the evaluate() return dict

Run the built-in validation before submitting a PR:

python AHOv9_integrated.py --validate --vacuum-validate 

AHO is a preliminary design tool. Results should be validated against higher-fidelity analysis (CFD, FEA) before use in any engineering decision.

