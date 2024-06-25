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

I'll be creating a duplicate table to make sure that any changes I make don't affect the original.

```sql
CREATE TABLE avocadostaging
LIKE avocadoprices;
```

```sql
INSERT avocadostaging
SELECT *
from avocadoprices;
```

### Removing Duplicates/Renaming Columns

Like I mentioned before, I'll have to add a column to make a unique id for each row, so that we can see if there's any duplicates.

But before we do that, we'll have to rename some columns:

```sql
ALTER TABLE avocadostaging
RENAME COLUMN MyUnknownColumn TO `index`,
RENAME COLUMN `Total Volume` to TotalVolume,
RENAME COLUMN `Total Bags` to TotalBags,
RENAME COLUMN `Small Bags` to SmallBags,
RENAME COLUMN `XLarge Bags` to XLargeBags,
RENAME COLUMN `Large Bags` to LargeBags;
```

Now that these are renamed, we shouldn't run into any issues when we go to add a row number.

```sql
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY `index`, `Date`, AveragePrice, TotalVolume, `4046`, `4225`, `4770`, TotalBags, SmallBags, LargeBags, XLargeBags, `type`, `year`, region) AS row_num
FROM avocadostaging;
```

I'll filter where row_num is greater than 2 and create a CTE to do so.

```sql
WITH duplicate_cte AS
(SELECT *,
ROW_NUMBER() OVER(
PARTITION BY `index`, `Date`, AveragePrice, TotalVolume, `4046`, `4225`, `4770`, TotalBags, SmallBags, LargeBags, XLargeBags, `type`, `year`, region) AS row_num
FROM avocadostaging
)
SELECT *
FROM duplicate_cte
WHERE row_num > 1;
```

When I ran this, it returned zero results, so there's no duplicates! Yay.

### Remove Nulls

For brevity, I won't be listing all of it, but I checked each column and found no null values.

```sql
SELECT *
FROM avocadostaging
WHERE `index` IS NULL;
```

### Standardize Data

The date column in this dataset is set to text, so I'll change it to a date format.

```sql
ALTER TABLE avocadostaging
MODIFY COLUMN `Date` DATE;
```

It's hard to tell if anything needs to be trimmed. The rows I saw looked okay, but considering there's over 18,000 of them, it's best to play it safe.

```sql
SELECT TRIM(`index`),
TRIM(`Date`),
TRIM(AveragePrice),
TRIM(TotalVolume),
TRIM(`4046`),
TRIM(`4225`),
TRIM(`4770`),
TRIM(TotalBags),
TRIM(SmallBags),
TRIM(LargeBags),
TRIM(XLargeBags),
TRIM(`type`),
TRIM(`year`),
TRIM(region)
FROM avocadostaging;
```

Now let's update the table!

```sql
UPDATE avocadostaging
SET `index` = TRIM(`index`),
`Date` = TRIM(`Date`),
AveragePrice = TRIM(AveragePrice),
TotalVolume = TRIM(TotalVolume),
`4046` = TRIM(`4046`),
`4225` = TRIM(`4225`),
`4770` = TRIM(`4770`),
TotalBags = TRIM(TotalBags),
SmallBags = TRIM(SmallBags),
LargeBags = TRIM(LargeBags),
XLargeBags = TRIM(XLargeBags),
`type` = TRIM(`type`),
`year` = TRIM(`year`),
region = TRIM(region);
```

One thing that I noticed is that there are mostly decimal values in the table. It doesn't make sense to say .48 avocados, so I'm going to round all these to create more clarity.

```sql
UPDATE avocadostaging
SET TotalVolume = ROUND(TotalVolume, 0),
`4046` = ROUND(`4046`, 0),
`4225` = ROUND(`4225`, 0),
`4770` = ROUND (`4770`, 0),
TotalBags = ROUND(TotalBags, 0),
SmallBags = ROUND(SmallBags, 0),
LargeBags = ROUND(LargeBags, 0),
XLargeBags = ROUND(XLargeBags, 0);
```



