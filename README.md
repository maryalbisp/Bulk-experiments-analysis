# Bulk-experiments-analysis

# Blank subtraction

The average of the blank for each condition is subtracted to each well at each time point.

# Data smooth

Apply Savitzky-Golay smoothing to specified columns in the DataFrame in my case: window_length=15 polyorder=1

What window_length=15 means

For each timepoint, the algorithm looks at a window of 15 consecutive points centered on that point, and fits a polynomial to those 15 values.

So each smoothed value is influenced by:

the point itself
about 7 points before
about 7 points after

What polyorder=1 means

This means that inside each 15-point window, the algorithm fits a degree-1 polynomial, so just a straight line:

y=at+b

Then it evaluates that fitted line at the center of the window.

So with polyorder=1, the method assumes that locally, my data can be approximated by a straight line.

# Instantaneous growth rate

computes specific growth rate from OD using a three-point log-slope, detects μ values that are unusually different from the other wells at the same timepoint, removes them, and then recomputes the affected wells using only the remaining valid OD points.

The main equation is:

μ=d(ln⁡OD)dt\mu = \frac{d(\ln OD)}{dt}

μ=dtd(lnOD)

So it is computing growth rate from the slope of log(OD) vs time.

It works in 6 main stages:

Compute an initial growth-rate curve for each well
Compute the mean and standard deviation across wells at each timepoint
Mark suspicious μ points as outliers
Recalculate μ for wells that had outliers, using only the remaining valid points
Recompute the final mean ± std after cleaning
Plot before vs after ← to check

# **Step-by-step explanation**

## Inputs

The function expects a dataframe with at least these columns:

- `Time_h` → time in hours
- `Well` → well name, like `F2`, `F3`, etc.
- `Absorbance_600` → OD600

You also provide:

- `well_prefix`: selects which wells to analyze, for example all wells starting with `"F"`
- `od_threshold`: minimum OD required for the **center point** to be used
- `n_std`: how many standard deviations away from the across-well mean a point must be to be considered an outlier

---

## Step 1: Initial μ calculation

For each selected well, the code sorts data by time and computes growth rate using a **three-point centered difference** on the log-transformed OD.

For a point at time tit_iti, it uses the point before and the point after:

μi=ln⁡(ODi+1)−ln⁡(ODi−1)ti+1−ti−1\mu_i = \frac{\ln(OD_{i+1}) - \ln(OD_{i-1})}{t_{i+1} - t_{i-1}}

μi=ti+1−ti−1ln(ODi+1)−ln(ODi−1)

This gives the slope of log(OD) around the middle point.

### Why three points?

Because it gives a local estimate of the derivative without fitting the whole curve.

### Why log(OD)?

Because exponential growth becomes linear on the log scale, and the slope is the specific growth rate, i found it easier and also suitable for the noisy data at low absorbance derivative points

### Why first and last points are invalid?

Because centered difference needs one point before and one point after.

So the first and last points cannot be calculated.

### What conditions must be met to calculate μ?  (should we add another one?)

For a point iii, the function checks:

- `od_prev > 0`
- `od_curr > 0`
- `od_next > 0`
- `od_curr >= od_threshold`  ← i am setting this one because for low abs, data is too noisy, we can get peaks that do not have sense.
- `t_next - t_prev > 0`

If these are not satisfied, μ is set to `NaN` and that point is marked invalid.

### Output of this step

It builds `result_df`, with one row per well and timepoint, containing:

- `Well`
- `Time_h`
- `Absorbance_600`
- `ln_OD`
- `Specific_Growth_Rate`
- `Valid`

At this stage, `Valid=True` means “μ could be computed,” not yet “this point is biologically trustworthy.”

---

## Step 2: Compute stats before outlier removal

The helper function `compute_time_stats()` groups the data by timepoint and computes:

- the mean μ across wells
- the standard deviation of μ across wells

So for each time t, it asks:

> Across all wells at this exact timepoint, what is the average growth rate and how much do wells differ?
> 

This gives `stats_before`.

### Important detail

At this stage it uses:

```
compute_time_stats(result_df,use_valid_only=False)
```

That means it includes all μ values that are not `NaN`, regardless of whether they may later become outliers. 

So this is your **raw across-well behavior before cleaning**.

---

## Step 3: Mark outliers

Now the function checks each calculated μ point against the across-well distribution at the same timepoint.

For each point, it gets:

- `mean_val` = mean μ at that time
- `std_val` = std μ at that time

Then it defines bounds:

lower bound=mean−n_std⋅std\text{lower bound} = \text{mean} - n\_std \cdot \text{std}

lower bound=mean−n_std⋅std

upper bound=mean+n_std⋅std\text{upper bound} = \text{mean} + n\_std \cdot \text{std}

upper bound=mean+n_std⋅std

If a point falls outside this interval, it is marked as invalid:

```
result_df.at[i,'Valid']=False
```

and saved in `outlier_points`.

### Meaning of this step

This does **not** remove raw OD points.

It removes **growth-rate estimates** that look too far from the other wells at the same time.

So the logic is:

> If one well has a μ value that is unusually high or low compared with the rest at that same timepoint, treat it as suspicious.
> 

---

## Step 4: Recalculate affected wells using only valid OD points

This is the most important and most subtle part.

Once outlier μ points are detected, the function does **not simply delete those μ values and leave the others unchanged**.

Instead, for every well that had at least one outlier, it **rebuilds the growth-rate calculation**.

### Why is recalculation needed?

Because each μ value depends on a three-point neighborhood:

μi uses ODi−1,ODi,ODi+1\mu_i \text{ uses } OD_{i-1}, OD_i, OD_{i+1}

μi uses ODi−1,ODi,ODi+1

So if one point is bad, it can also distort nearby μ values.

For that reason, the function recalculates the well from scratch using only the points that remain valid.

---

### Step 4a: Identify affected wells

It finds all wells that had at least one outlier:

```
affected_wells=sorted(set([wforw,_inoutlier_points]))
```

Only those wells are recalculated.

---

### Step 4b: Build the valid-point table for each affected well

For one affected well:

- it goes back to the original OD data
- it gets the current valid points from `result_df`
- then it rebuilds the set of usable points

This part is important:

```
valid_points=well_results[well_results['Valid']==True][['Time_h','Absorbance_600']].copy()
```

This keeps only points still marked valid after outlier filtering.

Then it loops through the original OD data and adds back points that:

- are above `od_threshold`
- are **not** outliers
- are not already present

Why does it do that?

Because the first and last points may not have had a μ value themselves, but they may still be useful as neighbors when recalculating μ for nearby points.

So this step reconstructs a clean OD trajectory for the well, excluding only the suspicious points.

---

### Step 4c: Invalidate the whole well temporarily

Before recalculating, it wipes the old μ values for that well:

```
result_df.loc[mask_well,'Specific_Growth_Rate']=np.nan
result_df.loc[mask_well,'Valid']=False
```

This avoids mixing old and new values.

---

### Step 4d: Recalculate μ on the cleaned OD sequence

Now it loops through the cleaned valid points and recomputes μ using the same centered-difference formula:

μi=ln⁡(ODi+1)−ln⁡(ODi−1)ti+1−ti−1\mu_i = \frac{\ln(OD_{i+1}) - \ln(OD_{i-1})}{t_{i+1} - t_{i-1}}

μi=ti+1−ti−1ln(ODi+1)−ln(ODi−1)

But now the neighbors may be farther apart in time if some point was removed.

So instead of using immediate original neighbors, it uses the nearest remaining valid neighbors.

That means the derivative is now computed on a **cleaner, irregularly spaced sequence**.

This is actually a smart part of the function.

It means:

> “Remove suspicious points, then estimate growth rate again from the remaining trustworthy OD values.”
> 

---

## Step 5: Compute stats after outlier removal

After recalculating affected wells, it computes the mean and std again:

```
stats_after=compute_time_stats(result_df,use_valid_only=True)
```

Now only rows with `Valid == True` are used.

So this gives the cleaned summary across wells.

---

## Step 6: Plot before and after

The function plots:

- mean μ before cleaning
- mean μ after cleaning
- shaded area for ± std before
- shaded area for ± std after

This helps you visually see:

- whether outlier removal changed the average trend
- whether variability across wells decreased
- whether suspicious spikes disappeared

Glucose:

<img width="1172" height="699" alt="image" src="https://github.com/user-attachments/assets/7dc12c5c-06f1-45f1-8d89-3398dc9e6330" />

# Plateau selection

## **The core: Normalized Scoring**

Rather than filtering windows by individual thresholds (CV ≤ X, slope ≤ Y, etc.), this function:

1. **Generates all possible windows** that meet a basic high-μ requirement
2. **Normalizes** each feature (mean μ, CV, slope, duration) to a 0-1 scale
3. **Combines them with weights** into a single score
4. **Selects the best-scoring window**

This is more flexible and often finds better plateaus because it can trade off between competing objectives

note: i decided to go with coefficient of variations to measure stability of each candidate window, at maxima growth rate there is window with a plateau but data points are dispersed, using CV allow me to keep the most flat region

## **Step-by-Step Breakdown**

### **Phase 1: Preprocessing & Setup**

### **Phase 2: Window Generation with High-μ Gate**

python

```
for left_idx in range(n):
    for right_idx in range(left_idx + min_points - 1, min(n, left_idx + max_points)):
        # Only keep windows where mean μ ≥ high_mu_threshold
        if mean_mu < high_mu_threshold:
            continue
```

High-μ is the **only hard filter**; CV, slope, and duration are **scored, not thresholded**

### **Phase 3: Feature Extraction for Each Window**

For every candidate window, compute four metrics:

| **Feature** | **What it measures** | **Desired direction** |
| --- | --- | --- |
| **Mean μ** | Average growth rate | Higher is better |
| **CV** | Relative variability | Lower is better (stable) |
| ** | Slope | ** |
| **Duration** | Length in time | Higher is better |

**Slope calculation options:**

- `'linear_fit'`: Linear regression slope (more robust to noise)
- `'end_to_end'`: Simple (y_last - y_first) / (t_last - t_first)

**Why normalization is powerful:**

- Features on different scales (CV ~0.05, duration ~10 hours) become comparable
- The best window in each feature gets score 1.0, worst gets 0.0
- Relative differences matter, not absolute values

### **Phase 5: Weighted Scoring**

python

```
score = (0.40 × mean_mu_norm) + (0.25 × cv_norm) + (0.25 × slope_norm) + (0.10 × duration_norm)
```

Default weights prioritize (maybe to discuss ?)

- **40%** on having high growth rate (primary objective)
- **25%** on stability (low CV)
- **25%** on flatness (low slope)
- **10%** on length (nice to have, but not essential)

### **Phase 6: Selection & Tie-Breaking**

Sort by: `score` (descending) → `mean_mu` (descending) → `duration` (descending) → `cv` (ascending)

Glucose

<img width="1578" height="790" alt="image" src="https://github.com/user-attachments/assets/e89a8702-2d62-4add-bf22-5aa103822fe4" />

Glucose + 4µM Cm

<img width="1589" height="790" alt="image" src="https://github.com/user-attachments/assets/31796df2-e984-4d80-8dc2-a8c995de667b" />

Glucose + 8µM Cm

<img width="1589" height="790" alt="image" src="https://github.com/user-attachments/assets/5b24a912-30dd-4f71-87b0-b5036fc6c267" />

# Fluorescence/Abs (Fl not corrected yet)

The fluorescence-to-absorbance ratio was used as an estimate of relative ribosome concentration at the population level at the range found in the previous function. measurement of emissions were done at 510nm and 550 nm :

Glucose:

<img width="1587" height="889" alt="image" src="https://github.com/user-attachments/assets/3bb05c54-be38-45b9-8061-4bfad89c2e1d" />


Glucose + 4µM Cm
<img width="1583" height="889" alt="image" src="https://github.com/user-attachments/assets/7cbd1d64-393b-4357-b25e-96c0e34ef940" />


Glucose +8µM Cm
<img width="1583" height="889" alt="image" src="https://github.com/user-attachments/assets/bedc978e-17ba-4c3a-8319-bbc811fe69d2" />



# Fl/Abs vs Growth rate

<img width="989" height="789" alt="image" src="https://github.com/user-attachments/assets/a8b7c220-fd77-47ed-be1f-f85a625384e7" />

