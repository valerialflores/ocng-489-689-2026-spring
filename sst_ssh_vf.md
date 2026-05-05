# Satellite-Based Analysis of Hurricane Intensification

**Author: Valeria Flores**
**OCNG 689**
**Role of SST and SSHA in the Gulf of Mexico**

I completed the code using my personal environment installed in my anaconda prompt since I started this data analysis before our lecture on Github. 

This notebook analyzes the relationship between sea surface temperature (SST), sea surface height anomaly (SSHA), and hurricane intensification in the Gulf of Mexico. Hurricane intensification is calculated from HURDAT2 as the 24-hour change in maximum sustained wind speed. Local mean SST and SSHA values are extracted around each storm position and compared to intensification.

## 1. Import libraries and load HURDAT2
```python
import pandas as pd
import numpy as np
import xarray as xr
import matplotlib.pyplot as plt
import statsmodels.api as sm
import cartopy.crs as ccrs
import cartopy.feature as cfeature
import glob
import os

os.makedirs("figures", exist_ok=True) #Folder for my figures

hurdat_file = "hurdat2.689.txt" #Load HURDAT2

with open(hurdat_file, "r") as f:
    lines = f.readlines()

#Converts HURDAT2 lat/lon strings to signed values
def parse_lat(lat_str):
    value = float(lat_str[:-1])
    return value if lat_str.endswith("N") else -value

def parse_lon(lon_str):
    value = float(lon_str[:-1])
    return -value if lon_str.endswith("W") else value
```

## 2. Parse HURDAT2 and create dataframe
```python
#Parse HURDAT2 lines
data = []
current_storm = None

for line in lines:
    parts = [p.strip() for p in line.split(",")]

    if parts[0].startswith("AL"): #Storm header line
        current_storm = parts[1] 

    else: #Storm data line
        if current_storm is not None:
            data.append([
                current_storm,
                parts[0], #Date
                parts[1], #Time
                parts[2], #RecordID
                parts[3], #Status
                parse_lat(parts[4]), #Lat
                parse_lon(parts[5]), #Lon
                int(parts[6]), #Wind Speed
                int(parts[7]) #Pressure
            ])

#Columns
columns = ["storm", "date", "time", "record_id", "status", "lat", "lon", "wind", "pressure"]

df = pd.DataFrame(data, columns=columns)

df["datetime"] = pd.to_datetime(df["date"] + df["time"], format="%Y%m%d%H%M")

#df.head()
```


## 3. Calculate 24-Hour Hurricane Intensification

This step calculates the dependent variable for the analysis. HURDAT2 is recorded every 6 hours, so 4 time steps represent 24 hours.

```python 
#Calculate 24-hour intensification
df = df.sort_values(["storm", "datetime"])

#6-hour data --> 4 steps = 24 hours
df["dV_24"] = df.groupby("storm")["wind"].diff(4)

#Remove NaNs
df = df.dropna(subset=["dV_24"]).reset_index(drop=True)

#Filter to Gulf domain
gulf_df = df[
    (df["lat"].between(18, 31)) &
    (df["lon"].between(-98, -80))
].copy()
```
## 4. Storm Tracks Map
The map uses full storm tracks for geographic context, while the statistical analysis uses the Gulf-filtered dataset.

```python
#Using full tracks for map context, analysis will only use gulf_df
storms = ["RITA", "HARVEY", "IDALIA"]
full_tracks = df[df["storm"].isin(storms)].copy()

#Map setup
fig = plt.figure(figsize=(9, 7))
ax = plt.axes(projection=ccrs.PlateCarree())

ax.set_extent([-100, -75, 15, 35], crs=ccrs.PlateCarree())
ax.set_facecolor("aliceblue")
ax.add_feature(cfeature.LAND, facecolor="lightgray")
ax.add_feature(cfeature.COASTLINE, linewidth=1)
ax.add_feature(cfeature.BORDERS, linewidth=0.5)

#Color scale for intensity
vmin = full_tracks["wind"].min()
vmax = full_tracks["wind"].max()

#Manual label positions
label_positions = {
    "HARVEY": (-96.5, 29.8),
    "RITA": (-91.5, 26.5),
    "IDALIA": (-84.0, 25.0)
}

#Plot tracks
for storm in storms:
    subset = full_tracks[full_tracks["storm"] == storm].sort_values("datetime")

    ax.plot(
        subset["lon"], subset["lat"],
        color="gray", linewidth=1, alpha=0.6,
        transform=ccrs.PlateCarree()
    )

    sc = ax.scatter(
        subset["lon"], subset["lat"],
        c=subset["wind"], cmap="plasma",
        vmin=vmin, vmax=vmax, s=28,
        transform=ccrs.PlateCarree(), zorder=3
    )

    ax.text(
        label_positions[storm][0],
        label_positions[storm][1],
        storm,
        fontsize=11,
        fontweight="bold",
        transform=ccrs.PlateCarree(),
        zorder=4
    )

#Title
ax.set_title(
    "Hurricane Tracks and Intensity Evolution\nin the Gulf of Mexico",
    fontsize=16
)

#Colorbar
cbar = plt.colorbar(sc, ax=ax, orientation="vertical", pad=0.02)
cbar.set_label("Maximum Sustained Wind Speed (kt)", fontsize=14)


gl = ax.gridlines(draw_labels=True, linewidth=0.5, linestyle="--", alpha=0.6)
gl.top_labels = False
gl.right_labels = False
gl.xlabel_style = {"size": 12}
gl.ylabel_style = {"size": 12}

#plt.savefig("figures/map.png", dpi=300, bbox_inches="tight")
#plt.show()
```

## 5. Objective 1: SST-intensification relationship analysis

```python
#Load OISST datasets for selected storms
sst1 = xr.open_dataset("sst.day.mean.2005.nc")
sst2 = xr.open_dataset("sst.day.mean.2017.nc")
sst3 = xr.open_dataset("sst.day.mean.2023.nc")

#Subset
sst1 = sst1.sel(lat=slice(18,31), lon=slice(262,280))
sst2 = sst2.sel(lat=slice(18,31), lon=slice(262,280))
sst3 = sst3.sel(lat=slice(18,31), lon=slice(262,280))

#Concat --> Joining data structures
sst = xr.concat([sst1, sst2, sst3], dim="time")

#Sort by time
sst = sst.sortby("time")
```

## 6. Create Start-time Variables for 24-Hour Intensification Windows
This next step ensures the ocean condition is sampled at the start of the 24-hour intensification period.
```python
#Sort data 
gulf_df = gulf_df.sort_values(["storm", "datetime"]).reset_index(drop=True)

#Start of 24-hour window
gulf_df["datetime_start"] = gulf_df.groupby("storm")["datetime"].shift(4)
gulf_df["lat_start"] = gulf_df.groupby("storm")["lat"].shift(4)
gulf_df["lon_start"] = gulf_df.groupby("storm")["lon"].shift(4)

#Clean data
gulf_df = gulf_df.dropna(
    subset=["dV_24", "datetime_start", "lat_start", "lon_start"]
).reset_index(drop=True)


#print(gulf_df[["storm", "datetime", "datetime_start", "wind", "dV_24"]].head())
```

## 7. Extract Local Mean SST Around Storm Positions
Instead of using one exact SST grid point or a Gulf-wide average, this step calculates the local SST around each hurricane position at the start of the 24-hour intensification window. A ±0.25° box is used around each storm location.
```python
#Converting start longitude to 0–360 for OISST
gulf_df["lon_start_360"] = gulf_df["lon_start"] % 360

#Subset SST to gulf region and time range --> faster to run
sst_gulf = sst.sel(
    time=slice(gulf_df["datetime_start"].min(), gulf_df["datetime_start"].max()),
    lat=slice(18, 31),
    lon=slice(262, 280)
)

#Calculate local SST box mean around storm point from HURDAT2
def get_box_mean_sst(row, box=0.25): #For each storm point, it grabs SST around the storm location in small +/- 0.25 box
    da = sst_gulf["sst"].sel(
        time=row["datetime_start"],
        method="nearest"
    )

    box_da = da.sel(
        lat=slice(row["lat_start"] - box, row["lat_start"] + box),
        lon=slice(row["lon_start_360"] - box, row["lon_start_360"] + box)
    )

    return float(box_da.mean().values)

#Apply local box mean
gulf_df["sst_start_boxmean"] = gulf_df.apply(get_box_mean_sst, axis=1)

#Saves the local average
gulf_df["sst_start_boxmean"] = pd.to_numeric(gulf_df["sst_start_boxmean"], errors="coerce")
gulf_df = gulf_df.dropna(subset=["sst_start_boxmean"]).reset_index(drop=True)

#print(gulf_df[["storm", "datetime_start", "lat_start", "lon_start", "sst_start_boxmean"]].head())
```
## 8. Plot and Statistical Analysis
**Question:** Does warmer water near the storm lead to stronger 24-hour hurricane intensification?

```python
#Plot
clean_sst = gulf_df[["sst_start_boxmean", "dV_24"]].copy() #Keep only SST and 24-hr intensification
clean_sst = clean_sst.replace([np.inf, -np.inf], np.nan).dropna() #Remove invalid values

x = clean_sst["sst_start_boxmean"].values
y = clean_sst["dV_24"].values

fig, ax = plt.subplots(figsize=(6,5))

#Plot storms
ax.scatter(x, y)

#Fit a linear regression line and calculate correlation
if len(x) > 1 and len(np.unique(x)) > 1:
    m, b = np.polyfit(x, y, 1) #m=slope, b=intercept of best-fit line
    r = np.corrcoef(x, y)[0,1] #pearson correlation coefficient (strength of relationship)
    
    #Sort x-values for smooth line plotting
    idx = np.argsort(x)
    ax.plot(x[idx], (m*x + b)[idx], color="red", linewidth=2) #Plot regression line 
    
    #Add equation and correlation value to plot
    ax.text(
        0.95, 0.05,
        f"y = {m:.2f}x {'+' if b >= 0 else '-'} {abs(b):.2f}\nr = {r:.2f}",
        transform=ax.transAxes,
        fontsize=12,
        horizontalalignment='right',
        verticalalignment='bottom',
        bbox=dict(facecolor='white', alpha=0.7)
    )

#Labels and formatting
ax.set_xlabel("Local Mean SST (°C)", fontsize=15)
ax.set_ylabel("24-hour Intensification (kt)", fontsize=15)
ax.set_title("SST vs Hurricane Intensification", fontsize=16)
ax.tick_params(axis='both', labelsize=14)
ax.grid()

#plt.savefig("figures/scatterSST.png", dpi=300, bbox_inches="tight")
#plt.show()


#print("Local SST box-mean correlation:", r)

# ---------------
#Regression table
clean_sst = gulf_df[["sst_start_boxmean", "dV_24"]].copy()
clean_sst = clean_sst.replace([np.inf, -np.inf], np.nan).dropna()

#Regression set up
X_sst = sm.add_constant(clean_sst["sst_start_boxmean"])
y_sst = clean_sst["dV_24"]

model_sst = sm.OLS(y_sst, X_sst).fit()

#Extract stats
r_squared = model_sst.rsquared
p_value = model_sst.pvalues[1]
r = np.sqrt(r_squared) * np.sign(model_sst.params[1]) #Correlation

#Quick stats check
print("SST RESULTS")
print(f"r = {r:.3f}")
print(f"R^2 = {r_squared:.3f}")
print(f"p-value = {p_value:.5f}") 

print(model_sst.summary()) #Full regression table
```
## 9. Objective 2: SSHA-intensification relationship analysis
SSHA is derived from SLA in the Copernicus dataset. A ±0.25° box mean is calculated around each storm point at the start of the 24-hour intensification period.

```python
#Load SLA data and extract local mean SLA
ssh = xr.open_dataset("ssh_gulf.nc")

#Sort time coordinate
if "time" in ssh.coords:
    ssh = ssh.sortby("time")

#Subset to the time range needed
ssh_gulf = ssh.sel(
    time=slice(gulf_df["datetime_start"].min(), gulf_df["datetime_start"].max())
)

#Calculate local SLA box mean around each storm point
def get_box_mean_ssh(row, box=0.25):
    #Select nearest daily SLA field to the storm start time
    da = ssh_gulf["sla"].sel(
        time=row["datetime_start"],
        method="nearest"
    )

    #Select small box around storm location
    box_da = da.sel(
        latitude=slice(row["lat_start"] - box, row["lat_start"] + box),
        longitude=slice(row["lon_start"] - box, row["lon_start"] + box)
    )

    #Return average SLA value in that box
    return float(box_da.mean().values)

#Apply local box mean to each storm point
gulf_df["ssh_start_boxmean"] = gulf_df.apply(get_box_mean_ssh, axis=1)

#Clean invalid values
gulf_df["ssh_start_boxmean"] = pd.to_numeric(
    gulf_df["ssh_start_boxmean"],
    errors="coerce"
)

gulf_df = gulf_df.dropna(subset=["ssh_start_boxmean"]).reset_index(drop=True)

#print(gulf_df[["storm", "datetime_start", "lat_start", "lon_start", "ssh_start_boxmean"]].head())
```

## 10. Plot and Statistical Analysis
**Question:** Does local sea surface height anomaly, as a proxy for upper ocean heat content, influence 24-hour hurricane intensification?

```python
#Plot
#Clean dataset for plotting
clean_ssh = gulf_df[["ssh_start_boxmean", "dV_24"]].copy()
clean_ssh = clean_ssh.replace([np.inf, -np.inf], np.nan).dropna()

x = clean_ssh["ssh_start_boxmean"].values
y = clean_ssh["dV_24"].values

fig, ax = plt.subplots(figsize=(6,5))

ax.scatter(x, y)

#Fit trend line and calculate correlation, same process used for sst vs intensity plot
if len(x) > 1 and len(np.unique(x)) > 1:
    m, b = np.polyfit(x, y, 1)
    r = np.corrcoef(x, y)[0,1]

    idx = np.argsort(x)
    ax.plot(x[idx], (m*x + b)[idx], color="red", linewidth=2)

    ax.text(
        0.95, 0.05,
        f"y = {m:.2f}x {'+' if b >= 0 else '-'} {abs(b):.2f}\nr = {r:.2f}",
        transform=ax.transAxes,
        fontsize=12,
        horizontalalignment='right',
        verticalalignment='bottom',
        bbox=dict(facecolor='white', alpha=0.7)
    )
#Labels and formatting
ax.set_xlabel("Local Mean SLA (m)", fontsize=15)
ax.set_ylabel("24-hour Intensification (kt)", fontsize=15)
ax.set_title("SSHA vs Hurricane Intensification", fontsize=16)
ax.tick_params(axis='both', labelsize=14)
ax.grid()

#plt.savefig("figures/scatterSSHA.png", dpi=300, bbox_inches="tight")
#plt.show()

#print("Local SSHA box-mean correlation:", r)

#Regression Table
#Clean data
clean_ssh = gulf_df[["ssh_start_boxmean", "dV_24"]].copy()
clean_ssh = clean_ssh.replace([np.inf, -np.inf], np.nan).dropna()

#Regression setup
X_ssh = sm.add_constant(clean_ssh["ssh_start_boxmean"])
y_ssh = clean_ssh["dV_24"]

#Fit model
model_ssh = sm.OLS(y_ssh, X_ssh).fit()

#Extract stats
r_squared = model_ssh.rsquared
p_value = model_ssh.pvalues[1]     
r = np.sqrt(r_squared) * np.sign(coefficient)

#Quick stats check
print("SSHA RESULTS")
print(f"r = {r:.3f}")
print(f"R^2 = {r_squared:.3f}")
print(f"p-value = {p_value:.5f}")

#Full regression table
print(model_ssh.summary())
```

## 11. Objective 3: Evaluating the influence of SST + SSHA on hurricane intensification
**Question:** Does combining local SST and SSHA improve the ability to explain 24-hour hurricane intensification compared to using SST alone?

```python
#Plot
#Clean dataset for combined analysis
clean_combined = gulf_df[["sst_start_boxmean", "ssh_start_boxmean", "dV_24"]].copy()
clean_combined = clean_combined.replace([np.inf, -np.inf], np.nan).dropna()

fig, ax = plt.subplots(figsize=(8, 7))

sc = ax.scatter(
    clean_combined["sst_start_boxmean"],
    clean_combined["ssh_start_boxmean"],
    c=clean_combined["dV_24"],
    cmap="plasma",
    s=120
)

ax.set_xlabel("Local Mean SST (°C)", fontsize=15)
ax.set_ylabel("Local Mean SLA (m)", fontsize=15)
ax.set_title("Combined SST and SSHA Relationship with Hurricane Intensification", fontsize=16)
ax.tick_params(axis="both", labelsize=14)

cbar = plt.colorbar(sc, ax=ax)
cbar.set_label("24-hour Intensification (kt)", fontsize=15)
cbar.ax.tick_params(labelsize=12)

plt.grid(True)
#plt.savefig("figures/scattercombined.png", dpi=300, bbox_inches="tight")
#plt.show()

#Regression table
X = clean_combined[["sst_start_boxmean", "ssh_start_boxmean"]]
X = sm.add_constant(X)
y = clean_combined["dV_24"]

model = sm.OLS(y, X).fit()

#Extract stats
r_squared = model.rsquared
p_values = model.pvalues


print("COMBINED MODEL RESULTS")
print(f"R² = {r_squared:.3f}")
print(f"SST p-value = {p_values[1]:.5f}")
print(f"SSHA p-value = {p_values[2]:.5f}")
print(model.summary())
```




