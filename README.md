# Analysis of Airbnb properties in Australia

This time, I will explore the Airbnb dataset from 2021 obtained at [insideairbnb.com](https://insideairbnb.com) for Australia. I'll visualise all the listings on the map using a great Python library [Datashader](https://datashader.org), will use [Metabase](http://metabase.com) (an open-source BI tool) to answer questions like

- What neighbourhood is the most popular Airbnb destination across Australia
- 2021 Christmas availability details
- How much are hosts making from renting to tourists (compare that to long-term rentals)
- Which hosts are running a business with multiple listings and what is the average number of listings per host

The dataset has about 200k listings, 6.3M reviews, and 70M calendar bookings. The live dashboard is available [here](https://metabase.lenalytics.me/public/dashboard/35a5f488-022a-4f16-88fa-1c3260324f82).
    
# Setup & Tools

The data exploration setup is similar to the one I described in one of my previous posts "[Data behind parking tickets in New York City](https://dev.to/lenalytics/data-behind-parking-tickets-in-new-york-city-23ac)". I will use

- Docker compose to launch Postgres database and Metabase container to use SQL to query the data
- Python for scripting, [Datashader](http://datashader.org) to visualise Geodata on the map
- As previously, I'll deploy everything to AWS for demo purposes

I assume that you have some basic understanding of containers, Docker and Compose, but if you don't, check out my previous [post](https://dev.to/lenalytics/data-behind-parking-tickets-in-new-york-city-23ac) which explains everything in detail.

# Importing Data

To import data into a Postgres database, let's create the following tables:

- [`listing`](https://gist.github.com/lenalytics/3f2a4bab5965c0da41544ee9e9ead0f0)
- [`review`](https://gist.github.com/lenalytics/8d256f2ca19a0880c8d417c4edbc5c8d)
- [`calendar `](https://gist.github.com/lenalytics/d69157a31e889c08518ff0d391bd781c)
- `suburbs` - This is an additional table, not present in the original dataset. It contains the geodata of all Australian suburbs as per *Australian Bureau of Statistics*

The analysed dataset (although it has great insights) is not ready to use out of the box. Some of the records cannot be imported into the database, that's why I will create a simple Python ETL flow, the purpose of which is to

- load CSV files one by one into memory
- perform the data transformation part
- insert results into the database

Python script GIST that performs that ETL process is available [here](https://gist.github.com/lenalytics/c1225258665d49b1f6fe05ebce439840).

# Visualising with Datashader
This is the first time I used this tool for visualising geodata, and it turned out to be quite good. The reason for choosing it was that it can easily plot big datasets with millions of points of data.

The basic setup is described in the [official documentation](https://datashader.org/index.html), I just want to point out a few things

- Datashader library is rather quirky
- I couldn't make it run either locally, or by using Virtualenv
- Recommended setup via Anaconda had worked, but only after a few tries because every time Conda got stuck on *[solving environment](https://stackoverflow.com/questions/63734508/stuck-at-solving-environment-on-anaconda)* issue
- Try using low-res rendering instead of high-res, i.e. 800x500 will look better than 1900x1200

Let's first create a simple visualisation of all Airbnb listings across AU and NZ

```python
import colorcet
import datashader as ds
import pandas as pd

df = pd.read_sql_table('listing', conn)
cvs = ds.Canvas(plot_width=800, plot_height=500)
agg = cvs.points(df, x='longitude', y='latitude')
img = ds.tf.shade(agg, cmap=colorcet.fire, how='log')
```

![datashader au nz](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/airdata_au_nz.png)

Now, let's visualise all the major cities separately. To do this, I will create a postgres view for each city, using its bounding geo coordinates. For example, a view for Melbourne can be created like that:

```sql
CREATE VIEW listing_melbourne AS
    SELECT *
    FROM listing
    WHERE (longitude BETWEEN 144.2589302195 AND 145.4866523875)
    AND  (latitude BETWEEN -38.517433861 AND -37.5733096779);
```
Then, I can ask Datashader to plot the entire `listing_melbourne` table like that:
```python
df = pd.read_sql_table('listing_melbourne', conn)
cvs = ds.Canvas(plot_width=800, plot_height=500)
agg = cvs.points(df, x='longitude', y='latitude')
img = ds.tf.shade(agg, cmap=colorcet.fire, how='log')
```

**Melbourne**

![Melbourne](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/airdata_melb.png)

Similar to Melbourne, it's easy to create datashader plots for Sydney, Brisbane, Perth, Adelaide, etc.

# Data Exploration

Let's now turn to Metabase and build a few queries to address the questions at the beginning of that post.

**What states have the most Airbnb listings**

→ In Australia only

```sql
SELECT region_parent_name, COUNT(*) AS listing_count
FROM listing
WHERE region_parent_name IN ('New South Wales', 'Victoria','Queensland','Western Australia','South Australia','Tasmania','Australian Capital Territory', 'Northern Territory')
GROUP BY region_parent_name
ORDER BY listing_count DESC
```

[Link to live query](https://metabase.lenalytics.me/public/question/6b02aa38-3a3a-4d18-a683-556712f97277)
![count](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/airbnb_count_by_state_au.png)


→ In Australia and New Zealand

```sql
SELECT region_parent_name as region, COUNT(*) AS listing_count
FROM listing
WHERE region_parent_name IN ('New South Wales', 'Victoria','Queensland','Western Australia','South Australia','Tasmania','Australian Capital Territory', 'Northern Territory')
GROUP BY region_parent_name

UNION ALL

SELECT 'New Zealand' as region, COUNT(*) AS listing_count
FROM listing
WHERE region_parent_name NOT IN ('New South Wales', 'Victoria','Queensland','Western Australia','South Australia','Tasmania','Australian Capital Territory', 'Northern Territory')

ORDER BY listing_count DESC
```

[Link to live query](https://metabase.lenalytics.me/public/question/e8ce0440-1395-42c3-9d48-21544a474342)
![count](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/airbnb_count_by_state_au_nz.png)

**What neighbourhoods are the most popular Airbnb destinations in Australia?**

To answer this question, I can use one of the following columns in the listing table: `neighbourhood`, `host_neighbourhood`, `host_location`. The problem here is that a lot of these columns have NULL or empty values. For example: `SELECT COUNT(*) FROM "listing" where "neighbourhood" !=''` will only return 124,363 rows, or 64% of all the listings. Coalescing these three fields also won't work properly.

Instead, I will use geodata (lat, long) of the listings, and [shapefiles](https://www.abs.gov.au/statistics/standards/australian-statistical-geography-standard-asgs-edition-3/jul2021-jun2026/access-and-downloads/digital-boundary-files) from the *Australian Bureau of Statistics* to upload the suburb shapes into our database. There are several ways to upload **.shp** files into Postgres, but I recommend QGIS for that for its simplicity.

After uploading suburbs geodata into the new table `suburbs`, I will run the following query:

```sql
SELECT
  suburbs.sa3_name21 AS suburb_name,
  COUNT(listing.*) AS listing_count
FROM listing
JOIN suburbs
ON ST_Contains(ST_SetSRID(
           suburbs.geom::GEOMETRY,
           7844
         ), ST_SetSRID(ST_MakePoint(listing.longitude, listing.latitude), 7844))
GROUP BY suburb_name
ORDER BY listing_count DESC
```

which gives us suburbs with the highest number of Airbnb listings:

[Link to live query](https://metabase.lenalytics.me/public/question/6beeb96c-d0d3-426b-8d3f-7119c941ee84)
![suburbs](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/count_by_suburbs.png)

or as a table:

[Link to live query](https://metabase.lenalytics.me/public/question/8b7f5be0-8e2a-4061-a1d2-12fa2fdf615d)
<details>
  <summary>Click to expand the image</summary>
  
  ![suburbs_table](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/bad_suburbs_listing_count.png)
</details>



## Property Details

Distribution by the **number of bedrooms** doesn't look unexpected:

```sql
SELECT bedrooms, COUNT(*) AS listing_count
FROM airdata.listing
WHERE bedrooms>0 AND bedrooms<15
GROUP BY bedrooms
ORDER BY listing_count DESC
```

[Link to live query](https://metabase.lenalytics.me/public/question/006adeb5-926e-4637-abd1-b20ff9259540)
![bedrooms](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/count_by_bedrooms.png)

**How many rental properties are there per host?**

```sql
WITH count_by_host AS (
SELECT host_id, COUNT(*) AS listing_count
FROM airdata.listing
GROUP BY host_id
)
SELECT listing_count, COUNT(count_by_host.host_id) as host_count
FROM count_by_host
GROUP BY listing_count
ORDER BY listing_count DESC
```

[Link to live query](https://metabase.lenalytics.me/public/question/6cfeb900-7b10-4865-bd7a-2853931dde34)
![bedrooms](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/hosts_count.png)

Some additional interesting findings:

- There are **117907** different hosts in Australia in total
- There are **5** hosts in Australia with # of listings > **300**
- There are **39** hosts in Australia with # of listings > **100**
- There are **1064** hosts in Australia with # of listings > **10**
- There are **5599** hosts in Australia with # of listings > **3**


## Christmas 2021 Availability

One of the most interesting questions to address is the number of listings still available for Xmas 2021 period, per particular destination, i.e. what percentage of, say, Bondi Beach properties is available. 

To find the answer, we need to use our `calendar` table, which has an enormous 70M records. The queries will be running for some time, so it's a good thing to increase statement timeout in the Postgres settings.

So, for example for Bondi Beach (`id=372`) we have:

```sql
WITH suburb_listings AS (
SELECT lst.id as lstid FROM "airdata"."listing" AS lst
JOIN "airdata"."suburbs" AS subs
ON ST_Contains(ST_SetSRID(
           subs.geom::GEOMETRY,
           7844
         ), ST_SetSRID(ST_MakePoint(lst.longitude, lst.latitude), 7844))
WHERE subs.id = 372
)
SELECT DISTINCT(cal.listing_id) FROM "airdata"."calendar" AS cal
WHERE cal.listing_id IN (SELECT lstid FROM suburb_listings)
AND (cal.date BETWEEN '2021-12-20 00:00:00+00' AND '2021-12-28 00:00:00+00')
AND cal.available=TRUE
```

The result is `347` listings. So out of `2350` total listings in Bondi Beach area, only 347 (< 15%) have *some* availability during the Christmas period. Let's try to visualise it as a chart. In order to do that, I will iterate the above query over all suburbs in the `suburbs` table, and visualise like that:

[Link to live query](https://metabase.lenalytics.me/public/question/5568a336-ed94-4e41-a282-75ba9e405c48)
![bedrooms](https://d1ydrm1s5noqxj.cloudfront.net/airbnb/xmas_availability.png)


Keep in mind, the above is based on the booking data as of April 2021.

# Summary

In this article, we've done the following things

- Cleaned and uploaded the dataset from [insideairbnb.com](https://insideairbnb.com) to a Postgres database using Python
- Visualised listings geodata using Datashader Python library
- Explored the dataset in details with Metabase (an open-source BI tool)
- Created and published a [live dashboard](https://metabase.lenalytics.me/public/dashboard/35a5f488-022a-4f16-88fa-1c3260324f82) with all the above results with the help of AWS.

Thank you!
