---
layout: post
title: Analyzing Active Covid Cases with Python
image: "/posts/coronavirus-2.jpg"
tags: [Python, Primes]
---

First let's start by importing all of the required modules needed for this exercise.  All of these might not be known at the start of your planning and mapping out you data analysis but the list can be added to later.  Let's get started

```ruby
#Installs

#Imported Modules
import datetime as dt
import urllib.request
import datetime
import calendar
import matplotlib.pyplot as plt
import seaborn as sns

# essential libraries
import json
import random
from urllib.request import urlopen

# storing and anaysis
import numpy as np
import pandas as pd

# visualization
import plotly.express as px
import plotly.graph_objs as go
import plotly.figure_factory as ff

# color pallette
cnf = '#393e46' # confirmed - grey
dth = '#ff2e63' # death - red
rec = '#21bf73' # recovered - cyan
act = '#fe9801' # active case - yellow

# converter
from pandas.plotting import register_matplotlib_converters
register_matplotlib_converters()   

# hide warnings
import warnings
warnings.filterwarnings('ignore')
```

---

Now lets import the raw data that have been loaded and saved as .CSV datasets.  For this project the files were downloaded from Johns Hopkins website.

```ruby
urls = 
[
    'https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_covid19_confirmed_global.csv',
    'https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_covid19_deaths_global.csv',
    'https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_covid19_recovered_global.csv'
]

#Get data from URL and set into CSV files - #for Windows use wget - #for Jupyter use urllib.request
[urllib.request.urlretrieve(url) for url in urls]
```

Now that we have imported our .CSV files and saved them, we can now retrieve and start to read the files

```ruby
confirmed_df = pd.read_csv('time_series_covid19_confirmed_global.csv')
deaths_df = pd.read_csv('time_series_covid19_deaths_global.csv')
recovered_df = pd.read_csv('time_series_covid19_recovered_global.csv')
```

After analyzing the files
```ruby
dates = confirmed_df.columns[4:]
confirmed_df_long = confirmed_df.melt(
    id_vars=['Province/State', 'Country/Region', 'Lat', 'Long'],
    value_vars=dates,
    var_name='Date',
    value_name='Confirmed'
)
deaths_df_long = deaths_df.melt(
    id_vars=['Province/State', 'Country/Region', 'Lat', 'Long'],
    value_vars=dates,
    var_name='Date',
    value_name='Deaths'
)
recovered_df_long = recovered_df.melt(
    id_vars=['Province/State', 'Country/Region', 'Lat', 'Long'],
    value_vars=dates,
    var_name='Date',
    value_name='Recovered'
)

# Merging confirmed_df_long and deaths_df_long
full_table = confirmed_df_long.merge(
  right=deaths_df_long,
  how='left',
  on=['Province/State', 'Country/Region', 'Date', 'Lat', 'Long']
)
# Merging full_table and recovered_df_long
full_table = full_table.merge(
  right=recovered_df_long,
  how='left',
  on=['Province/State', 'Country/Region', 'Date', 'Lat', 'Long']
)

#Rename Columns for using in Plots
full_table.rename({"Country/Region":"Country_Region", "Province/State":"Province_State"}, axis=1, inplace=True)

print(full_table.head(10))

full_table.to_csv('full_table.csv')
```

After determining what type of additional data needed in our dataset for reporting later, I decided to add the Month, Day and Year columns and then re-index the columns of data that were date related so that they were all together.
```ruby
full_table = pd.read_csv('full_table.csv', index_col=0)

full_table['Date'] = pd.to_datetime(full_table['Date'])

full_table['Day'] = full_table['Date'].dt.day
full_table['Month'] = full_table['Date'].dt.month.apply(lambda x: calendar.month_name[x])
full_table['Year'] = full_table['Date'].dt.year

full_table = pd.read_csv('full_table.csv', index_col=0)

full_table = full_table[['Province_State', 'Country_Region', 'Lat', 'Long', 'Date', 'Day', 'Month', 'Year', 'Confirmed', 'Deaths', 'Recovered']]

full_table.to_csv('full_table.csv')

```

There was NaN values as one of the columns Province/State did not have any valid data as not all countries had Provinces or States so NaN was replaced with '0'.
```ruby
full_table = pd.read_csv('full_table.csv', index_col=0)

full_table['Recovered'] = full_table['Recovered'].fillna(0)

print(full_table.isna().sum())
```

DUring a review of where our data was at we recognized data where Cruise Ships were at ports as these are not countries and cannot define which country figures should be connected to so this will remove them from the dataframe
```ruby
full_table = pd.read_csv('full_table.csv', index_col = 0)

ship_rows = full_table['Province_State'].str.contains('Grand Princess') \
            | full_table['Province_State'].str.contains('Diamond Princess') \
            | full_table['Country_Region'].str.contains('Diamond Princess') \
            | full_table['Country_Region'].str.contains('MS Zaandam')

full_ship = full_table[ship_rows]

full_table = full_table[~(ship_rows)]

full_table.to_csv('full_table.csv')
```

This was one of the last steps performed on the data to aggregate the information on the dataset to make the dataset look cleaner in preparation of our vizualisations.
```ruby
full_table = pd.read_csv('full_table.csv', index_col = 0)

full_table['Active'] = full_table['Confirmed'] - full_table['Recovered'] - full_table['Deaths']

full_grouped = full_table.groupby(['Date', 'Month', 'Year', 'Country_Region', 'Province_State'])[['Confirmed', 'Deaths', 'Recovered', 'Active']].sum().reset_index()

# new cases
temp = full_grouped.groupby(['Country_Region', 'Date'])[['Confirmed', 'Deaths', 'Recovered']]
temp = temp.sum().diff().reset_index()
mask = temp['Country_Region'] != temp['Country_Region'].shift(1)
temp.loc[mask, 'Confirmed'] = np.nan
temp.loc[mask, 'Deaths'] = np.nan
temp.loc[mask, 'Recovered'] = np.nan
# renaming columns
temp.columns = ['Country_Region', 'Date', 'New cases', 'New deaths', 'New recovered']
# merging new values
full_grouped = pd.merge(full_grouped, temp, on=['Country_Region', 'Date'])
# filling na with 0
full_grouped = full_grouped.fillna(0)
# fixing data types
cols = ['New cases', 'New deaths', 'New recovered']
full_grouped[cols] = full_grouped[cols].astype('int')
#
full_grouped['New cases'] = full_grouped['New cases'].apply(lambda x: 0 if x<0 else x)

#Final Completed Output
full_grouped.to_csv('COVID-19-Completed.csv')

#Print Head (10)
print(full_grouped.head(8))

```

Our first vizualization will be of th eTop 20 Countries in order of country with the most active cases.  For our vizualiation we will be using a simple bar plot

```ruby
#Load data from saved CSV
df = pd.read_csv('COVID-19-Completed.csv')

# Group df by country
df = df.groupby('Country_Region').agg('sum')

# Choose relevant columns
df = df[['Confirmed', 'Deaths', 'Recovered', 'Active']]

# Create dataFrame, df_active, of 20 countries with most active cases
df_recovered = df.sort_values(by='Deaths', ascending=False)[:20]

# Convert index of countries to series for plotting
x_vals = df_recovered.index.to_series()

# Select 'Confirmed' column as y-values
y_vals = df_recovered['Deaths']

# Set dark grid
sns.set()

# Convert index of countries to series for plotting
y_vals = df_recovered.index.to_series()

# Select 'Active' column as y-values
x_vals = df_recovered['Deaths']

# Set size of figure
plt.figure(figsize=(16,9))

# Create horizontal bar plot
sns.barplot(x=x_vals, y=y_vals, palette='Greens_r')

# Title plot
plt.title('Coronavirus Active through June 26, 2020', size=15)

# Save figure
plt.savefig('Coronavirus Active through June 26, 2020', dpi=300)

# Show plot
plt.show()

```

Our final vizualization is the number of Active Cases, Deaths and Recoveries over a period of time
```ruby
temp = full_table.groupby('Date')['Recovered', 'Deaths', 'Active'].sum().reset_index()
temp = temp.melt(id_vars="Date", value_vars=['Recovered', 'Deaths', 'Active'],
                 var_name='Case', value_name='Count')
temp.head()

fig = px.area(temp, x="Date", y="Count", color='Case',
             title='Cases over time', color_discrete_sequence = [rec, dth, act])
fig.update_layout(margin=dict(t=80,l=0,r=0,b=0))
fig #.write_image('covid-eda-2-1.png')
```



![Active Cases](https://user-images.githubusercontent.com/51378038/138585169-45a970fe-96e3-424a-accf-d98d8096afda.jpg)
![Cases Over Time](https://user-images.githubusercontent.com/51378038/138585172-e9cfb003-3983-4f44-912b-93c865e0314c.jpg)
