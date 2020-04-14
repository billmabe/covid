# COVID
Cleaning the NY Times COVID-19 data to create daily time series

This script produces two csv files
1. US county -- daily COVID-19 case counts (https://raw.githubusercontent.com/billmabe/covid/master/county_data.csv)
2. US state -- daily COVID-19 case counts (https://raw.githubusercontent.com/billmabe/covid/master/state_data.csv)

To that end, the script, 
- reads in state and country data files from the NY Times' GitHub repo
- removes duplicate observations (none yet found)
- tests each of the three files for whether their values for case and deaths variables are cumulative sums or daily counts.
- revise cumulative case totals for days where cumulative totals decreased from previous day
- create unbroken time series
- fill time series "up" so that all rows have the correct values (not necessary)
- create the daily count variables
