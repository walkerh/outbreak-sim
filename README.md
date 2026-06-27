# PATHOGEN — Outbreak Simulation

An interactive, browser-based epidemic simulation using a stochastic agent-based SEIRD + Vaccinated model.

## Preview

- [Stable (main)](https://htmlpreview.github.io/?https://raw.githubusercontent.com/walkerh/outbreak-sim/refs/heads/main/outbreak-sim.html)
- [Experimental (dev)](https://htmlpreview.github.io/?https://raw.githubusercontent.com/walkerh/outbreak-sim/refs/heads/dev/outbreak-sim.html)

## Usage

Open `outbreak-sim.html` in any modern browser. No server, build step, or dependencies required.

## Model

Each individual is an agent in one of six states:

| State | Meaning |
|---|---|
| **S** Susceptible | Can be infected |
| **E** Exposed | Infected but not yet infectious (latent period) |
| **I** Infected | Infectious |
| **R** Recovered | Immune |
| **D** Dead | Removed from transmission |
| **V** Vaccinated | Immune from day 0 |

State transitions are stochastic each simulated day:

- **S→E** with probability β × (I / (N − D)), where β = R₀ × γ and D = current dead count
- **E→I** with probability σ = 1 / latent period
- **I→{D,R}** departs with probability γ; conditional on departure, dies with probability μ, recovers with probability (1 − μ)

The simulation ends when no Exposed or Infected individuals remain.

## Controls

| Parameter | Description |
|---|---|
| **Pathogen preset** | Loads empirically grounded parameters for a known pathogen (Measles, Smallpox, 1918 H1N1, 2009 H1N1); vaccinated fraction is left unchanged for the user to explore |
| **R₀** (0.5–25) | Basic reproduction number — average secondary infections in a fully susceptible population |
| **Latent period** | Days from exposure to becoming infectious |
| **Infectious period** | Days an infected individual remains contagious |
| **Mortality rate μ** (log scale, 0–50%) | Fraction of infected individuals who die; slider is logarithmic so every decade of CFR gets equal resolution |
| **Vaccinated fraction** | Population already immune at day 0 |
| **Population size** | Number of simulated individuals |
| **Days/second** | Simulation playback speed |

The **Hide Vaccinated** checkbox removes vaccinated dots from the population grid and hides the V line from the time course chart, reducing visual clutter at high vaccination fractions.

## Display

- **Population grid** — each dot is one individual, colored by state; infected agents show a glow halo
- **Time course chart** — SEIRD curves over simulated days; Y-axis scaled to fit all visible series; when "Hide Vaccinated" is unchecked the axis expands to include the V line, and when checked the V line is also hidden from the chart
