# 🏎️ Motorsport Analytics: Pit Strategy & Stint Risk Models
 
Two focused, tested Python simulations for race strategy analysis:
 
1. **`tire_strategy.py`** — models per-lap pace for two tire compounds and
   finds the lap that *minimizes total race time* once a real pit-stop
   time loss is priced in — not just the pace crossover.
2. **`stint_risk_model.py`** — a genuine lap-by-lap Monte Carlo simulation
   combining tire-wear noise with independently modeled traffic incidents,
   fully reproducible via a seed.
Both scripts are CLI tools with validated inputs, dataclass configs (no
magic numbers buried in function bodies), and a pytest suite that
exercises the actual logic — including a real bug the tests caught and
fixed during development (see below).
 
---
 
## 1. Compound Degradation & Pit-Window Optimizer (`tire_strategy.py`)
 
**Pace model**, per compound:
 
```
LapTime(lap) = base_time
             + wear_rate * lap
             + thermal_factor * thermal_scaling * lap ** thermal_exponent
```
 
`wear_rate` is the linear mechanical component; the thermal term captures
the "cliff" — degradation that accelerates non-linearly (`thermal_exponent`,
default 1.5) later in the stint.
 
**What's new vs. the original version:**
- `optimal_pit_lap()` compares *every* candidate pit lap **and** "never
  pit" against each other, using a real pit-loss constant (seconds lost
  boxing). The pace crossover lap is still reported, but it's now labeled
  as a pace signal — pitting exactly on the crossover lap is very often
  *not* optimal once pit loss is priced in.
- **A real bug, caught by tests, fixed in this version:** the original
  optimizer computed a "no-stop total time" for reference but never
  actually compared it against the best pitting option — so it could
  recommend pitting even when staying out was faster. `test_optimal_pit_lap_declines_to_pit_when_loss_outweighs_gain`
  reproduces this and pins the fix.
- All coefficients (base pace, wear rates, thermal factors, pit loss,
  race length) are CLI flags, not hardcoded.
- Input validation: negative wear rates, zero laps, non-positive base
  times, and negative pit losses all raise `ValueError` instead of
  silently producing garbage.
```bash
python tire_strategy.py                                   # defaults, 25 laps
python tire_strategy.py --laps 40 --pit-loss 22 --soft-wear 0.12
```
 
Example output (defaults — 22s pit loss, 25-lap race):
```
Pace crossover: soft becomes slower than hard at lap 24
Strategy result: do NOT pit — with a 22.0s pit loss, every pit option
is at least as slow as running the whole race on soft.
```
With a longer race or faster soft degradation, the same optimizer
correctly recommends pitting:
```bash
python tire_strategy.py --laps 40 --soft-wear 0.12 --pit-loss 22
# Strategy result: box on lap 18 (soft -> hard, 22.0s pit loss)
#   — saves 22.02s vs. running soft the whole race.
```
 
---
 
## 2. Monte Carlo Stint & Traffic Risk Model (`stint_risk_model.py`)
 
**What changed vs. the original version:** the original drew a *single*
random degradation value per simulated race and scaled it by
`laps ** 1.1` — that's one noisy number per run, not a stint simulation,
and despite the name it never modeled traffic at all. This version:
 
- Simulates **every lap of every run independently** as a `(runs, laps)`
  numpy array (10,000 runs × 20 laps = 200,000 simulated lap-events).
- Models **two distinct risk sources**, each independently configurable:
  1. **Wear noise** — Normal per lap, scaled up over the stint via
     `wear_exponent` (a tire's pace variance grows as it ages).
  2. **Traffic** — an independent Bernoulli draw *each lap*
     (`traffic_prob_per_lap`), with a Lognormal time penalty applied only
     when the event fires (time losses are strictly positive and
     right-skewed — most traffic costs a little, rare incidents cost a
     lot). This is the mechanism the original code's docstring claimed to
     model but never actually implemented.
- **Reproducible**: `StintConfig.seed` feeds `numpy.random.default_rng`.
  Same config → identical output (verified by test); pass `--seed -1` for
  fresh randomness each run.
- Reports P10/P50/P90 *and* mean traffic incidents per stint, so the
  traffic contribution is visible, not just implied.
```bash
python stint_risk_model.py
python stint_risk_model.py --runs 50000 --laps 30 --traffic-prob 0.12
```
 
The resulting histogram now visibly shows a sharp peak for incident-free
runs plus a right-skewed tail for stints that hit one or more traffic
events — a qualitatively different (and more honest) distribution shape
than a single Gaussian.
 
---
 
## Repository Layout
 
```text
motorsport-strategy/
├── tire_strategy.py           # Compound degradation + pit-loss-aware optimizer
├── stint_risk_model.py        # Lap-by-lap Monte Carlo: wear + traffic
├── tests/
│   ├── test_tire_strategy.py      # 10 tests incl. the pit-optimizer bug fix
│   └── test_stint_risk_model.py   # 8 tests incl. seed reproducibility
├── output/                    # Generated PNGs (git-ignored in practice)
├── requirements.txt
└── README.md
```
 
## Installation & Usage
 
```bash
git clone https://github.com/SmyanAggarwal/motorsport-strategy-dashboards.git
cd motorsport-strategy-dashboards
pip install -r requirements.txt
python tire_strategy.py
python stint_risk_model.py
pytest tests/ -v
```
 
## Known Limitations (stated explicitly rather than left implicit)
 
- Both models are deterministic-pace-plus-noise curve fits, not physics
  simulations — coefficients are illustrative defaults, not calibrated
  against real telemetry. Treat them as a strategy-logic sandbox, not a
  source of real lap-time predictions.
- The Monte Carlo traffic model treats each lap's incident probability as
  independent and identically distributed; real traffic risk is
  correlated with grid position, pace delta to the car ahead, and race
  phase (denser directly after a start or safety-car restart). A future
  version could condition `traffic_prob_per_lap` on lap number.
- `optimal_pit_lap` assumes exactly one pit stop and two compounds; it
  does not search multi-stop strategies.
 
 
