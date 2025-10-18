# CLIGEN County Climate Database

## Overview

This repository contains a comprehensive county-level climate database for the contiguous United States, designed to provide climate inputs for the Water Erosion Prediction Project (WEPP) model. The database consists of 3,227 CLIGEN-formatted climate files, one for each U.S. county, generated from historical meteorological observations spanning 1980-2017.

CLIGEN (CLImate GENerator) is a stochastic weather generator that produces daily climate time series data required by WEPP and other erosion prediction models. This database provides observed historical climate data in CLIGEN format, enabling accurate soil erosion modeling at the county level across the United States.

## Dataset Description

### Files and Structure

The repository contains the following key components:

- **`observed_climates/`** - Directory containing 3,227 `.cli` files (CLIGEN format) with historical climate data
  - Total size: ~4.3 GB
  - One file per U.S. county
  - Coverage: 1980-2017 (38 years)
  - Each file contains daily precipitation, temperature, and radiation data

- **`observed_climate_lookup.csv`** - Lookup table linking counties to their climate files
  - Columns: `AFFGEOID`, `par`, `cli`, `lng`, `lat`
  - 3,234 records (includes 7 failed counties)

- **`par_by_county.csv`** - County metadata and CLIGEN parameter file assignments
  - Columns: `STATEFP`, `COUNTYFP`, `COUNTYNS`, `AFFGEOID`, `GEOID`, `NAME`, `LSAD`, `ALAND`, `AWATER`, `par`, `c_lng`, `c_lat`
  - 3,227 successfully processed counties

- **`failed_counties.txt`** - List of 7 counties that could not be processed (primarily Alaska and U.S. territories)

### Coverage

- **Geographic Coverage**: All U.S. counties in the contiguous United States plus Alaska and territories (where data available)
- **Temporal Coverage**: January 1, 1980 to December 31, 2017 (38 years)
- **Variables**: 
  - Daily precipitation (mm)
  - Daily maximum temperature (°C)
  - Daily minimum temperature (°C)
  - Daily solar radiation (Langleys/day)
  - Wind velocity and direction
  - Dew point temperature

### Failed Counties

The following 7 counties could not be processed due to data availability issues:
- 0500000US02016 (Alaska)
- 0500000US60010 (American Samoa)
- 0500000US60020 (American Samoa)
- 0500000US60030 (American Samoa)
- 0500000US60040 (American Samoa)
- 0500000US60050 (American Samoa)

## Construction Methodology

The dataset was constructed through a two-step process:

### Step 1: Identify CLIGEN Parameter Files (`_0_identify_pars.py`)

This script identifies the most appropriate CLIGEN parameter file for each county based on geographic proximity:

1. **Load County Boundaries**: Uses the U.S. Census Bureau's 500k county shapefile (cb_2017_us_county_500k)

2. **Determine County Centroid**: For each county:
   - Fetches SSURGO soil map data for the county extent
   - Builds a mask for the county polygon
   - Calculates the geographic centroid (longitude, latitude)

3. **Find Nearest CLIGEN Station**: 
   - Uses the CLIGEN 2015 station database
   - Identifies the closest weather station to the county centroid
   - Records the station's parameter file (`.par` file)

4. **Output**: Generates `par_by_county.csv` with county metadata and assigned CLIGEN parameter files

### Step 2: Build Climate Files (`_1_build_climates.py`)

This script generates observed climate files for each county:

1. **Retrieve Historical Data**: For each county centroid:
   - Fetches daily meteorological observations from Daymet v3
   - Variables: precipitation (prcp), minimum temperature (tmin), maximum temperature (tmax)
   - Time period: 1980-2017
   - Data source: Daymet NetCDF files (`/geodata/daymet/`)

2. **Convert to CLIGEN Input Format**:
   - Transforms Daymet data to CLIGEN PRN format
   - Creates `.prn` files with daily observations

3. **Run CLIGEN**: 
   - Executes CLIGEN using the assigned parameter file and observed data
   - Generates `.cli` files in CLIGEN format
   - CLIGEN fills gaps and ensures data consistency

4. **Output**: 
   - `.cli` files in `observed_climates/` directory
   - `.prn` files with raw observations
   - `.log` files documenting the build process
   - `observed_climate_lookup.csv` linking counties to climate files

### Data Sources

- **County Boundaries**: U.S. Census Bureau Cartographic Boundary Shapefiles (cb_2017_us_county_500k)
- **CLIGEN Stations**: CLIGEN 2015 station database
- **Meteorological Data**: Daymet Version 3 daily surface weather data
  - Daymet provides gridded estimates of daily weather parameters
  - 1 km spatial resolution
  - Covers North America from 1980 onwards
- **Soil Data**: SSURGO (Soil Survey Geographic Database), March 2017

## CLIGEN File Format

Each `.cli` file follows the CLIGEN Version 5.3 format with the following structure:

### Header Information
- Station metadata (name, location, elevation)
- Observation period and years simulated
- Monthly climatological summaries:
  - Average maximum temperature
  - Average minimum temperature
  - Average solar radiation
  - Average precipitation

### Daily Records
Each line represents one day with columns:
- Day, Month, Year
- Precipitation (mm)
- Duration (hours)
- Time to peak intensity
- Intensity pattern
- Maximum temperature (°C)
- Minimum temperature (°C)
- Solar radiation (Langleys/day)
- Wind velocity (m/s)
- Wind direction (degrees)
- Dew point temperature (°C)

## Usage

### Accessing Climate Data by County

Use the lookup table to find the climate file for a specific county:

```python
import pandas as pd

# Load the lookup table
lookup = pd.read_csv('observed_climate_lookup.csv')

# Find climate file for a specific county (example: AFFGEOID)
county_id = '0500000US01005'  # Barbour County, Alabama
climate_info = lookup[lookup['AFFGEOID'] == county_id]

cli_filename = climate_info['cli'].values[0]
print(f"Climate file: observed_climates/{cli_filename}")
print(f"Location: ({climate_info['lng'].values[0]}, {climate_info['lat'].values[0]})")
```

### Reading CLIGEN Files

CLIGEN files are fixed-width text files. The daily data starts after the header:

```python
import pandas as pd

def read_cligen_daily(filepath):
    """Read daily data from CLIGEN .cli file"""
    # Skip header lines (typically first ~20 lines)
    # Parse fixed-width format
    colspecs = [(0, 3), (3, 6), (6, 12), (12, 18), (18, 24), 
                (24, 30), (30, 36), (36, 42), (42, 48), 
                (48, 54), (54, 60), (60, 66)]
    names = ['day', 'month', 'year', 'prcp', 'dur', 'tp', 'ip',
             'tmax', 'tmin', 'rad', 'wvl', 'wdir', 'tdew']
    
    df = pd.read_fwf(filepath, colspecs=colspecs, names=names, 
                     skiprows=14, header=None)
    return df

# Example usage
df = read_cligen_daily('observed_climates/0500000US01005.cli')
print(df.head())
```

### Integration with WEPP

These climate files can be used directly with WEPP models:

1. Select the appropriate `.cli` file based on county location
2. Reference the file in your WEPP run file
3. WEPP will read the daily climate data for erosion calculations

### County Metadata

Access detailed county information from the par_by_county.csv:

```python
import pandas as pd

# Load county data
counties = pd.read_csv('par_by_county.csv', header=None,
                       names=['STATEFP', 'COUNTYFP', 'COUNTYNS', 'AFFGEOID', 
                              'GEOID', 'NAME', 'LSAD', 'ALAND', 'AWATER', 
                              'par', 'c_lng', 'c_lat'])

# Find counties in a specific state (e.g., Alabama = 01)
alabama_counties = counties[counties['STATEFP'] == '01']
print(alabama_counties[['NAME', 'par', 'c_lng', 'c_lat']])
```

## Dependencies

The dataset was constructed using the following software and libraries:

### Python Packages
- **wepppy**: WEPP Python interface and climate utilities
  - `wepppy.climates.cligen`: CLIGEN station management and execution
  - `wepppy.climates.daymet_singlelocation_client`: Daymet data retrieval
  - `wepppy.all_your_base.geo`: Geospatial utilities
- **numpy**: Numerical operations
- **pandas**: Data manipulation (for usage examples)
- **matplotlib**: Visualization (used during development)

### External Tools
- **CLIGEN Version 5.3**: Stochastic weather generator
- **GDAL/OGR**: Geospatial data processing

### Data Requirements (for reconstruction)
- U.S. Census Bureau county shapefiles
- CLIGEN 2015 station parameter files
- Daymet v3 NetCDF climate data (1980-2017)
- SSURGO soil database (March 2017)

## Reproducing the Dataset

To regenerate this database from scratch:

1. **Prerequisites**:
   - Install wepppy and dependencies
   - Download Daymet v3 data for 1980-2017
   - Obtain U.S. county shapefiles
   - Access to SSURGO database

2. **Step 1 - Identify Parameters**:
   ```bash
   python _0_identify_pars.py
   ```
   This generates `par_by_county.csv` and `failed_counties.txt`

3. **Step 2 - Build Climates**:
   ```bash
   python _1_build_climates.py
   ```
   This generates climate files in `observed_climates/` and `observed_climate_lookup.csv`

   For parallel processing across multiple workers:
   ```bash
   python _1_build_climates.py 0 &
   python _1_build_climates.py 1 &
   python _1_build_climates.py 2 &
   python _1_build_climates.py 3 &
   ```

## Citations and References

### CLIGEN
- Nicks, A.D., L.J. Lane, and G.A. Gander. 1995. "Weather Generator." Chapter 2 in USDA Water Erosion Prediction Project (WEPP) Technical Documentation, NSERL Report No. 10. USDA-ARS National Soil Erosion Research Laboratory, West Lafayette, IN.

### Daymet
- Thornton, M.M., R. Shrestha, Y. Wei, P.E. Thornton, S. Kao, and B.E. Wilson. 2020. Daymet: Daily Surface Weather Data on a 1-km Grid for North America, Version 4. ORNL DAAC, Oak Ridge, Tennessee, USA. https://doi.org/10.3334/ORNLDAAC/1840

### WEPP
- Flanagan, D.C., and M.A. Nearing (Eds.). 1995. USDA-Water Erosion Prediction Project: Hillslope Profile and Watershed Model Documentation. NSERL Report No. 10. USDA-ARS National Soil Erosion Research Laboratory, West Lafayette, IN.

### SSURGO
- Soil Survey Staff. Natural Resources Conservation Service, United States Department of Agriculture. Soil Survey Geographic (SSURGO) Database. Available online at https://sdmdataaccess.sc.egov.usda.gov.

## License

This dataset is provided for research and educational purposes. Please cite appropriately when using this data in publications.

## Contact and Support

For questions or issues regarding this dataset, please open an issue in the GitHub repository.

## Version History

- **Version 1.0** (Initial Release): 3,227 counties, 1980-2017 climate data, CLIGEN v5.3 format
