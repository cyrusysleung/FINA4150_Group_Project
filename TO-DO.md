# Step-Up Autocallable Note Pricer - To-Do List

Note: AI generated via Claude Haiku 4.5, subject to change.

## Project Goal
Find the Coupon (XXX%) such that the Monte Carlo price of the note is 98% of its issue price, using time-dependent interest rates & dividend yields and local volatility surfaces.

---

## Phase 0: Core Infrastructure (OOP Design)

> Build the foundational classes and utilities that support the entire pricing framework.

### 0.1 Market Data Structures
- [ ] Create `RiskFreeRateCurve` class:
  - [ ] Store time-dependent risk-free rates as discrete points
  - [ ] Implement `rate(t: float) -> float` method for point-in-time lookups using cubic spline interpolation
  - [ ] Use `scipy.interpolate.CubicSpline` for 1D curve interpolation
  
- [ ] Create `DividendYieldCurve` class (per asset):
  - [ ] Store time-dependent continuous dividend yields ($q_i(t)$) as discrete points
  - [ ] Implement `yield(t: float) -> float` method using cubic spline interpolation
  - [ ] Use `scipy.interpolate.CubicSpline` for smooth curve evaluation

- [ ] Create `LocalVolatilitySurface` class (per asset):
  - [ ] Store 2D surface: $\sigma_{\text{local}}(S, t)$ (spot level vs. time) as discrete grid
  - [ ] Implement `volatility(S: float, t: float) -> float` method for 2D lookups
  - [ ] Use `scipy.interpolate.RectBivariateSpline` (cubic spline in 2D) for smooth surface interpolation
  - [ ] Methods to calibrate from implied vol surface (optional enhancement)

- [ ] Create `CorrelationMatrix` class:
  - [ ] Store 3×3 correlation matrix ($\rho$)
  - [ ] Implement Cholesky decomposition caching
  - [ ] Validate positive-definite property

### 0.2 Contract & Barrier Structures
- [ ] Create `Barrier` class:
  - [ ] Attributes: `level`, `type` (Knock-Out/Knock-In), `observation_type` (discrete/continuous)
  - [ ] Methods: `is_triggered(spot_price: float) -> bool`

- [ ] Create `Payoff` class (abstract):
  - [ ] Base class for payoff calculations
  - [ ] Methods: `calculate(S_path, coupon, observation_dates) -> list[(payoff, discount_time)]`

- [ ] Create `AutocallablePayoff` class (extends `Payoff`):
  - [ ] Implement payoff logic: coupons, knock-out redemption, knock-in protection
  - [ ] Handle minimum coupon periods

- [ ] Create `Contract` class:
  - [ ] Attributes: denomination, maturity, observation_dates, barriers, payoff_spec
  - [ ] Methods: validate contract consistency

### 0.3 Market & Configuration Classes
- [ ] Create `MarketData` class:
  - [ ] Aggregates: spot prices, risk-free curve, dividend yield curves, local vol surfaces, correlation matrix
  - [ ] Methods: `get_spot(asset_id)`, `get_rate(t)`, `get_dividend_yield(asset_id, t)`, `get_local_vol(asset_id, S, t)`

- [ ] Create `SimulationConfig` class:
  - [ ] Attributes: N_paths, dt, random_seed, variance_reduction_method
  - [ ] Methods: `validate()`, `get_time_steps()`

---

## Phase 1: Setup and Foundations

> This phase involves setting up the environment, gathering all necessary inputs, and defining the contract's parameters.

### 1.1 Environment Setup (L1)
- [x] Initialize a new Python project
- [x] Import necessary libraries:
  - `numpy` for numerical operations and random number generation
  - `scipy.stats` for the normal distribution (CDF/PPF)
  - `scipy.optimize` for the root-finding algorithm (e.g., bisect or newton)
  - `scipy.interpolate.CubicSpline` for 1D cubic spline interpolation (rate curves, dividend yields)
  - `scipy.interpolate.RectBivariateSpline` for 2D cubic spline interpolation (local volatility surfaces)

### 1.2 Market Data & Model Inputs
- [ ] Create instance of `MarketData` and populate:
  - [ ] Initial spot prices ($S_0$) for all 3 indices (KOSPI2, SPX, HSCEI)
  - [ ] Risk-free rate curve ($r(t)$) as `RiskFreeRateCurve` (USD SOFR, time-dependent)
  - [ ] Continuous dividend yield curves ($q_i(t)$) as `DividendYieldCurve` for each asset (time-dependent)
  - [ ] Local volatility surfaces ($\sigma_{\text{local}}(S_i, t)$) as `LocalVolatilitySurface` for each asset
  - [ ] 3×3 correlation matrix ($\rho$) as `CorrelationMatrix`

### 1.3 Contract Parameter Definition (Project1.pdf)
- [ ] Create `Contract` instance:
  - [ ] Set Note Denomination: 100.0 (or 1.0 if working in percentages)
  - [ ] Set Target Price: 98.0 (or 0.98)
  - [ ] Set Maturity ($T$): 3.0 (years)
  - [ ] Define Observation Dates: [0.5, 1.0, 1.5, 2.0, 2.5, 3.0]
  - [ ] Define Knock-Out (KO) Barrier: 1.0 (100% of initial spot) as `Barrier` instance
  - [ ] Define Knock-In (KI) Barrier: 0.5 (50% of initial spot) as `Barrier` instance
  - [ ] Define Minimum Coupon (per annum): 0.0001 (0.01%)
  - [ ] Assign `AutocallablePayoff` to contract

### 1.4 Simulation Parameters
- [ ] Create `SimulationConfig` instance:
  - [ ] Set number of MC paths: N_paths (start with 100,000, increase for accuracy)
  - [ ] Define time step ($\Delta t$) for continuous KI barrier checking (e.g., daily: $dt = 1/252$)
  - [ ] Select variance reduction method (e.g., `VarianceReductionType.ANTITHETIC`)
  - [ ] Set random seed for reproducibility
  - [ ] Calculate total number of time steps: $N_{steps} = T / dt$
  - [ ] Create a mapping from observation dates to simulation time steps

---

## Phase 2: Core Pricing Engine

> This is the main part of the pricer, where the simulation and payoff logic are built using OOP design.

### 2.1 Multi-Asset Path Generator (L4, L5)

#### 2.1.1 PathGenerator Class (Base)
- [ ] Create abstract `PathGenerator` class:
  - [ ] Methods: `generate(market_data, config) -> NDArray` (returns $(N_{paths}, N_{steps} + 1, 3)$ array)
  - [ ] Abstract method: `_apply_dynamics()`

#### 2.1.2 GeometricBrownianMotion with Local Volatility
- [ ] Create `GBMLocalVolPathGenerator(PathGenerator)` class:
  - [ ] Implement Cholesky decomposition on correlation matrix (cache result)
  - [ ] Main method: `generate()`:
    - [ ] Generate $(N_{paths}, N_{steps}, 3)$ array of independent standard normal randoms ($Z$)
    - [ ] Apply Cholesky matrix $L$ to correlate: `Z_corr = Z @ L.T`
    - [ ] Initialize `S` array of shape $(N_{paths}, N_{steps} + 1, 3)$ with `S[:, 0, :] = S0`
    - [ ] **Time-loop** for $t = 1$ to $N_{steps}$:
      - For each asset $i$ and each path $p$:
        - Retrieve current spot: `S_prev = S[p, t-1, i]`
        - Retrieve time-dependent drift: `r_t = market_data.get_rate(t * dt)`, `q_t = market_data.get_dividend_yield(i, t * dt)`
        - Retrieve local volatility: `sigma_local = market_data.get_local_vol(i, S_prev, t * dt)`
        - Calculate risk-neutral drift: `drift = (r_t - q_t - 0.5 * sigma_local**2) * dt`
        - Calculate stochastic term: `stoch = sigma_local * sqrt(dt)`
        - Apply GBM exact solution:
          $$S[p, t, i] = S[p, t-1, i] \cdot \exp(\text{drift} + \text{stoch} \cdot Z_{\text{corr}}[p, t-1, i])$$
    - [ ] Return complete `S` array

#### 2.1.3 Variance Reduction (Optional - L5)
- [ ] Create `VarianceReduction` base class with strategies:
  - [ ] `AntitheticVariates`: Generate $N_{paths}/2$ paths, mirror with $-Z$, append both
  - [ ] `ControlVariate`: Implement if needed for advanced optimization

### 2.2 Payoff Calculation & Pricing

#### 2.2.1 Pricer Class
- [ ] Create `Pricer` class:
  - [ ] Constructor: `__init__(market_data, contract, config, path_generator)`
  - [ ] Main pricing method: `price(coupon: float) -> float`:
    - [ ] Initialize `total_discounted_payoff = 0.0`
    - [ ] **Path-loop** for each path $p = 0$ to $N_{paths} - 1$:
      - [ ] Call `_evaluate_path(p, paths[p], coupon)` → returns `(path_payoff, path_events)`
      - [ ] Accumulate: `total_discounted_payoff += path_payoff`
    - [ ] Return `average_price = total_discounted_payoff / N_paths`

#### 2.2.2 Path Evaluation Logic
- [ ] Create `PathEvaluator` class:
  - [ ] Method: `evaluate_path(S_path, contract, coupon, market_data)`:
    - [ ] Initialize `total_discounted_payoff = 0.0`, `is_knocked_in = False`, `is_knocked_out = False`
    - [ ] **Observation-loop** for $i = 1$ to 6:
      - `current_obs_date = i * 0.5`
      - `current_obs_step = obs_steps[i]`
      - `start_step = obs_steps[i-1]` (or 0 if $i=1$)
      - **Continuous KI Check** (barrier monitoring between observation dates):
        - For each time step $t$ from `start_step + 1` to `current_obs_step`:
          - `S_t = S_path[t, :]`
          - `performance = S_t / S0`
          - `laggard_perf = min(performance)`
          - If `laggard_perf <= ki_barrier.level` and `not is_knocked_in`:
            - Set `is_knocked_in = True`
      - **Discrete KO Check** (at observation date):
        - `S_obs = S_path[current_obs_step, :]`
        - `laggard_perf_obs = min(S_obs / S0)`
        - If `laggard_perf_obs >= ko_barrier.level` (Knock-Out triggered):
          - `redemption_payoff = 100.0 * (1.0 + i * coupon)`
          - `r_obs = market_data.get_rate(current_obs_date)` (time-dependent discount rate)
          - `discount_factor = exp(-r_obs * current_obs_date)`
          - `total_discounted_payoff += redemption_payoff * discount_factor`
          - Set `is_knocked_out = True`
          - Break (path terminates)
        - Else (No KO, coupon paid):
          - `min_coupon_payoff = 100.0 * min_coupon * 0.5` (min coupon for 6 months)
          - `r_obs = market_data.get_rate(current_obs_date)`
          - `discount_factor = exp(-r_obs * current_obs_date)`
          - `total_discounted_payoff += min_coupon_payoff * discount_factor`
    - [ ] **Final Redemption** (if not knocked out):
      - `S_final = S_path[-1, :]`
      - `laggard_perf_final = min(S_final / S0)`
      - If `not is_knocked_in`: `redemption = 100.0`
      - Else: `redemption = 100.0 * min(1.0, laggard_perf_final)` (put protection)
      - `r_final = market_data.get_rate(maturity)` (time-dependent discount rate at maturity)
      - `discount_factor_final = exp(-r_final * maturity)`
      - `total_discounted_payoff += redemption * discount_factor_final`
    - [ ] Return `total_discounted_payoff`

---

## Phase 3: Solver Implementation

> This phase connects the pricer to a root-finder to solve for the unknown coupon.

### 3.1 Generate Base Paths
- [ ] Create `path_generator` instance (e.g., `GBMLocalVolPathGenerator`)
- [ ] Generate one large set of correlated paths: `base_paths = path_generator.generate(market_data, config)`
  - **Note**: Using a single, fixed set of paths makes the solver stable. Re-running MC for every guess introduces noise.

### 3.2 Create Pricer & Objective Function (L2)
- [ ] Instantiate `pricer = Pricer(market_data, contract, config, path_generator)`
  - [ ] Pre-generate `base_paths` and cache in pricer
  - [ ] Create objective function:
    ```python
    def objective_function(coupon: float) -> float:
        price = pricer.price(coupon)
        return price - target_price  # target_price = 98.0
    ```

### 3.3 Run Root-Finder (L2)
- [ ] Select a safe bracket: $a = 0.0$, $b = 0.50$ (50% p.a.)
- [ ] Verify `objective_function(a) < 0` and `objective_function(b) > 0`
- [ ] Call `coupon_result = scipy.optimize.bisect(objective_function, a, b, xtol=1e-6)`
- [ ] Print the resulting coupon: "**The implied coupon is: {coupon_result:.6%}**"

---

## Phase 4: Validation and Refinement

> This phase is for testing the model's correctness and improving its performance.

### 4.1 Model Validation (Sanity Checks)
- [ ] Create a `ModelValidator` class:
  - [ ] **Test 1**: Set `coupon = 0`, `ki_barrier = 0` (no knock-in protection). Price should be < 100 (due to min coupons & discounting)
  - [ ] **Test 2**: Set `coupon = 0.10`. Price should be higher than Test 1
  - [ ] **Test 3**: Set `ko_barrier = 5.0` (very hard to hit). Price approximates coupon-paying note without KO risk
  - [ ] **Test 4**: Set `ki_barrier = 0.0` (cannot be knocked in). Final redemption should always be 100 (no downside)
  - [ ] **Test 5**: Verify time-dependent rates are applied correctly in discounting

### 4.2 Convergence Analysis
- [ ] Create `ConvergenceAnalyzer` class:
  - [ ] Run `pricer.price(coupon_fixed)` with $N_{paths} = [10k, 50k, 100k, 500k, 1M]$
  - [ ] Store results and plot price vs. $N_{paths}$
  - [ ] Verify prices converge smoothly and stabilize

### 4.3 Local Volatility Surface Validation
- [ ] Create `LocalVolValidator` class:
  - [ ] Verify local vol surface is smooth (no sharp discontinuities)
  - [ ] Check monotonicity properties (if calibrated from market data)
  - [ ] Plot 2D surface: `volatility(S, t)` for visualization

### 4.4 Variance Reduction Testing (Optional - L5)
- [ ] Implement `VarianceReductionTester`:
  - [ ] Run pricing with and without antithetic variates
  - [ ] Compare variance reduction metrics
  - [ ] Measure computational efficiency gains

### 4.5 Sensitivity Analysis (Optional)
- [ ] Create `SensitivityAnalyzer` class:
  - [ ] Compute Greeks: Delta, Gamma, Vega, Rho for the note
  - [ ] Compute coupon sensitivity to market parameters
  - [ ] Generate sensitivity plots