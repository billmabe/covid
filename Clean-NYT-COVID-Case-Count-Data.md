Cleaning the NY Times COVID-19 Case Count Data
================
Bill Mabe
April 10, 2020

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
    ##     0     1 
    ##  1368 88365

``` r
table(county_distinct_cumul$dth_is_cumulative)
```

    ## 
    ##     0     1 
    ##   314 89419

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
    ##    2 2982

``` r
table(state_distinct_cumul$dth_is_cumulative)
```

    ## 
    ##    0    1 
    ##    4 2980

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
county_complete <- county_distinct_rev %>% complete(nesting(state, county), date = seq(min(date), max(date), by = "day"))
state_complete <- state_distinct_rev %>% complete(nesting(state), date = seq(min(date), max(date), by = "day"))
```

#### See if there are any gaps left in the time series

``` r
gaps1 <- county_complete %>% 
            group_by(state, county) %>% 
            mutate(no_gap = date - lag(date))
table(gaps1$no_gap)
```

    ## 
    ##     1 
    ## 90090

``` r
gaps2 <- state_complete %>% 
            group_by(state) %>% 
            mutate(no_gap = date - lag(date))
table(gaps2$no_gap)
```

    ## 
    ##    1 
    ## 2984

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

# Write csv files to save

``` r
write.csv(county_data, file = "~/Documents/GitHub/covid/county_data.csv", row.names = FALSE)
write.csv(state_data, file = "~/Documents/GitHub/covid/state_data.csv", row.names = FALSE)
write.csv(county_data, file = "~/Documents/Projects/covid/county_data.csv", row.names = FALSE)
write.csv(state_data, file = "~/Documents/Projects/covid/state_data.csv", row.names = FALSE)
```
