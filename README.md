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

Time to address region - there's some very vague ones like "Midsouth" and "West." There's also "TotalUS" which would just be duplicated data of everything we have if it is indeed the entire country.

Since this is just an EDA, I'll look into how many rows would be removed if we took these out. In real life, this would be something that would need clarification of course.

```sql
SELECT COUNT(region)
from avocadostaging
WHERE region = 'Midsouth' AND region = 'TotalUS' AND region = 'West';
```
All in all, it's about 1,000 rows to remove. This is quite a bit, but this data is not useful for analysis so I'll remove them.

```sql
DELETE FROM avocadostaging
WHERE region = 'Midsouth' AND region = 'TotalUS' AND region = 'West';
```

After considering the data again, the regions are still going to be difficult to parse out. I can at least enter spacing, but some cities are lumped in with each other. There are also entire states in this column, so the data will look very skewed.
I'm going to continue to remove some data, specifically anything under "California" and "South Carolina"

```sql
DELETE FROM avocadostaging
WHERE region = 'California'; AND region = 'South Carolina';
```

Now I'd like to separate the region names from camelcase into spaced values. This turned out to be something that's a lot harder than I originally envisioned. I had to search online for a function that would do this for me.

```sql
DELIMITER @@
DROP FUNCTION IF EXISTS change_case@@
CREATE FUNCTION change_case (word VARCHAR(15000)) RETURNS VARCHAR(15000)
COMMENT 'Change the case to Proper Case, A-Z (65-90 decimal|ASCII) , 
  but if first char in between a-z (97-122) than change case to upper' DETERMINISTIC
BEGIN
  SET @text = word; -- input str
  SET @result = ""; -- modified str
  SET @i = 2;       -- counter
  -- if first char is in between a-z than change to upper
  IF ASCII(SUBSTRING(@text, 1, 1)) BETWEEN 97 AND 122 THEN 
    SET @text = CONCAT(UPPER(SUBSTRING(@text, 1, 1)), SUBSTRING(@text, 2));
  END IF;
  SET @result = UPPER(SUBSTRING(@text, 1, 1));
  WHILE @i <= LENGTH(@text) DO
    SET @t = SUBSTRING(@text, @i, 1);
    SET @p = SUBSTRING(@text, @i-1, 1);
    -- if curr_char is upper and pre_char is lower then insert space and the char       eg coName   > co Name
    IF ASCII(@t) BETWEEN 65 AND 90 AND ASCII(@p) BETWEEN 97 AND 122 THEN
    -- eg SomeCO   > Some CO
        SET @result = CONCAT(@result,' ', @t);
    -- pre_char is space & curr_char is lower
    ELSEIF (ASCII(@p) = 32 OR @p = '_' OR @p = '-') AND ASCII(@t) BETWEEN 97 AND 122 THEN
    -- eg some cO  > some CO
      SET @result = CONCAT(@result,' ', UPPER(@t));
    ELSEIF @t = '_' OR @t = '-' THEN
    -- Replace _ OR - > space
      SET @result = CONCAT(@result,' ');
    -- for lower case
    ELSE
    -- someco        > someco
      SET @result = CONCAT(@result, @t);
    END IF;
    SET @i = @i + 1;
  END WHILE;
  SET @result = REGEXP_REPLACE(@result, '[ ]+', ' ');
  RETURN @result;
END @@
```

Checking to see that the function did its job.

```sql
SELECT change_case(Region)
from avocadostaging;
```

Looks like it did! Let's update the table.

```sql
UPDATE avocadostaging
SET region = change_case(region);
```

## EDA

Let's have a look at the cheapest average price of avocados:

```sql
SELECT MIN(AveragePrice)
from avocadostaging;
```

<img width="110" alt="Screenshot 2024-06-26 at 11 43 46 AM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/14c77f63-b4a2-4668-8d51-1f5bfd23992d">

.44 cents! That would be a dream in 2024. This may be from an area where there's avocados locally.

How about the max average price?

```sql
SELECT MAX(AveragePrice)
from avocadostaging;
```

<img width="115" alt="Screenshot 2024-06-26 at 11 50 33 AM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/c5e0c7c8-9fc3-4e76-841d-378490654635">

$3.25 even in 2024 is very high for a singular avocado. This may be in an area where avocados aren't grown locally?

Which years sold the most avocados?

```sql
SELECT Year(Date), SUM(TotalVolume)
from avocadostaging
GROUP BY Year(Date)
ORDER BY Year(Date);
```
<img width="173" alt="Screenshot 2024-06-26 at 11 58 48 AM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/06e04275-2269-4dfa-b294-c008dfaac6e8">

Looks like 2017 was the highest year, but I can't be completely sure because I only have data from Jan-March of 2018.

Let's have a look by month

```sql
SELECT Month(Date), SUM(TotalVolume)
from avocadostaging
GROUP BY Month(Date)
ORDER BY Month(Date);
```
<img width="177" alt="Screenshot 2024-06-26 at 12 09 58 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/a91b571b-05b8-4851-be37-e6019dac6294">

The 2018 data is definitely changing the outcome here... but if we go by the other months, it looks like May is pretty popular. Would this maybe be due to cincdo de Mayo celebrations?

What about organic and conventional avocados? Which is more popular?

```sql
SELECT COUNT(type)
from avocadostaging
WHERE type = "conventional";
```
<img width="103" alt="Screenshot 2024-06-26 at 1 58 33 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/0de0b0e6-67c7-4fa9-89c4-a8848bafd909">

```sql
SELECT COUNT(type)
from avocadostaging
WHERE type = "organic";
```
<img width="80" alt="Screenshot 2024-06-26 at 1 49 44 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/c493c494-88e2-474e-90f0-ef06526bae6e">

Surprisingly, these are almost equal! I would think conventional avocados would be more popular because they are typically much cheaper.

Speaking of being cheaper, what's the total cost of conventional avocados vs. organic?

```sql
SELECT type, SUM(AveragePrice)
from avocadostaging
group by type
ORDER BY SUM(AveragePrice);
```
<img width="190" alt="Screenshot 2024-06-26 at 2 01 53 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/1f053641-4495-41e4-95ab-c289af57bdd1">

Clearly, organic avocados cost quite more. 30% more in fact! And since the # of conventional and organic avocados bought is almost equal, that price increase doesn't seem to sway customers.

So where would we go to find the cheapest organic avocados?

```sql
SELECT SUM(AveragePrice), type, region
from avocadostaging
WHERE type = "organic"
GROUP BY region
ORDER BY SUM(AveragePrice);
```
<img width="393" alt="Screenshot 2024-06-26 at 2 04 36 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/e7058e78-e8ae-488e-afc7-532a72be4f26">

Looks like in this dataset, Houston has the cheapest! 

What about the cheapest conventional avocados?

```sql
SELECT SUM(AveragePrice), type, region
from avocadostaging
WHERE type = "conventional"
GROUP BY region
order by SUM(AveragePrice);
```

<img width="313" alt="Screenshot 2024-06-26 at 2 13 25 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/9066f8fa-2585-47e0-b2c5-c5880d8e1445">

Phoenix/Tuscon areas followed by Houston have the cheapest regular avocados. 

Do people prefer buying bagged or loose avocados?

```sql
SELECT SUM(4046), SUM(4225), SUM(4770), SUM(SmallBags), sum(LargeBags), SUM(XLargeBags)
from avocadostaging;
```
<img width="497" alt="Screenshot 2024-06-26 at 2 19 45 PM" src="https://github.com/AmandaRigdon/AvocadoSales/assets/137234405/8c5bd3dc-c035-44de-a858-3520b3fd629d">

Looks like PLU 4225 avocados are the most popular. The 4225 PLU is for a generally large avocado that's 8-10 ounces in weight. If people are buying bags, then they are the small ones. This makes sense since most families can't get thru a huge bag of avocados before they spoil.

I also went ahead and made some visualizations in Tableau [here!](https://public.tableau.com/views/avocados_17194371192000/Dashboard12?:language=en-US&publish=yes&:sid=&:display_count=n&:origin=viz_share_link)
