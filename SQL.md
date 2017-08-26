
# Data Overview and SQL 

This section contains basic stats about the dataset, SQL Queries and some additional explorations. 

### File Sizes


las-vegas_nevada.osm............216 MB

all_nodes.csv...................65.6 MB

all_ways.csv....................4.7 MB

all_users.csv...................20 KB

city_info.csv...................1 KB

node_tags.csv...................2.2 MB

way_nodes.csv...................27.6 MB

way_tags.csv....................9.2 MB



I used PostgreSQL to implement SQL tables.
Directly copied csv data to sql tables.

### Tables
The following shows the relations or sql tables formed: 

<img src='images\tables.jpg' align='left'>

### Number of Nodes and Ways

<img src='images\count.jpg' align='left'>

### Users

First we see the number of unique users:

<img src='images\unique_users.jpg' align='left'>

Next, we see which users contributed the most.

<img src='images\max_users.jpg' align='left'>

### Top 10 appearing amenities

<img src='images\amenities.jpg' align='left'>

### Top 10 appearing Postcodes

<img src='images\postcodes.jpg' align='left'>

### Top 3 Religions

<img src='images\religion.jpg' align='left'>

<img src='images\restaurantss.jpg' align='left'>

### Top 5 popular cuisines

<img src='images\restaurants.jpg' align='left'>

### Conclusion

After the review and exploration of the Las Vegas OSM data available it is easy to point out the following:
1. The Las Vegas area is still incomplete.
2. There's still a lot of scope to clean and make the data more consistent. 
3. 2 users contribute to major node data when compared to others. 


```python

```
