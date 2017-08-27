
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

![](images/tables.jpg?raw=true)

### Number of Nodes and Ways


![](images/count.jpg?raw=true)

### Users

First we see the number of unique users:


![](images/unique_users.jpg?raw=true)

Next, we see which users contributed the most.


![](images/max_users.jpg?raw=true)

### Top 10 appearing amenities


![](images/amenities.jpg?raw=true)

### Top 10 appearing Postcodes


![](images/postcodes.jpg?raw=true)

### Top 3 Religions


![](images/religion.jpg?raw=true)

<img src='images\restaurantss.jpg' align='left'>

### Top 5 popular cuisines


![](images/restaurants.jpg?raw=true)

### Conclusion

After the review and exploration of the Las Vegas OSM data available it is easy to point out the following:
1. The Las Vegas area is still incomplete.
2. There's still a lot of scope to clean and make the data more consistent. 
3. 2 users contribute to major node data when compared to others. 


```python

```
