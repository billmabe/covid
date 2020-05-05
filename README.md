# COVID
Cleaning the NY Times COVID-19 data to create daily time series and also copying country testing data from https://github.com/owid/covid-19-data/tree/master/public/data/

Sources:
Case count data: NY Times
COVID-19 testing: Our World in Data

This script produces three csv files
1. US county -- daily COVID-19 case counts (https://raw.githubusercontent.com/billmabe/covid/master/county_data.csv)
2. US state -- daily COVID-19 case counts (https://raw.githubusercontent.com/billmabe/covid/master/state_data.csv)
3. test_output -- cumulative tests by country from Our World in Data referenced above and output on my Github here: (https://raw.githubusercontent.com/billmabe/covid/master/test_output.csv)

For the case count data: 
- Reads in state and country data files from the NY Times' GitHub repo.
- Removes duplicate observations (none yet found).
- Tests each of the three files for whether their values for case and deaths variables are cumulative sums or daily counts.
- Revise cumulative case totals for days where cumulative totals decreased from previous day.
- Create unbroken time series.
- Fill time series "up" so that all rows have the correct values (not necessary).
- Create the daily count variables.

For the testing data:
The script pulls three variables from the test data gathered by Our World in Data and keeps just three variables:
- Entity
- Cumulative tests
- Cumulative test per 1,000 population
