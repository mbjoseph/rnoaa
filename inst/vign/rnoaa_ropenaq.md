<!--
%\VignetteEngine{knitr::knitr}
%\VignetteIndexEntry{air_quality_and_weather_data}
%\VignetteEncoding{UTF-8}
-->



# Complementing air quality data with weather data using rnoaa

__Author__ Maëlle Salmon

# Introduction: getting air quality data

This vignette aims at explaining how you can complement a data.frame with weather data using rnoaa. In this vignette we shall use air quality data from the OpenAQ platform queried with the ropenaq package, for India. Using [ropenaq](https://github.com/ropensci/ropenaq) one can get e.g. PM2.5 values over time in Indianapolis over the last days. For getting all data for march we'll loop over several pages.

First, we need to know how many measures are available for Indianapolis for March 2016.


```r
library("ropenaq")

measurementsIndianapolis <- aq_measurements(city = "Indianapolis", parameter = "pm25",
                                     date_from = as.character(Sys.Date() - 30),
                                     date_to = as.character(Sys.Date()),
                                     limit = 10000 )
save(measurementsIndianapolis, file = "data/measurementsIndianapolis.RData")
```

We filter negative values.


```r
load("data/measurementsIndianapolis.RData")
library("dplyr")
measurementsIndianapolis %>% head() %>% knitr::kable()
```



|location        |day        |     value| longitude| latitude|
|:---------------|:----------|---------:|---------:|--------:|
|Indpls - I-70 E |2017-10-20 | 18.500000| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-21 | 12.414286| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-22 |  8.573333| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-23 |  7.633333| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-24 |  4.528571| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-25 |  6.455556| -86.13088| 39.78793|

```r
measurementsIndianapolis <- filter(measurementsIndianapolis, value > 0)
```

We now transform these data into daily data.


```r
# only keep stations with geographical information
measurementsIndianapolis <- filter(measurementsIndianapolis, !is.na(latitude))
# now transform to daily data
measurementsIndianapolis <- measurementsIndianapolis %>%
  #mutate(day = as.Date(dateLocal)) %>%
  group_by(location, day) %>%
  summarize(value = mean(value),
            longitude = longitude[1],
            latitude = latitude[1]) %>%
  ungroup()
measurementsIndianapolis %>% head() %>% knitr::kable()
```



|location        |day        |     value| longitude| latitude|
|:---------------|:----------|---------:|---------:|--------:|
|Indpls - I-70 E |2017-10-20 | 18.500000| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-21 | 12.414286| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-22 |  8.573333| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-23 |  7.633333| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-24 |  4.528571| -86.13088| 39.78793|
|Indpls - I-70 E |2017-10-25 |  6.455556| -86.13088| 39.78793|

Air quality and weather are correlated, so one could be interested in getting a time series of say temperature for the same location. The OpenAQ platform itself does not provide weather data but nearly all stations have geographical coordinates. Our goal here will be to use rnoaa to complement this table with precipitation and temperature.

# Find weather stations

For finding the right station(s) we shall use the `meteo_nearby_stations` function. It returns a list with the weather stations nearby each latitude/longitude given as arguments, respecting the other arguments such as maximal radius, first year with data, etc. For finding stations one might have to play a bit with the parameters until there is at least one station for each location.

Here we query stations with a less than 15km distance from the air quality stations, with precipitation and temperature data, and with data starting from 2016. Note that the query takes a while.


```r
library("rnoaa")
station_data <- ghcnd_stations()
lat_lon_df <- select(measurementsIndianapolis,
                     location,
                     latitude,
                     longitude) %>% unique() %>%
  ungroup() %>%
  rename(id = location) %>%
  mutate(id = factor(id))

stationsIndianapolis <- meteo_nearby_stations(lat_lon_df = as.data.frame(lat_lon_df),
                                       station_data = station_data,
                                       radius = 5,
                                       year_min = as.character(format(Sys.Date(), "%Y")),
                                       var = c("TAVG", "PRCP"))
stationsIndianapolis <- unique(bind_rows(stationsIndianapolis) %>% select(- distance))

save(stationsIndianapolis, file = "data/stationsIndianapolis.RData")
```


```r
load("data/stationsIndianapolis.RData")
stationsIndianapolis %>% knitr::kable()
```



|id          |name                 | latitude| longitude|
|:-----------|:--------------------|--------:|---------:|
|US1INMR0134 |INDIANAPOLIS 1.3 SSE |  39.7574|  -86.1404|
|US1INMR0122 |INDIANAPOLIS 3.4 WSW |  39.7565|  -86.2044|

Now let us plot the AQ and weather stations on a quick and dirty map with no legend, red for AQ stations, blue for weather stations.

> Not shown


```r
library("ggmap")
map <- get_map(location = "Indianapolis", zoom = 11)
ggmap(map) +
  geom_point(aes(x = longitude, y = latitude),
             data = stationsIndianapolis, col = "blue", size = 4)+
  geom_point(aes(x = longitude, y = latitude),
             data = measurementsIndianapolis, col = "red", size = 4)
```


# Query weather data for these stations

For pulling weather data from these weather monitors, we shall use the `meteo_pull_monitors` function.


```r
library("rnoaa")
monitors <- stationsIndianapolis$id
all_monitors_clean <- meteo_pull_monitors(monitors,
                                      date_min = as.character(Sys.Date() - 30),
                                     date_max = as.character(Sys.Date())) %>%
  rename(day = date,
         location = id)
```

```
#> file path:          /Users/sckott/Library/Caches/rnoaa/ghcnd/US1INMR0134.dly
```

```
#> file last updated:  2019-09-23 14:44:37
```

```
#> file min/max dates: 2015-05-01 / 2019-09-30
```

```
#> file path:          /Users/sckott/Library/Caches/rnoaa/ghcnd/US1INMR0122.dly
```

```
#> file last updated:  2019-09-23 14:44:38
```

```
#> file min/max dates: 2012-11-01 / 2019-09-30
```

```r
all_monitors_clean %>% head() %>% knitr::kable()
```



|location    |day        | dapr| mdpr| prcp| snow|
|:-----------|:----------|----:|----:|----:|----:|
|US1INMR0134 |2019-08-24 |   NA|   NA|   NA|   NA|
|US1INMR0134 |2019-08-25 |   NA|   NA|   NA|   NA|
|US1INMR0134 |2019-08-26 |   NA|   NA|  165|   NA|
|US1INMR0134 |2019-08-27 |   NA|   NA|  152|   NA|
|US1INMR0134 |2019-08-28 |   NA|   NA|   18|   NA|
|US1INMR0134 |2019-08-29 |   NA|   NA|    0|   NA|

Here we notice some values are not available. Therefore, we might need to go back to weather stations searching with, for instance, a larger radius. In this case let's say we're ok with the result of the search.

# Join the two tables, thus complementing the original table

Therefore, in this case we will bind the rows of the air quality table with the weather table.


```r
measurementsIndianapolis <- bind_rows(measurementsIndianapolis, all_monitors_clean)
measurementsIndianapolis %>% head() %>% knitr::kable()
```



|location        |day        |     value| longitude| latitude| dapr| mdpr| prcp| snow|
|:---------------|:----------|---------:|---------:|--------:|----:|----:|----:|----:|
|Indpls - I-70 E |2017-10-20 | 18.500000| -86.13088| 39.78793|   NA|   NA|   NA|   NA|
|Indpls - I-70 E |2017-10-21 | 12.414286| -86.13088| 39.78793|   NA|   NA|   NA|   NA|
|Indpls - I-70 E |2017-10-22 |  8.573333| -86.13088| 39.78793|   NA|   NA|   NA|   NA|
|Indpls - I-70 E |2017-10-23 |  7.633333| -86.13088| 39.78793|   NA|   NA|   NA|   NA|
|Indpls - I-70 E |2017-10-24 |  4.528571| -86.13088| 39.78793|   NA|   NA|   NA|   NA|
|Indpls - I-70 E |2017-10-25 |  6.455556| -86.13088| 39.78793|   NA|   NA|   NA|   NA|


 Now some locations are air quality locations and have only missing values in the weather columns, and some locations are weather locations and have only missing values in the air quality columns.

We can plot the data we got.


```r
data_plot <- measurementsIndianapolis %>%
  rename(pm25 = value) %>%
  select(- longitude, - latitude)

data_plot <- tidyr::gather_(data_plot, "parameter", "value",           names(data_plot)[3:ncol(data_plot)])

library("ggplot2")
ggplot(data_plot) +
  geom_line(aes(x = day, y = value, col = location)) +
  facet_grid(parameter ~ ., scales = "free_y")
```

![plot of chunk unnamed-chunk-10](figure/unnamed-chunk-10-1.png)
