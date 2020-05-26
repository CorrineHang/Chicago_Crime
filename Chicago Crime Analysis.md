# Chicago Crime Analysis

Chicago is one of the most dangerous cities in the US. This analysis is made according to the crime record tracked by the Chicago Police Department's Bureau of Records since 2001.

## Is the city safer than before?

```sql
--Data Selection and Pre-processing

SELECT unique_key, case_number, date, year,
  EXTRACT(MONTH FROM date) AS month, 
  FORMAT_DATE('%a',  DATE(date)) AS dayofweek,
  EXTRACT(HOUR FROM date) AS hour,
  iucr, primary_type, description, 
  location_description, arrest, community_area
FROM chicago_crime.crime
```

### Crime rate by year

```sql
WITH rate AS (
SELECT year, COUNT(case_number) AS crime_rate
FROM chicago_crime.crime
WHERE year < 2020
GROUP BY 1
ORDER BY 1),
rate_change AS (
SELECT year, crime_rate,
  FORMAT('%3.2f', 100*(crime_rate/(LAG(crime_rate) OVER (ORDER BY year ASC))-1)) AS drop_rate
FROM rate)
--SELECT AVG(CAST(drop_rate AS FLOAT64)) FROM rate_change
SELECT * FROM rate_change
```

In general, crime rate in Chicago has dropped 46.70% compared to 2001, with an average of 3.38% per year.

## Crime Pattern in Chicago

### Crime rate by month and type

```sql
SELECT FORMAT_DATE('%Y/%m', DATE(TIMESTAMP_TRUNC(date, MONTH))) AS Monthly, primary_type, COUNT(case_number) as Num FROM chicago_crime.data
WHERE TIMESTAMP_DIFF(date, TIMESTAMP '2020-05-01', DAY) < 0
GROUP BY 1, 2
ORDER BY 1 ASC, 3 DESC
```

There's a seasonal pattern of crime rate, which can be explained by top crime types like theft, battery, criminal damages etc. These crimes dropped significantly when the number of activities surge in summer and plunged in winter. The pattern doesn't seem to work in 2020 when spring comes, probably because of the Covid-19 pandemic.

```sql
SELECT year, primary_type, COUNT(*) AS num,
  FORMAT('%3.2f',100*COUNT(*)/(SUM(COUNT(*)) OVER(PARTITION BY year))) AS perc,
  RANK() OVER(PARTITION BY year ORDER BY COUNT(*) DESC) AS rank
FROM chicago_crime.data
WHERE year >= 2010
GROUP BY 1,2
ORDER BY 1,3 DESC
```

The list of top 10 crimes didn't change during last 10 years, but the order changed. Huge drop starting from 2016 in narcotics probably due to the relaxation of cannabis possession. In 2019, Theft, battery, criminal damage and assault are the top 4 types and takes up >60% of crime rates.

## Top types features

### Theft

not much difference among day of week, people should be careful when walking on street. There were much fewer crimes on street during late night and early morning. Make sense because fewer people on street then.

### Battery

More reports on weekends. Can happen anywhere and anytime during a day.

### Criminal Damage

Mostly to vehicle and property. More reports after 6pm.

### Assault

Mostly not with weapon. Fewer in early morning but and can be from anywhere.

## Gun-related crimes and murders

```sql
SELECT * 
FROM chicago_crime.data
WHERE iucr IN (
  SELECT IUCR
  FROM chicago_crime.IUCR_codes
  WHERE description like '%GUN%')
```

Unfortunately, violent crimes in Chicago didn't seem to decrease a lot in the past five years. 

In fact, in 2016, homicides and gun-related crimes took up 3,397 of the 4,966 crimes increased. According to Wikipedia, Chicago was responsible for nearly half of 2016's increase in homicides in the US, though the nation's crime rates remain near historic lows.

But there is a difference regarding the composition of gun-related crime. These reports fell more into the weapon violation category, which might be less severe than other ones like robbery and assault.

## Community Safety

#### Check data quality

```sql
SELECT year, countif(community_area is not null)/count(*) AS perc
FROM chicago_crime.data
GROUP BY 1
ORDER BY 1
```

Data after 2003(>0.999) can be used. Use data 2010 and 2017, taking census data availability into account.

GEO?

Data Studio doesn't support zipcode level geo map.

### Which communities are safer and which are more dangerous?

```sql
SELECT area_num, community, num_crime, FORMAT('%3.4f',crime_density) AS crime_density_area, RANK() OVER(ORDER BY crime_density ASC) AS RANK
FROM (
  SELECT B.AREA_NUM AS area_num, COMMUNITY AS community, COUNT(*) AS num_crime, SHAPE_AREA AS area, COUNT(*)/SHAPE_AREA*10000 AS crime_density
  FROM chicago_crime.crime A
  JOIN community_data.community_area B
  ON A.community_area = B.AREA_NUM
  WHERE year = 2019
  GROUP BY 1, 2, 4
  ORDER BY 1) C
ORDER BY 1
```

Together with community map, central/west/near south worse than far north/south.

Using area VS using population. Some communities may have people conduct crimes but not a residence here and may not be a good indicator of how dangerous it is visiting here. Some communities may be large and not many residency, and using area do not reflect the real density.

### Relationship between arrest rate and crime rate

```sql
SELECT D.*, RANK() OVER(ORDER BY crime_density_ppl_2010 ASC) AS Crime_2010_rank,
  RANK() OVER(ORDER BY crime_density_ppl_2017 ASC) AS Crime_2017_rank,
  ArrestRank
FROM (
  SELECT COMMUNITY AS community, COUNTIF(year = 2019) AS crime_2019, 
    ppl_2010, FORMAT('%3.4f',COUNTIF(year = 2010)/ppl_2010) AS crime_density_ppl_2010,
    ppl_2017, FORMAT('%3.4f',COUNTIF(year = 2017)/ppl_2017) AS crime_density_ppl_2017,
    FORMAT('%3.3f',COUNTIF(year = 2010)/ppl_2010 - COUNTIF(year = 2017)/ppl_2017) AS improvement
  FROM chicago_crime.crime C
  JOIN (
    SELECT community, number, ppl_2010, population AS ppl_2017
    FROM community_data.community_ppl A
    LEFT JOIN community_data.ppl_2017 B
    ON A.Num = B.number) ppl
  ON C.community_area = ppl.number
  GROUP BY 1,3,5
  ORDER BY 1) D
JOIN (
  SELECT community, RANK() OVER(ORDER BY arrest_2010 DESC) AS ArrestRank
  FROM community_data.arrest_by_community) E
ON D.community = E.community
ORDER BY improvement DESC

```

Chicago police have been working hard to reduce the crimes in the city. 6 out of 10 communities with the largest drop in crime rate were also in top arrest communities in 2010. However these communities are still among the worst performance ones.

### Do crimes happen more in wealthier communities or poorer ones?

Use the percent of children in poverty as an indication of community economics. Rank of 1 indicates wealthiest community and 77 indicates poorest.

```sql
SELECT B.community, rank, pvt_ch_perc,COUNTIF(primary_type = 'THEFT') AS theft,
  COUNTIF(primary_type != 'THEFT') as other_Type,
  COUNTIF(primary_type = 'THEFT')/COUNT(primary_type) AS theft_perc,
  COUNTIF(primary_type = 'HOMICIDE') AS homicide,
  crime_density_ppl_2017
FROM chicago_crime.data A
LEFT JOIN community_data.pvt_ch_by_community B
ON A.community_area = B.number
LEFT JOIN community_data.crime_density_ppl C
ON C.community = B.community
WHERE year >= 2015
GROUP BY 1,2,3,crime_density_ppl_2017
ORDER BY 2
```

In general, the crime rates per resident tends to be lower when the percent of children in poverty is lower in communities.

However, the percentage of theft is generally higher in wealthier communities, which makes sense because people are carrying more money or keep items with higher value in properties.

These communities do have much fewer homicides though, making them safer place to live.

