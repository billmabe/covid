Cleaning the NY Times COVID-19 Case Count Data
================
Bill Mabe
April 10, 2020

``` r
knitr::opts_chunk$set(echo = TRUE)
```

# PURPOSE

The purpose of this script is to produce two working data frames: 1. US
state 2. US city / county

# PROCESS

1)  read in data
2)  find and remove duplicate observations
3)  test each of the files for whether their values for Confirmed,
    Deaths, and Recovered are cumulative totls or daily totals.
4)  correct revised cumulative case totals
5)  create unbroken time series
6)  fill time series “up” so that all rows have the correct values
7)  create the daily count variables

## A. READ COVID-19 CASE AND DEATH DATA

``` r
require(readr)
```

    ## Loading required package: readr

``` r
require(dplyr)
```

    ## Loading required package: dplyr

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
require(tidyr)
```

    ## Loading required package: tidyr

``` r
require(lubridate)
```

    ## Loading required package: lubridate

    ## 
    ## Attaching package: 'lubridate'

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     intersect, setdiff, union

    ## The following objects are masked from 'package:base':
    ## 
    ##     date, intersect, setdiff, union

``` r
require(parsedate)
```

    ## Loading required package: parsedate

    ## 
    ## Attaching package: 'parsedate'

    ## The following object is masked from 'package:readr':
    ## 
    ##     parse_date

``` r
require(rlang)
```

    ## Loading required package: rlang

``` r
require(here)
```

    ## Loading required package: here

    ## here() starts at /Users/billmabe/Documents/GitHub/covid

``` r
require(stringr)
```

    ## Loading required package: stringr

``` r
require(purrr)
```

    ## Loading required package: purrr

    ## 
    ## Attaching package: 'purrr'

    ## The following objects are masked from 'package:rlang':
    ## 
    ##     %@%, as_function, flatten, flatten_chr, flatten_dbl, flatten_int,
    ##     flatten_lgl, flatten_raw, invoke, list_along, modify, prepend,
    ##     splice

``` r
require(rvest)
```

    ## Loading required package: rvest

    ## Loading required package: xml2

    ## 
    ## Attaching package: 'xml2'

    ## The following object is masked from 'package:rlang':
    ## 
    ##     as_list

    ## 
    ## Attaching package: 'rvest'

    ## The following object is masked from 'package:purrr':
    ## 
    ##     pluck

    ## The following object is masked from 'package:readr':
    ## 
    ##     guess_encoding

``` r
require(leaflet)
```

    ## Loading required package: leaflet

``` r
county_dat <- read_csv("https://raw.githubusercontent.com/nytimes/covid-19-data/master/us-counties.csv")
```

    ## Parsed with column specification:
    ## cols(
    ##   date = col_date(format = ""),
    ##   county = col_character(),
    ##   state = col_character(),
    ##   fips = col_character(),
    ##   cases = col_double(),
    ##   deaths = col_double()
    ## )

``` r
state_dat <- read_csv("https://raw.githubusercontent.com/nytimes/covid-19-data/master/us-states.csv")
```

    ## Parsed with column specification:
    ## cols(
    ##   date = col_date(format = ""),
    ##   state = col_character(),
    ##   fips = col_character(),
    ##   cases = col_double(),
    ##   deaths = col_double()
    ## )

``` r
tests <- read_csv("https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/testing/covid-testing-latest-data-source-details.csv")
```

    ## Parsed with column specification:
    ## cols(
    ##   `ISO code` = col_character(),
    ##   Entity = col_character(),
    ##   Date = col_date(format = ""),
    ##   `Source URL` = col_character(),
    ##   `Source label` = col_character(),
    ##   Notes = col_character(),
    ##   `Number of observations` = col_double(),
    ##   `Cumulative total` = col_double(),
    ##   `Cumulative total per thousand` = col_double(),
    ##   `Daily change in cumulative total` = col_double(),
    ##   `Daily change in cumulative total per thousand` = col_double(),
    ##   `3-day rolling mean daily change` = col_double(),
    ##   `3-day rolling mean daily change per thousand` = col_double(),
    ##   `7-day rolling mean daily change` = col_double(),
    ##   `7-day rolling mean daily change per thousand` = col_double(),
    ##   `General source label` = col_character(),
    ##   `General source URL` = col_character(),
    ##   `Short description` = col_character(),
    ##   `Detailed description` = col_character()
    ## )

## B. FIND AND REMOVE DUPLICATE RECORDS (IF THERE ARE ANY)

``` r
nrow(county_dat %>% distinct()) == nrow(county_dat)
```

    ## [1] TRUE

``` r
nrow(county_dat %>% distinct(date, county, state)) == nrow(county_dat)
```

    ## [1] TRUE

``` r
nrow(state_dat %>% distinct()) == nrow(state_dat)
```

    ## [1] TRUE

``` r
nrow(state_dat %>% distinct(date, state)) == nrow(state_dat)
```

    ## [1] TRUE

## C. ASCERTAIN WHETHER COUNTS ARE CUMULATIVE OR DAILY

Before doing anything with these data it is first necessary to determine
whether the data contains cumulative totals or daily counts.

How do we determine if the counts of cases and deaths are cumulative or
by date? If the value of Confirmed today is always greater than the
value of Confirmed yesterday, then we can assume that the counts are
cumulative.

#### 1\. County Data

``` r
county_distinct_cumul <- county_dat %>% 
  group_by(state, county) %>%
  mutate(lag_cases = lag(cases),
         lag_deaths = lag(deaths),
         cases_is_cumulative = ifelse(cases >= lag_cases, 1, 0),
         dth_is_cumulative = ifelse(deaths >= lag_deaths, 1, 0)
         )

table(county_distinct_cumul$cases_is_cumulative)
```

    ## 
    ##      0      1 
    ##   1832 110704

``` r
table(county_distinct_cumul$dth_is_cumulative)
```

    ## 
    ##      0      1 
    ##    393 112143

#### 2\. State Data

``` r
state_distinct_cumul <- state_dat %>% 
  group_by(state) %>%
  mutate(lag_cases = lag(cases),
         lag_deaths = lag(deaths),
         cases_is_cumulative = ifelse(cases >= lag_cases, 1, 0),
         dth_is_cumulative = ifelse(deaths >= lag_deaths, 1, 0)
         )
table(state_distinct_cumul$cases_is_cumulative)
```

    ## 
    ##    0    1 
    ##    4 3420

``` r
table(state_distinct_cumul$dth_is_cumulative)
```

    ## 
    ##    0    1 
    ##    5 3419

The tables show that in both data sets, the cases and deaths variables
are cumulative counts. In a very small number of cases, however, the
number of cases at time t is LESS than the number of cases at time t -
1. This suggests that perhaps the local area revised the case or death
counts since the original report. To accommodate this and to avoid
ending up with daily case values below zero, I use the lead function to
change the values of cases and deaths, when the future values of those
variables (in the next two days only) were LESS than the current values
of those variables. This is intended to improve the accuracy of the
daily counts and to backwards revise the cumulative counts so that they
reflect changes that local health authorities made to revise the case
and death counts. Of course, this assumes that more recent data are more
accurate than the original reports. This assumption could be wrong. I
think this is, however, a reasonable assumption that also has the
benefit of generating negative numbers of COVID-19 cases for some days.

## D. CLEAN THE ORIGINAL CUMULATIVE COUNT VARIABLES

#### 1\. County Data

``` r
county_distinct_rev <- county_distinct_cumul %>% 
  arrange(state, county, date) %>%
  group_by(state, county) %>%
  mutate(lead_cases = lead(cases),
         lead_deaths = lead(deaths),
         lead2_cases = lead(cases, 2),
         lead2_deaths = lead(deaths, 2),
         cases = ifelse(!is.na(lead_cases) & cases > lead_cases, lead_cases, cases),
         deaths = ifelse(!is.na(lead_deaths) & deaths > lead_deaths, lead_deaths, deaths),
         cases = ifelse(!is.na(lead2_cases) & cases > lead2_cases, lead2_cases, cases),
         deaths = ifelse(!is.na(lead2_deaths) & deaths > lead2_deaths, lead2_deaths, deaths)
         )
```

#### 2\. State Data

``` r
state_distinct_rev <- state_distinct_cumul %>% 
  arrange(state, date) %>%
  group_by(state) %>%
  mutate(lead_cases = lead(cases),
         lead_deaths = lead(deaths),
         lead2_cases = lead(cases, 2),
         lead2_deaths = lead(deaths, 2),
         cases = ifelse(!is.na(lead_cases) & cases > lead_cases, lead_cases, cases),
         deaths = ifelse(!is.na(lead_deaths) & deaths > lead_deaths, lead_deaths, deaths),
         cases = ifelse(!is.na(lead2_cases) & cases > lead2_cases, lead2_cases, cases),
         deaths = ifelse(!is.na(lead2_deaths) & deaths > lead2_deaths, lead2_deaths, deaths)
         )
```

## E. CREATE UNBROKEN TIME SERIES FOR EACH GEOGRAPHICAL UNIT

#### Complete the time series using `tidyr::complete`

``` r
county_complete <- county_distinct_rev %>% 
  complete(nesting(state, county), date = seq(min(date), max(date), by = "day"))

state_complete <- state_distinct_rev %>% 
  complete(nesting(state), date = seq(min(date), max(date), by = "day"))
```

#### See if there are any gaps left in the time series

``` r
gaps1 <- county_complete %>% 
            group_by(state, county) %>% 
            mutate(no_gap = date - lag(date))
table(gaps1$no_gap)
```

    ## 
    ##      1 
    ## 112952

``` r
gaps2 <- state_complete %>% 
            group_by(state) %>% 
            mutate(no_gap = date - lag(date))
table(gaps2$no_gap)
```

    ## 
    ##    1 
    ## 3424

If they all equal 1 means there are no gaps in the time series.

## F. FILL THE TIME SERIES UP – there are no gaps so this code is not necessary.

Convert the NAs to the last value of the variable using fill down, which
fills correctly, despite the name. I want to fill in the last recorded
value in time before the NA values. I do not want to back fill the most
recent value into the past where there are currently NAs.

``` r
#county_complete_filled <- county_complete %>% group_by(state, county) %>% fill(lag_cases:lead2_deaths, .direction = "down")
#state_complete_filled <- state_complete %>% group_by(state) %>% fill(lag_cases:lead2_deaths, .direction = "down")

#all.equal(county_complete_filled, county_complete)
#all.equal(state_complete_filled, state_complete)
```

## G. CREATE THE DAILY COUNT VARIABLES

#### 1\. County File

``` r
county_data <- county_complete %>% 
                filter(!is.na(county)) %>%
                rename(cumulative_cases = cases,
                       cumulative_deaths = deaths) %>%
                arrange(state, county, date) %>%
                group_by(state, county) %>%
                mutate(cases = cumulative_cases - lag(cumulative_cases),
                       deaths = cumulative_deaths - lag(cumulative_deaths))
```

#### 2\. State File

``` r
state_data <- state_complete %>% 
                filter(!is.na(state)) %>%
                rename(cumulative_cases = cases,
                       cumulative_deaths = deaths) %>%
                arrange(state, date) %>%
                group_by(state) %>%
                mutate(cases = cumulative_cases - lag(cumulative_cases),
                       deaths = cumulative_deaths - lag(cumulative_deaths))
```

# TESTING DATA

``` r
l <- str_split(tests$Entity, " - ")
tests <- tests %>% mutate(Country = map_chr(l, function(x) x[1]),
                          `Test Units` = map_chr(l, function(x) x[2])
                          ) %>%
                   arrange(Country, desc(`Cumulative total`)) %>%
                   group_by(Country) %>%
                   slice(1:1)
  
test_output <- tests %>% 
  arrange(desc(`Cumulative total per thousand`)) %>% 
  select(Country, `Test Units`, `Cumulative total per thousand`, `Cumulative total`)
nrow(test_output)
```

    ## [1] 82

# LATITUDE / LONGITUDE DATA

``` r
countries_url <- "https://developers.google.com/public-data/docs/canonical/countries_csv"
ctries <- read_html(countries_url)

# Read data from table
ctry_table <- html_nodes(ctries, "table td") %>%
  html_text()

# Create tibble
country_df <- as_tibble(matrix(ctry_table, ncol = 4, byrow = TRUE), .name_repair = "universal")
```

    ## New names:
    ## * `` -> ...1
    ## * `` -> ...2
    ## * `` -> ...3
    ## * `` -> ...4

``` r
# Pull column names from Google
nms <- html_nodes(ctries, "table") %>%
       html_text()
names(country_df) <- str_trim(str_split(nms, "\n")[[1]][1:4])

# Convert lat / lon from char to numeric
country_df <- country_df %>% mutate(latitude = as.numeric(latitude),
                                    longitude = as.numeric(longitude)
                                    )
country_df
```

    ## # A tibble: 245 x 4
    ##    country latitude longitude name                
    ##    <chr>      <dbl>     <dbl> <chr>               
    ##  1 AD          42.5    1.60   Andorra             
    ##  2 AE          23.4   53.8    United Arab Emirates
    ##  3 AF          33.9   67.7    Afghanistan         
    ##  4 AG          17.1  -61.8    Antigua and Barbuda 
    ##  5 AI          18.2  -63.1    Anguilla            
    ##  6 AL          41.2   20.2    Albania             
    ##  7 AM          40.1   45.0    Armenia             
    ##  8 AN          12.2  -69.1    Netherlands Antilles
    ##  9 AO         -11.2   17.9    Angola              
    ## 10 AQ         -75.3   -0.0714 Antarctica          
    ## # … with 235 more rows

# Join Testing Data with Lat Lon Data

``` r
test_loc <- inner_join(test_output, country_df, by = c("Country" = "name"))
str(test_loc)
```

    ## tibble [81 × 7] (S3: grouped_df/tbl_df/tbl/data.frame)
    ##  $ Country                      : chr [1:81] "Iceland" "Bahrain" "Luxembourg" "Lithuania" ...
    ##  $ Test Units                   : chr [1:81] "samples" "units unclear" "people tested" "samples tested" ...
    ##  $ Cumulative total per thousand: num [1:81] 150.3 91.4 78.8 54.8 47.8 ...
    ##  $ Cumulative total             : num [1:81] 51304 155501 49299 149106 413517 ...
    ##  $ country                      : chr [1:81] "IS" "BH" "LU" "LT" ...
    ##  $ latitude                     : num [1:81] 65 25.9 49.8 55.2 31 ...
    ##  $ longitude                    : num [1:81] -19.02 50.64 6.13 23.88 34.85 ...
    ##  - attr(*, "groups")= tibble [81 × 2] (S3: tbl_df/tbl/data.frame)
    ##   ..$ Country: chr [1:81] "Argentina" "Australia" "Austria" "Bahrain" ...
    ##   ..$ .rows  :List of 81
    ##   .. ..$ : int 65
    ##   .. ..$ : int 22
    ##   .. ..$ : int 17
    ##   .. ..$ : int 2
    ##   .. ..$ : int 76
    ##   .. ..$ : int 26
    ##   .. ..$ : int 16
    ##   .. ..$ : int 75
    ##   .. ..$ : int 47
    ##   .. ..$ : int 24
    ##   .. ..$ : int 39
    ##   .. ..$ : int 59
    ##   .. ..$ : int 63
    ##   .. ..$ : int 43
    ##   .. ..$ : int 51
    ##   .. ..$ : int 23
    ##   .. ..$ : int 7
    ##   .. ..$ : int 55
    ##   .. ..$ : int 52
    ##   .. ..$ : int 9
    ##   .. ..$ : int 80
    ##   .. ..$ : int 30
    ##   .. ..$ : int 40
    ##   .. ..$ : int 19
    ##   .. ..$ : int 54
    ##   .. ..$ : int 46
    ##   .. ..$ : int 28
    ##   .. ..$ : int 44
    ##   .. ..$ : int 1
    ##   .. ..$ : int 73
    ##   .. ..$ : int 79
    ##   .. ..$ : int 50
    ##   .. ..$ : int 8
    ##   .. ..$ : int 5
    ##   .. ..$ : int 11
    ##   .. ..$ : int 60
    ##   .. ..$ : int 32
    ##   .. ..$ : int 77
    ##   .. ..$ : int 12
    ##   .. ..$ : int 4
    ##   .. ..$ : int 3
    ##   .. ..$ : int 48
    ##   .. ..$ : int 74
    ##   .. ..$ : int 69
    ##   .. ..$ : int 78
    ##   .. ..$ : int 36
    ##   .. ..$ : int 15
    ##   .. ..$ : int 81
    ##   .. ..$ : int 13
    ##   .. ..$ : int 68
    ##   .. ..$ : int 45
    ##   .. ..$ : int 64
    ##   .. ..$ : int 38
    ##   .. ..$ : int 67
    ##   .. ..$ : int 41
    ##   .. ..$ : int 6
    ##   .. ..$ : int 10
    ##   .. ..$ : int 42
    ##   .. ..$ : int 18
    ##   .. ..$ : int 61
    ##   .. ..$ : int 70
    ##   .. ..$ : int 33
    ##   .. ..$ : int 25
    ##   .. ..$ : int 31
    ##   .. ..$ : int 21
    ##   .. ..$ : int 53
    ##   .. ..$ : int 37
    ##   .. ..$ : int 20
    ##   .. ..$ : int 34
    ##   .. ..$ : int 14
    ##   .. ..$ : int 57
    ##   .. ..$ : int 66
    ##   .. ..$ : int 62
    ##   .. ..$ : int 35
    ##   .. ..$ : int 71
    ##   .. ..$ : int 56
    ##   .. ..$ : int 29
    ##   .. ..$ : int 27
    ##   .. ..$ : int 49
    ##   .. ..$ : int 58
    ##   .. ..$ : int 72
    ##   ..- attr(*, ".drop")= logi TRUE

# Draw Map of Cumulative Tests

# Write csv files to save

``` r
write.csv(county_data, file = here("county_data.csv"), row.names = FALSE)
write.csv(state_data, file = here("state_data.csv"), row.names = FALSE)
write.csv(county_data, file = "~/Documents/Projects/covid_app/county_data.csv", row.names = FALSE)
write.csv(state_data, file = "~/Documents/Projects/covid_app/state_data.csv", row.names = FALSE)
write.csv(test_loc, file = here("test_output.csv"), row.names = FALSE)
write.csv(test_loc, file = "~/Documents/Projects/covid_app/test_output.csv", row.names = FALSE)
```
