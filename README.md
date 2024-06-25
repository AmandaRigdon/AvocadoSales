## Overview

Because avocado is my favorite fruit (yes, fruit) ever, I figured why not do a project on avocados? 

I headed on over to Kaggle to find a nice dataset of avocado prices around the US from 2015 - 2018 (partial 2018).

[Avocado Prices](https://www.kaggle.com/datasets/neuromusic/avocado-prices)

As a millenial, I'm buying a lot of avocados for my avocado toast and avocado-filled recipes, so I opened this dataset with a couple of burning questions:

1. Where are the cheapest avocados located? What about the most expensive?
2. What's the price difference between organic and regular? Is it really that large?
3. What time of year do avocado sales do best?
4. Do people prefer buying loose avocado or bagged?

### The Dataset

This data is ranked as already being pretty clean, but we have a couple of issues here right off the bat:

* Region: Regions are smushed together, and they don't make a ton of sense. As an American, I can guess what city is where, but otherwise this would be very confusing for someone outside of the US. 
* Index: I initially thought this would be similar to an "id" column, but it actually signifies a different day/year for the same region...my plan to check for duplicates will become more tedious.
* Spacing: The column names have some spacing which MySQL does not like at all. So those will need to be fixed.
* Timeframe: This dataset cuts off in March of 2018, so if we look at anything by year, the numbers will be skewed.

With that in mind, here's my handy dandy plan to clean the data before it's actually looked at:

* Remove Duplicates
* Change Column Names
* Standardize Data
* Nulls
* Remove columns/data that is not necessary

I'll be doing all data cleaning and EDA in MySQL. Then I'll plug in that data to Tableau to create some nice visuals!

## Data Cleaning


