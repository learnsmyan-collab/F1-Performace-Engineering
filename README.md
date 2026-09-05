# f1 race strategy & tire degradation models

python toolkit for simulating f1 race strategy, compound degradation crossovers, and monte carlo traffic/wear risk spreads. designed to evaluate pit windows and stint performance profiles.

## what's in here
- `tire_strategy.py`: models linear wear and exponential thermal degradation for hard vs. soft compounds, automatically calculating the exact pit window crossover lap and generating a comparison plot (`compound_crossover.png`).
- [Tire Strategy Degradation ][output/compound_crossover.png]
- `monte_carlo_stint.py`: runs 10,000-iteration probabilistic simulations over a stint to map p50 median expectations against p90 worst-case traffic/wear risk lines (`monte_carlo_stint.png`).
- [Tire Strategy Stint ][output/monte_carlo_stint.png]

## governing equations
### 1. tire degradation & crossover model
lap times incorporate base pace, linear mechanical wear, and non-linear thermal degradation:
- $\text{LapTime} = \text{base\_time} + (\text{wear\_rate} \cdot \text{lap}) + (\text{thermal\_factor} \cdot \text{lap}^{1.5} \cdot 0.04)$

### 2. monte carlo stint risk spread
stint variations are sampled from a normal distribution and scaled non-linearly over distance:
- $\text{StintTime} = (\text{base\_lap} \cdot \text{laps}) + (\text{deg\_samples} \cdot \text{laps}^{1.1})$

## quick setup
1. make sure you have dependencies installed:
   ```bash
   pip install numpy pandas matplotlib
