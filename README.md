# MELODIST - An open-source MEteoroLOgical observation time series DISaggregation Tool

*Welcome to MELODIST - MEteoroLOgical observation time series DISaggregation Tool*

MELODIST is an open-source toolbox written in Python for disaggregating daily meteorological time series to hourly time steps. It is licensed under GPLv3 (see license file). The software framework consists of disaggregation functions for each variable including temperature, humidity, precipitation, shortwave radiation, and wind speed. These functions can simply be called from a station object, which includes all relevant information about site characteristics. The data management of time series is handled using data frame objects as defined in the pandas package. In this way, input and output data can be easily prepared and processed. For instance, the pandas package is data i/o capable and includes functions to plot time series using the matplotlib library.

An example file (example.py) is provided along the package itself. This example demonstrates the usage of MELODIST for all variables. First, a station object is created providing some basic details on the site’s characteristics (e.g., latitude and longitude are relevant for radiation disaggregation). Once the station object is defined and filled with data, each disaggregation step is done through calling the designated function specified for each variable. Each of these functions requires a `method` argument and if needed additional parameters to work properly. Some of these methods (see below) require additional statistical evaluations of hourly time series prior to the disaggregation procedure. This information is stored in a station statistics object that is associated to the station object (see example file for further details).

## Station object:
In the framework of MELODIST a station object includes all relevant information including metadata and time series. A station is generated using the constructor method:

    s = melodist.Station(lon=longitude, lat=latitude, timezone=timezone)

Data is simply added by assignment (e.g., the data frame `data_obs_daily`):

    s.data_daily = data_obs_daily

A station statistics object can be generated in a similar manner. As station statistics are derived through analysing hourly observations for calibration, a reference to the data frame including hourly observations is given:

    s.statistics = melodist.StationStatistics(data_obs_hourly)

Statistics can be derived for each variable by calling the respective function of the statistics object `s.statistics`: `calc_wind_stats()`, `calc_humidity_stats()`, `calc_temperature_stats()`, `calc_precipitation_stats()`, and `calc_radiation_stats`.

## Naming convention for dataframe columns

MELODIST expects exact naming conventions for the time series provided in pandas dataframes. Please find the specification of column names below:

* `temp`: Temperature [K]
* `precip`: Precipitation [mm/time step]
* `glob`: Global (shortwave) radiation [W/m**2]
* `hum`: Relative humidity [%]
* `wind`: Wind speed [m/s]
* `ssd`: sunshine duration [min]

For daily data, additional columns need to be specified (if applicable):

* `temp`: Average temperature [K]
* `tmin`: Minimum temperature [K]
* `tmax`: Maximum temperature [K]
* `hum_min`: Minimum humidity [%]
* `hum_max`: Maximum humidity [%]
* `ssd`: sunshine duration [h]

Please note that the dataframe's index must contain datetime values.

## Disaggregation methods:

The `Station` class provides functions to perform the disaggregation procedures for each variable: `disaggregate_temperature()`, `disaggregate_humidity()`, `disaggregate_wind()`, `disaggregate_radiation()`, and `disaggregate_precipitation()`. Moreover, an interpolation approach is also available using the `interpolate()` function.

Hint: It is worth noting that each of the implemented disaggregation methods is directly accessible, e.g., `melodist.precipitation.disagg_prec()`. In this case all relevant parameters (e.g., those derived through calibrations) need to provided in the function call. This method specific call of functions is not necessary if a station and the corresponding station statistic object is defined. Thus, it is recommended to define objects and to perform the disaggregation procedures using the object’s methods. Also, the names and signatures of these functions are likely subject to change in future versions of MELODIST.

At least the `method` argument is required for each of the disaggregation functions in a `Station`object. Please find below a list of available disaggregation methods for each variable:

### Temperature (based on minimum and maximum temperature recordings):
* `method='sine'` (T1) standard sine redistribution, requires 1 additional parameter `min_max_time='fix'` (T1a): The diurnal course of temperature is fixed without any seasonal variations.
* `min_max_time='sun_loc'` (T1b): The diurnal course of temperature is modelled based on sunrise, noon and sunset calculations.
* `min_max_time='sun_loc_shift'` (T1c): This option activates empirical corrections of the ideal course modelled by `sun_loc` (requires calling `calc_temperature_stats()` prior to the disaggregation).
* An optional parameter `mod_nighttime` (T1d, bool, default: `False`) allows one to apply a linear interpolation of night time values, which proves preferable during polar nights.

### Humidity:
* `method='minimal'` (H1): The dew point temperature is set to the minimum temperature on that day.
* `method='dewpoint_regression'` (H2): Based on hourly observations, a regression approach is applied to calculate daily dew point temperature. Regression parameters must be specified (which is automatically done if `calc_humidity_stats()` is called prior to disaggregation).
* `method='linear_dewpoint_variation'` (H3): This method extends H2 through linearly varying dew point temperature between consecutive days. The parameter `kr` needs to be specified (kr=6 if monthly radiation exceeds 100 W/m^2 else kr=12).
* `method='min_max'` (H4): this method requires minimum and maximum relative humidity for each day.

### Wind speed:
* `method = 'equal'` (W1): If this method is chosen, the daily average wind speed is assumed to be valid for each hour on that day.
* `method = 'cosine'` (W2): The cosine function option simulates a diurnal course of wind speed and requires calibration (`calc_wind_stats()`).
* `method = 'random'` (W3): This option is a stochastic method that draws random numbers to disaggregate wind speed taking into account the daily average (no parameter estimation required).

### Radiation:
* `method='pot_rad'` (R1): This method allows one to disaggregate daily averages of shortwave radiation using hourly values of potential (clear-sky) radiation calculated for the location of the station.
* `method='pot_rad_via_ssd'` (R2): If daily sunshine recordings are available, the Angstrom model is applied to transform sunshine duration to shortwave radiation.
* `method='pot_rad_via_bc'` (R3): In this case, the Bristow-Campbell model is applied which relates minimum and maximum temperature to shortwave radiation. 

### Precipitation
* `method='equal'` (P1): In order to derive hourly from daily values, the daily total is simply divided by 24 resulting in an equal distribution.
* `method='cascade'` (P2): The cascade model is more complex and requires a parameter estimation method (`calc_precipitation_stats()`). Statistics can be calculated using different options (parameters). Using the keyword `months`, the seasons for which the statistics will be calculated independently can be specified (see example file). The keyword `percentile` allows one to adjust the threshold to separate precipitation intensities into two classes (low and high) for building the parameters. The default value is 50% (median). An additional optional argument `avg_stats` is used to decide whether statistics of all cascade levels will be averaged (default is `True`). All options previously listed are optional and can be changed to tune the disaggregation results.
* `method='masterstation'` (P3). If hourly values are available for another site in the vicinity of the station considered, the cumulative sub-daily mass curve can be transferred from the station that provides hourly values to the station of interest.

## Utilities:
Among other features the `melodist.util` module includes some functions that might be useful for data analyses:

* `detect_gaps(dataframe, timestep, print_all=False, print_max=5, verbose=True)` can be used to find gaps in the data frame. A gap will be detected if any increment of time is not equal to the specified time step (in seconds).

* Some methods require time series with full days (24 h) only. `drop_incomplete_days(dataframe)` drops heading and trailing days if they are not complete (0-23h).

* For testing purposes an aggregation function is provided which aggregates hourly time series (data frames) to daily time series taking into account the characteristics of each meteorological variable (e.g., mean value for precipitation, daily total for precipitation, ...): `daily_from_hourly()`

## Data input/output:

MELODIST includes a feature to read and save parameter settings for all disaggregation methods in order to transfer settings or to continue work at a later time. This feature is based on the JSON format which provides a flexible and easily readable ASCII file format for different applications.

To save MELODIST parameters included in station statistics object you can simply call the `to_json(filename)` method of this object. At any time it is possible to recall this settings by creating a new station statistics object based on the settings stored in that file:

    new_stationstatistics_object = melodist.StationStatistics.from_json(filename)

Since MELODIST is based on pandas, numerous ways to import and export pandas data frames are possible. The `to_csv()` function of pandas is ideal to save and load time series without any restriction with respect to MELODIST applications. 

## Literature
Förster, K., Hanzer, F., Winter, B., Marke, T., and Strasser, U.: An open-source MEteoroLOgical observation time series DISaggregation Tool (MELODIST v0.1.0), Geosci. Model Dev. Discuss., doi:10.5194/gmd-2016-51, in review, 2016. 

