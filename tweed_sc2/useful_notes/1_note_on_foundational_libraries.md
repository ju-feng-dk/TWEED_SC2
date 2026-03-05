# Using foundational libraries: NumPy, SciPy, Pandas, and Matplotlib

This note demonstrates basic usage of some of the foundational Python libraries in the context of wind energy applications. We'll work with synthetic wind speed data, fit a statistical distribution, model a simple wind turbine power curve, and estimate energy production.

Make sure you understand the code, and the usage of the these libraries. If needed, go to study the related documentaion and tutorials that dedicated to these libraries:
- [NumPy](https://numpy.org/doc/stable/)
- [SciPy](https://docs.scipy.org/doc/scipy/)
- [Pandas](https://pandas.pydata.org/docs/)
- [Matplotlib](https://matplotlib.org/stable/index.html)

## 1. Import Required Libraries

We'll import NumPy for numerical operations, Pandas for data handling, SciPy for statistical fitting, and Matplotlib for plotting.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy import stats
from scipy.optimize import curve_fit

# Set random seed for reproducibility
np.random.seed(48)
```

## 2. Generate Synthetic Wind Speed Data

We simulate one year of hourly wind speed data using a Weibull distribution, which is commonly used to model wind speed distributions. We'll create a time index and store the data in a Pandas DataFrame.

```python
# Number of hours in a year (365 days * 24 hours)
hours_in_year = 365 * 24

# Generate time index
time_index = pd.date_range(start='2024-01-01', periods=hours_in_year, freq='h')

# Weibull parameters: shape (k) and scale (A)
k = 2.0
A = 8.0  # m/s

# Generate wind speeds (m/s). Note that the generated wind speeds follows the 
# distribution, but the temporal order is likey unrealistic.
wind_speeds = stats.weibull_min.rvs(c=k, scale=A, size=hours_in_year)

# Create DataFrame
df = pd.DataFrame({'wind_speed': wind_speeds}, index=time_index)

# Display first few rows
print(df.head())
```

## 3. Basic Statistics and Visualization

Compute summary statistics and plot the time series and histogram.

```python
# Summary statistics
print(df.describe())

# Plot time series (first 500 hours for clarity)
df['wind_speed'][:500].plot(figsize=(12, 4), 
                            title='Hourly Wind Speed (first 500 hours)')
plt.ylabel('Wind Speed (m/s)')
plt.xlabel('Time')
plt.grid(True)
plt.show()
```

## 4. Fit a Weibull Distribution to the Data

We use SciPy's `weibull_min` fit method to estimate the parameters from the synthetic data.

```python
# Fit Weibull distribution
params = stats.weibull_min.fit(df['wind_speed'], floc=0)  # floc=0 fixes location to 0
k_fit, loc_fit, A_fit = params
print(f"Fitted parameters: shape k = {k_fit:.2f}, scale A = {A_fit:.2f} m/s")
```

## 5. Plot Histogram with Fitted Distribution

Overlay the fitted Weibull PDF on the histogram to visually assess the fit.

```python
# Create histogram
plt.figure(figsize=(8, 5))
count, bins, ignored = plt.hist(df['wind_speed'], bins=50, density=True, 
    alpha=0.6, color='skyblue', label='Data histogram')

# Plot fitted PDF
x = np.linspace(0, 25, 100)
pdf_fitted = stats.weibull_min.pdf(x, k_fit, loc=loc_fit, scale=A_fit)
plt.plot(x, pdf_fitted, 'r-', lw=2, 
    label=f'Weibull fit (k={k_fit:.2f}, A={A_fit:.2f})')

plt.xlabel('Wind Speed (m/s)')
plt.ylabel('Probability Density (-)')
plt.title('Wind Speed Distribution')
plt.legend()
plt.grid(True)
plt.show()
```

## 6. Wind Turbine Power Curve Model

We define a simple power curve using a piecewise function: cut-in speed 3 m/s, rated speed 12 m/s, cut-out speed 25 m/s, rated power 2 MW.

```python
def power_curve(wind_speed, cut_in=3, rated_speed=12, cut_out=25, 
                rated_power=2000):
    """
    Simple turbine power curve (power in kW).
    Returns power output for given wind speed(s).
    """
    power = np.zeros_like(wind_speed)
    # Region II: between cut-in and rated
    mask = (wind_speed >= cut_in) & (wind_speed < rated_speed)
    # Cubic increase from 0 to rated power
    power[mask] = rated_power * ((wind_speed[mask] - cut_in) / 
                                 (rated_speed - cut_in))**3
    # Region III: between rated and cut-out
    mask = (wind_speed >= rated_speed) & (wind_speed < cut_out)
    power[mask] = rated_power
    # Above cut-out: power = 0 (already zero)
    return power

# Compute power for each hour
df['power_kW'] = power_curve(df['wind_speed'])

# Plot power curve
wsp = np.linspace(0, 30, 200)
pwr = power_curve(wsp)
plt.figure(figsize=(8, 5))
plt.plot(wsp, pwr, 'b-', lw=2)
plt.xlabel('Wind Speed (m/s)')
plt.ylabel('Power (kW)')
plt.title('Turbine Power Curve')
plt.grid(True)
plt.xlim(0, 30)
plt.ylim(0, 2200)
plt.show()
```

## 7. Estimate Annual Energy Production (AEP)

Using the hourly power output, we can sum to get total energy (kWh) and convert to MWh.

```python
total_energy_kWh = df['power_kW'].sum()
total_energy_MWh = total_energy_kWh / 1000 # Note this only works for hourly data
print(f"Annual Energy Production: {total_energy_MWh:.2f} MWh")

# Also compute capacity factor
capacity_factor = total_energy_kWh / (hours_in_year * 2000)  # 2000 kW rated
print(f"Capacity Factor: {capacity_factor:.2%}")
```

## 8. Visualize Power Output Over Time

Plot power output for a sample period.

```python
df['power_kW'][:500].plot(figsize=(12, 4), 
                          title='Power Output (first 500 hours)')
plt.ylabel('Power (kW)')
plt.xlabel('Time')
plt.grid(True)
plt.show()
```

## 9. (Optional) Simple Wind Rose with Wind Direction

If we had wind direction data, we could create a wind rose. Here we generate synthetic direction data and use Matplotlib's polar plot.

```python
import matplotlib.cm as cm
import matplotlib.colors as colors

# Assume wind_direction and wind_speeds are already defined (e.g., from earlier data)
wind_direction = np.random.uniform(0, 360, size=hours_in_year)

# Create a 2D histogram (wind speed vs direction) and plot as polar
fig = plt.figure(figsize=(8, 8))
ax = fig.add_subplot(111, projection='polar')

# Bin direction and speed
nbins = 36
theta_bins = np.linspace(0, 2*np.pi, nbins+1)
speed_bins = np.linspace(0, 25, 10)

# Compute 2D histogram
H, theta_edges, speed_edges = np.histogram2d(
    np.radians(wind_direction), wind_speeds, 
    bins=[theta_bins, speed_bins])

# Set up colormap and normalization for the speed bins
cmap = cm.viridis
norm = colors.Normalize(vmin=0, vmax=25)

# Plot as bar sectors with colors mapped to speed
theta_centers = (theta_edges[:-1] + theta_edges[1:])/2
width = 2*np.pi / nbins
for j in range(len(speed_edges)-1):
    # Get color for this speed bin (using the bin's midpoint)
    speed_mid = (speed_edges[j] + speed_edges[j+1]) / 2
    color = cmap(norm(speed_mid))
    
    # Normalize by total counts to get frequency in percent
    radii = H[:, j] / H.sum() * 100
    # Stack bars: bottom is cumulative sum of previous bins
    bottom = np.sum(H[:, :j], axis=1) / H.sum() * 100
    ax.bar(theta_centers, radii, width=width, bottom=bottom,
           color=color, edgecolor='black', linewidth=0.5,
           label=f'{speed_edges[j]:.1f}-{speed_edges[j+1]:.1f} m/s')

ax.set_theta_zero_location('N')
ax.set_theta_direction(-1)
ax.set_title('Wind Rose (frequency %)')

# Place legend to the right of the plot
ax.legend(loc='center left', bbox_to_anchor=(1.2, 0.5))

plt.tight_layout()
plt.show()
```

## Conclusion

This tutorial covered:

- Generating synthetic wind data with NumPy and SciPy.
- Data handling and statistics with Pandas.
- Fitting a Weibull distribution using SciPy.
- Visualizing distributions and time series with Matplotlib.
- Modeling a turbine power curve and estimating annual energy production.
- Creating a simple wind rose.

These libraries form the foundation for more advanced wind energy analysis, such as resource assessment, turbine siting, and performance monitoring.