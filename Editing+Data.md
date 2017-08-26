
# OpenStreetMap Data Case Study

### Map Area

Las Vegas, Nevada

This map is of Vegas which I crave to visit someday in my life.

### Problems Encountered in the Map:

After initially downloading a small sample size of the Las Vegas area and running it against a provisional data.py file, I noticed five main problems with the data:

1. Inconsistent Postal Codes
2. Useless info in Changeset
3. Splitting Timestamp
4. 'FIXME' tags
5. Name tags split


```python
import xml.etree.cElementTree as ET
import pprint 
from collections import defaultdict
import re
import pandas as pd
```


```python
filename='las-vegas_nevada.osm'
sample_file='sample_las-vegas.osm'
```

## Initially, counting tag names. 


```python
def count_tags(filename):
    tags={}
    for event,t in ET.iterparse(filename):
        if t.tag in tags.keys():
            tags[t.tag]+=1
        else:
            tags[t.tag]=1
    return tags
```


```python
count_tags(sample_file)
```




    {'member': 2,
     'nd': 2815,
     'node': 2629,
     'osm': 1,
     'relation': 1,
     'tag': 1277,
     'way': 282}



## In this project we clean data associated with nodes and ways, leaving out relations. 


```python
street_type_re = re.compile(r'\b\S+\.?$', re.IGNORECASE)
expected = []
PROBLEMCHARS = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')
```


```python
def get_tags(elem):
    tags=[]
    for tag in elem.findall('tag'):
        if PROBLEMCHARS.match(tag.attrib['k']):
            continue
        else:
            elem_tags={}
            elem_tags['id']=elem.attrib['id']
            elem_tags['value']=tag.attrib['v']
            try:
                split_key=tag.attrib['k'].split(':',1)
                elem_tags['type']=split_key[0]
                elem_tags['key']=split_key[1]
            except:
                elem_tags['key']=tag.attrib['k']
                elem_tags['type']='regular'
            tags.append(elem_tags)
    return tags
    
def nodes(filename):
    with open (filename) as f:
        all_nodes = []
        node_tags =[]
        for event, elem in ET.iterparse(filename, events=("start",)):
            if elem.tag=='node':
                node={}
                for att,values in elem.items():
                    node[att]=values
                all_nodes.append(node)
                elem_tags=get_tags(elem)
                node_tags.extend(elem_tags)            
    return pd.DataFrame(all_nodes),pd.DataFrame(node_tags)

def ways(filename):
    with open(filename) as f:
        all_ways = []
        way_nodes = []
        way_tags = []
        for event, elem in ET.iterparse(filename, events=("start",)):
            if elem.tag == 'way':
                way_attribs={}
                for att,values in elem.items():
                    way_attribs[att]=values
                all_ways.append(way_attribs)
                for i,nodes in enumerate(elem.findall('nd')):
                    way_n={}
                    way_n['id']=elem.attrib['id']
                    way_n['node_id']=nodes.attrib['ref']
                    way_n['position']=i
                    way_nodes.append(way_n)
                tags=get_tags(elem)
                way_tags.extend(tags)
    return pd.DataFrame(all_ways),pd.DataFrame(way_nodes),pd.DataFrame(way_tags)
                
                    
```


```python
all_nodes,node_tags=nodes(filename)
```


```python
all_ways, way_nodes, way_tags=ways(filename)
```

## I wish to edit all my dataframes into a form such that I can export a similar looking table to SQL.

## 1. all_nodes


```python
all_nodes.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>changeset</th>
      <th>id</th>
      <th>lat</th>
      <th>lon</th>
      <th>timestamp</th>
      <th>uid</th>
      <th>user</th>
      <th>version</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>45401766</td>
      <td>31551114</td>
      <td>36.1662859</td>
      <td>-115.149225</td>
      <td>2017-01-23T14:55:12Z</td>
      <td>25975</td>
      <td>AmZaf</td>
      <td>10</td>
    </tr>
    <tr>
      <th>1</th>
      <td>15770302</td>
      <td>31551289</td>
      <td>36.0685037</td>
      <td>-115.2037262</td>
      <td>2013-04-18T07:52:34Z</td>
      <td>1282393</td>
      <td>Reno Editor</td>
      <td>6</td>
    </tr>
    <tr>
      <th>2</th>
      <td>7129896</td>
      <td>31551290</td>
      <td>36.0677428</td>
      <td>-115.2105885</td>
      <td>2011-01-30T03:51:34Z</td>
      <td>398919</td>
      <td>gMitchellD</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>7129896</td>
      <td>31551291</td>
      <td>36.0675724</td>
      <td>-115.2143331</td>
      <td>2011-01-30T03:51:34Z</td>
      <td>398919</td>
      <td>gMitchellD</td>
      <td>3</td>
    </tr>
    <tr>
      <th>4</th>
      <td>7129896</td>
      <td>31551345</td>
      <td>36.0672624</td>
      <td>-115.2149755</td>
      <td>2011-01-30T03:51:34Z</td>
      <td>398919</td>
      <td>gMitchellD</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
</div>



## Ignoring Changeset
Changeset does not provide us with any relevant info that is worth looking into, thus I will drop the column altogether. 
## Converting timestamp
Dropping 'Timestamp' to two columns as 'Time' and 'Date'


```python
def func_for_date(row):
    return row['timestamp'].split('T')[0]
def func_for_time(row):
    return row['timestamp'].split('T',1)[1].strip('Z')
```


```python
def edit_basic(df):
    del df['changeset']
    df['date']=df.apply(func_for_date,axis=1)
    df['time']=df.apply(func_for_time,axis=1)
    del df['timestamp']
    return df
```


```python
all_nodes=edit_basic(all_nodes)
```

## Users can have a different SQL table identified uniquely by uid. 


```python
def get_users(df):
    """a=set()
    for i in range(len(df)):
        b=(df.iloc[i]['uid'],df.iloc[i]['user'])
        a.add(b)"""
    a=df[['user','uid']]
    a=a.drop_duplicates()
    return a
```


```python
all_users=get_users(all_nodes)
#a=pd.DataFrame(list(all_users),columns=['uid','user'])
del(all_nodes['user'])
```


```python
all_users.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user</th>
      <th>uid</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AmZaf</td>
      <td>25975</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Reno Editor</td>
      <td>1282393</td>
    </tr>
    <tr>
      <th>2</th>
      <td>gMitchellD</td>
      <td>398919</td>
    </tr>
    <tr>
      <th>6</th>
      <td>NE2</td>
      <td>207745</td>
    </tr>
    <tr>
      <th>7</th>
      <td>woodpeck_repair</td>
      <td>145231</td>
    </tr>
  </tbody>
</table>
</div>




```python
all_nodes.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>lat</th>
      <th>lon</th>
      <th>uid</th>
      <th>version</th>
      <th>date</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>31551114</td>
      <td>36.1662859</td>
      <td>-115.149225</td>
      <td>25975</td>
      <td>10</td>
      <td>2017-01-23</td>
      <td>14:55:12</td>
    </tr>
    <tr>
      <th>1</th>
      <td>31551289</td>
      <td>36.0685037</td>
      <td>-115.2037262</td>
      <td>1282393</td>
      <td>6</td>
      <td>2013-04-18</td>
      <td>07:52:34</td>
    </tr>
    <tr>
      <th>2</th>
      <td>31551290</td>
      <td>36.0677428</td>
      <td>-115.2105885</td>
      <td>398919</td>
      <td>2</td>
      <td>2011-01-30</td>
      <td>03:51:34</td>
    </tr>
    <tr>
      <th>3</th>
      <td>31551291</td>
      <td>36.0675724</td>
      <td>-115.2143331</td>
      <td>398919</td>
      <td>3</td>
      <td>2011-01-30</td>
      <td>03:51:34</td>
    </tr>
    <tr>
      <th>4</th>
      <td>31551345</td>
      <td>36.0672624</td>
      <td>-115.2149755</td>
      <td>398919</td>
      <td>3</td>
      <td>2011-01-30</td>
      <td>03:51:34</td>
    </tr>
  </tbody>
</table>
</div>



## 2. node_tags


```python
node_tags.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>key</th>
      <th>type</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>31551114</td>
      <td>is_in</td>
      <td>regular</td>
      <td>Nevada</td>
    </tr>
    <tr>
      <th>1</th>
      <td>31551114</td>
      <td>continent</td>
      <td>is_in</td>
      <td>North America</td>
    </tr>
    <tr>
      <th>2</th>
      <td>31551114</td>
      <td>country</td>
      <td>is_in</td>
      <td>USA</td>
    </tr>
    <tr>
      <th>3</th>
      <td>31551114</td>
      <td>name</td>
      <td>regular</td>
      <td>Las Vegas</td>
    </tr>
    <tr>
      <th>4</th>
      <td>31551114</td>
      <td>ar</td>
      <td>name</td>
      <td>لاس فيغاس</td>
    </tr>
  </tbody>
</table>
</div>



### We, see that there are some rows with info regarding Las Vegas, the city as a whole. We make a different df for these values. 
It quite noticeable that the first three rows are inconsistent or aren't uniform. We sort that out as well.



```python
node_tags.iloc[0].type='is_in'
node_tags.iloc[0].key='state'
city_info=node_tags.loc[node_tags.id=='31551114']
del city_info['id']
```


```python
city_info.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>key</th>
      <th>type</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>state</td>
      <td>is_in</td>
      <td>Nevada</td>
    </tr>
    <tr>
      <th>1</th>
      <td>continent</td>
      <td>is_in</td>
      <td>North America</td>
    </tr>
    <tr>
      <th>2</th>
      <td>country</td>
      <td>is_in</td>
      <td>USA</td>
    </tr>
    <tr>
      <th>3</th>
      <td>name</td>
      <td>regular</td>
      <td>Las Vegas</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ar</td>
      <td>name</td>
      <td>لاس فيغاس</td>
    </tr>
  </tbody>
</table>
</div>



## Editing Postal Codes

a. We see that some postal codes are rather specific in nature. For these we just take in the first five numbers. 
b. Others start with NV for Nevada. 
c. We skip the rest.


```python
node_tags.loc[(node_tags.key=='postcode') & (node_tags.value.str.len()!=5)].head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>key</th>
      <th>type</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>26604</th>
      <td>993789695</td>
      <td>postcode</td>
      <td>addr</td>
      <td>89119-1001</td>
    </tr>
    <tr>
      <th>33080</th>
      <td>1700469005</td>
      <td>postcode</td>
      <td>addr</td>
      <td>89109-1907</td>
    </tr>
    <tr>
      <th>33197</th>
      <td>1721875458</td>
      <td>postcode</td>
      <td>addr</td>
      <td>8929</td>
    </tr>
    <tr>
      <th>48425</th>
      <td>2540047619</td>
      <td>postcode</td>
      <td>addr</td>
      <td>NV 89117</td>
    </tr>
    <tr>
      <th>48547</th>
      <td>2572969955</td>
      <td>postcode</td>
      <td>addr</td>
      <td>NV 89123</td>
    </tr>
  </tbody>
</table>
</div>




```python
def edit_postcode(p):
    if len(p)>5:
        if p[5]=='-':
            return p[:5]
        elif p[0]=='N':
            return p[3:]
    return p
```


```python
def edit_pcodes(df):
    for i in range(len(df)):
        if df.iloc[i]['key']=='postcode':
            df.iloc[i]['value']=edit_postcode(df.iloc[i]['value'])
    return df
```


```python
node_tags=edit_pcodes(node_tags)
```

## 3. all_ways


```python
all_ways.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>changeset</th>
      <th>id</th>
      <th>timestamp</th>
      <th>uid</th>
      <th>user</th>
      <th>version</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>10364623</td>
      <td>4879375</td>
      <td>2012-01-11T21:28:27Z</td>
      <td>92286</td>
      <td>Paul Johnson</td>
      <td>8</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11581490</td>
      <td>7149377</td>
      <td>2012-05-12T22:59:41Z</td>
      <td>121241</td>
      <td>zephyr</td>
      <td>7</td>
    </tr>
    <tr>
      <th>2</th>
      <td>49171267</td>
      <td>14278330</td>
      <td>2017-06-01T17:05:13Z</td>
      <td>1330847</td>
      <td>TheDutchMan13</td>
      <td>3</td>
    </tr>
    <tr>
      <th>3</th>
      <td>49171267</td>
      <td>14278332</td>
      <td>2017-06-01T17:05:13Z</td>
      <td>1330847</td>
      <td>TheDutchMan13</td>
      <td>3</td>
    </tr>
    <tr>
      <th>4</th>
      <td>47352899</td>
      <td>14278334</td>
      <td>2017-04-01T12:01:54Z</td>
      <td>360392</td>
      <td>maxerickson</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
</div>



### Editing the above Dataframe same as I did with all_nodes.


```python
all_ways=edit_basic(all_ways)
all_ways.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>uid</th>
      <th>user</th>
      <th>version</th>
      <th>date</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>4879375</td>
      <td>92286</td>
      <td>Paul Johnson</td>
      <td>8</td>
      <td>2012-01-11</td>
      <td>21:28:27</td>
    </tr>
    <tr>
      <th>1</th>
      <td>7149377</td>
      <td>121241</td>
      <td>zephyr</td>
      <td>7</td>
      <td>2012-05-12</td>
      <td>22:59:41</td>
    </tr>
    <tr>
      <th>2</th>
      <td>14278330</td>
      <td>1330847</td>
      <td>TheDutchMan13</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
    </tr>
    <tr>
      <th>3</th>
      <td>14278332</td>
      <td>1330847</td>
      <td>TheDutchMan13</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
    </tr>
    <tr>
      <th>4</th>
      <td>14278334</td>
      <td>360392</td>
      <td>maxerickson</td>
      <td>3</td>
      <td>2017-04-01</td>
      <td>12:01:54</td>
    </tr>
  </tbody>
</table>
</div>




```python
all_users=pd.concat([all_users,get_users(all_ways)])
del(all_ways['user'])
```


```python
all_users=all_users.drop_duplicates()
```


```python
all_users.describe()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>user</th>
      <th>uid</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>1053</td>
      <td>1053</td>
    </tr>
    <tr>
      <th>unique</th>
      <td>1053</td>
      <td>1052</td>
    </tr>
    <tr>
      <th>top</th>
      <td>understan</td>
      <td>605155</td>
    </tr>
    <tr>
      <th>freq</th>
      <td>1</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>




```python
all_ways.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>uid</th>
      <th>version</th>
      <th>date</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>4879375</td>
      <td>92286</td>
      <td>8</td>
      <td>2012-01-11</td>
      <td>21:28:27</td>
    </tr>
    <tr>
      <th>1</th>
      <td>7149377</td>
      <td>121241</td>
      <td>7</td>
      <td>2012-05-12</td>
      <td>22:59:41</td>
    </tr>
    <tr>
      <th>2</th>
      <td>14278330</td>
      <td>1330847</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
    </tr>
    <tr>
      <th>3</th>
      <td>14278332</td>
      <td>1330847</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
    </tr>
    <tr>
      <th>4</th>
      <td>14278334</td>
      <td>360392</td>
      <td>3</td>
      <td>2017-04-01</td>
      <td>12:01:54</td>
    </tr>
  </tbody>
</table>
</div>



## 4. way_tags


```python
way_tags.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>key</th>
      <th>type</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>4879375</td>
      <td>ref</td>
      <td>regular</td>
      <td>CR 215</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4879375</td>
      <td>name</td>
      <td>regular</td>
      <td>Bruce Woodbury Beltway</td>
    </tr>
    <tr>
      <th>2</th>
      <td>4879375</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>check lanes; are bikes allowed?</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4879375</td>
      <td>lanes</td>
      <td>regular</td>
      <td>2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4879375</td>
      <td>oneway</td>
      <td>regular</td>
      <td>yes</td>
    </tr>
  </tbody>
</table>
</div>




```python
#way_tags postcodes
way_tags=edit_pcodes(way_tags)
```


```python
way_tags.loc[way_tags.key=='FIXME']
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>key</th>
      <th>type</th>
      <th>value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2</th>
      <td>4879375</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>check lanes; are bikes allowed?</td>
    </tr>
    <tr>
      <th>466</th>
      <td>14279106</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>473</th>
      <td>14279108</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>480</th>
      <td>14279112</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>588</th>
      <td>14279371</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>623</th>
      <td>14279500</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>629</th>
      <td>14279505</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>634</th>
      <td>14279509</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>668</th>
      <td>14279670</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>1084</th>
      <td>14281502</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>1143</th>
      <td>14281802</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>1185</th>
      <td>14281932</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>1820</th>
      <td>14283629</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>check name</td>
    </tr>
    <tr>
      <th>2045</th>
      <td>14284230</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>3950</th>
      <td>14289421</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>4198</th>
      <td>14290141</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>4418</th>
      <td>14290505</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>4775</th>
      <td>14291010</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>4842</th>
      <td>14291048</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>5163</th>
      <td>14291337</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>5331</th>
      <td>14291434</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>5379</th>
      <td>14291479</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>5446</th>
      <td>14291513</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>5670</th>
      <td>14291675</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>40804</th>
      <td>14299009</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>80460</th>
      <td>14307445</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>82799</th>
      <td>14307903</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>104955</th>
      <td>14312366</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>106855</th>
      <td>14312794</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>are bikes allowed?</td>
    </tr>
    <tr>
      <th>155950</th>
      <td>14322482</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Area Needs Checking</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>376024</th>
      <td>424918381</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>376035</th>
      <td>424918383</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>376040</th>
      <td>424918384</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>376045</th>
      <td>424918385</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>376870</th>
      <td>430920613</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>376878</th>
      <td>430920614</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>377439</th>
      <td>435072207</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>377449</th>
      <td>435072208</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>377458</th>
      <td>435072209</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>377467</th>
      <td>435072210</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>reconfigured?</td>
    </tr>
    <tr>
      <th>380808</th>
      <td>457819755</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>verfiy bicycle=yes</td>
    </tr>
    <tr>
      <th>380900</th>
      <td>457824207</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>dual carriageway</td>
    </tr>
    <tr>
      <th>380975</th>
      <td>457839022</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>dual carriageway</td>
    </tr>
    <tr>
      <th>382531</th>
      <td>460475610</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>check lanes</td>
    </tr>
    <tr>
      <th>400845</th>
      <td>488751533</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400866</th>
      <td>488751537</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400877</th>
      <td>488751538</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400888</th>
      <td>488751539</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400909</th>
      <td>488751541</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400923</th>
      <td>488751543</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>400934</th>
      <td>488751544</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401033</th>
      <td>488755095</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401087</th>
      <td>488755103</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401092</th>
      <td>488756219</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401105</th>
      <td>488756224</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401110</th>
      <td>488756808</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401114</th>
      <td>488756809</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401273</th>
      <td>488793086</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>Divided highway</td>
    </tr>
    <tr>
      <th>401837</th>
      <td>489162618</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>dual carriageway</td>
    </tr>
    <tr>
      <th>401855</th>
      <td>489162625</td>
      <td>FIXME</td>
      <td>regular</td>
      <td>dual carriageway</td>
    </tr>
  </tbody>
</table>
<p>368 rows × 4 columns</p>
</div>



In the above example we see, for 'FIXME' tag, data has been corrected already. For places, where it hasn't been, we have insufficient data to fix it. Thus, we remove all the 'FIXME' tags, in order to get a cleaner dataset.


```python
way_tags=way_tags[way_tags.key!='FIXME']
```

Further, the name is split and stored in:

['name_1','name_base','name_base_1','name_direction_prefix','name_full']

This is not required as the final name is anyway stored in 'name'. Thus, it is redundant data and removing the tags with above mentioned keys.

By doing this we reduce the data from 2708 rows to 892 rows.



```python
#Edit way_tags to remove random values of names
way_omit_tags=pd.Series(['name_1','name_base','name_base_1','name_direction_prefix','name_full','name_ty','name_type_1'])
way_tags=way_tags[~way_tags.key.isin(way_omit_tags)]
```

We kept 'name_type' as it holds relevant data in terms of the type 'way'.


```python
map_name={'Aly':'Alley',
 'Ave':'Avenue',
 'Ave:Rd':'Drive',
 'Blvd':'Boulevard',
 'Br':'Brook',
 'Cir':'Circle',
 'Ct':'Court',
 'Ctr':'Drive',
 'Cv':'Cove',
 'Dr':'Drive',
 'Dr.':'Drive',
 'Dr:Rd':'Drive',
 'Dr; Dr; Dr; Rd':'Divided Highway',
 'Dr;Dr;Dr;Rd':'Divided Highway',
 'Hwy':'Highway',
 'Ln':'Lane',
 'Ln; Pky':'Parkway',
 'Loop':'Loop',
 'Mal':'Mall',
 'Pass':'Pass',
 'Path':'Path',
 'Pkwy':'Parkway',
 'Pky':'Parkway',
 'Pl':'Place',
 'Plz':'Plaza',
 'Rd':'Road',
 'Rd:St':'Street',
 'Rd; Blvd':'Boluevard',
 'Rd; Dr':'Drive',
 'Rd; Dr; Rd; St':'Road',
 'Rd; Rd; Dr; Rd':'Road',
 'Rte':'Route',
 'Spur':'Spur',
 'Sq':'Square',
 'St':'Street',
 'St:Trl':"Trail",
 'Ter':'Terrace',
 'Trl':'Trail',
 'Way':'Way',
 'Way:Way; Rd; Way':'Way',
 'Way; Rd; Way':'Way',
 'Xing':'Xing'}
```


```python
way_tags['value'] = way_tags.loc[way_tags.key=='name_type'].value.map(map_name)
```

## 5. way_nodes


```python
way_nodes.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>node_id</th>
      <th>position</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>4879375</td>
      <td>302158005</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>4879375</td>
      <td>1127286209</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>4879375</td>
      <td>302157999</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4879375</td>
      <td>1127286264</td>
      <td>3</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4879375</td>
      <td>302158219</td>
      <td>4</td>
    </tr>
  </tbody>
</table>
</div>



### We add the number of way  'nodes' to 'way_nodes' table. 


```python
way_node_count = way_nodes.groupby('id').count().sort_values('position', ascending=False).reset_index()[["id",'position']]
way_node_count.columns=['id','node_count']
def add_count(row):
    return row.node_count
all_ways['node_count']=way_node_count.apply(add_count,axis=1)
del(way_node_count)
```


```python
all_ways.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>uid</th>
      <th>version</th>
      <th>date</th>
      <th>time</th>
      <th>node_count</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>4879375</td>
      <td>92286</td>
      <td>8</td>
      <td>2012-01-11</td>
      <td>21:28:27</td>
      <td>670.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>7149377</td>
      <td>121241</td>
      <td>7</td>
      <td>2012-05-12</td>
      <td>22:59:41</td>
      <td>653.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>14278330</td>
      <td>1330847</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
      <td>646.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>14278332</td>
      <td>1330847</td>
      <td>3</td>
      <td>2017-06-01</td>
      <td>17:05:13</td>
      <td>643.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>14278334</td>
      <td>360392</td>
      <td>3</td>
      <td>2017-04-01</td>
      <td>12:01:54</td>
      <td>643.0</td>
    </tr>
  </tbody>
</table>
</div>



## Finally, getting a data frame for users so as it can be further converted into an SQL table.

## The final csv files are as:


```python
#nodes
all_nodes.to_csv('all_nodes.csv',index=False)
node_tags.to_csv('node_tags.csv',index=False)
#ways
all_ways.to_csv('all_ways.csv',index=False)
way_nodes.to_csv('way_nodes.csv',index=False)
way_tags.to_csv('way_tags.csv',index=False)
#additional
city_info.to_csv('city_info.csv',index=False)
all_users.to_csv('all_users.csv',index=False)
```
