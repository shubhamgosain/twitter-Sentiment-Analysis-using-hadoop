# twitter-Sentiment-Analysis-using-hadoop
A Project where one can fetch and read tweets and show the analysis like who is most influential and also see what people of different countries think about a current trending topic and show their polarity


**PREREQUISITES**
=================

	* Download JSON Serde at:
	* http://files.cloudera.com/samples/hive-serdes-1.0-SNAPSHOT.jar
	* and to renominate it as hive-serdes-1.0.jar

- Execute the following commands on terminal:
	+ hive> add jar hive-serdes-1.0.jar;

**SENTIMENT ANALYSIS**
======================

Create the tweets_raw table containing the records as received from Twitter
---------------------------------------------------------------------------
```
CREATE EXTERNAL TABLE tweets_raw (
   id BIGINT,
   created_at STRING,
   source STRING,
   favorited BOOLEAN,
   retweet_count INT,
   retweeted_status STRUCT<
      text:STRING,
      user:STRUCT<screen_name:STRING,name:STRING>>,
   entities STRUCT<
      urls:ARRAY<STRUCT<expanded_url:STRING>>,
      user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
      hashtags:ARRAY<STRUCT<text:STRING>>>,
   text STRING,
   user STRUCT<
      screen_name:STRING,
      name:STRING,
      friends_count:INT,
      followers_count:INT,
      statuses_count:INT,
      verified:BOOLEAN,
      utc_offset:STRING, -- was INT but nulls are strings
      time_zone:STRING>,
   in_reply_to_screen_name STRING,
   year int,
   month int,
   day int,
   hour int
)
ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe '
LOCATION '/user/YOURUSER/upload/upload/data/tweets_raw'
;
```

Create sentiment dictionary
---------------------------
```
CREATE EXTERNAL TABLE dictionary (
    type string,
    length int,
    word string,
    pos string,
    stemmed string,
    polarity string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/YOURUSER/upload/upload/data/dictionary';

CREATE EXTERNAL TABLE time_zone_map (
    time_zone string,
    country string,
    notes string
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/YOURUSER/upload/upload/data/time_zone_map';
```


Clean up tweets
---------------
```
CREATE VIEW tweets_simple AS
SELECT
  id,
  cast ( from_unixtime( unix_timestamp(concat( '2013 ', substring(created_at,5,15)), 'yyyy MMM dd hh:mm:ss')) as timestamp) ts,
  text,
  user.time_zone
FROM tweets_raw
;

CREATE VIEW tweets_clean AS
SELECT
  id,
  ts,
  text,
  m.country
 FROM tweets_simple t LEFT OUTER JOIN time_zone_map m ON t.time_zone = m.time_zone;
```


Compute sentiment
-----------------

- Explode each tweet text in array of words.
	* Example: 330166074362433536 ["waiting","for","iron","man","3","to","start"]
```

create view l1 as select id, words from tweets_raw lateral view explode(sentences(lower(text))) dummy as words;
- Explode each tweet -> word on multiple raws

	* 330166074362433536 waiting
	* 330166074362433536 for
	* 330166074362433536 iron
	* 330166074362433536 man
	* 330166074362433536 3
	* 330166074362433536 to
	* 330166074362433536 start


create view l2 as select id, word from l1 lateral view explode( words ) dummy as word ;


- Used the dictionary file to score the sentiment of each Tweet by the number of positive words compared to the number of negative
words, and then assigned a positive, negative, or neutral sentiment value to each Tweet.


create view l3 as select
    id,
    l2.word,
    case d.polarity
      when  'negative' then -1
      when 'positive' then 1
      else 0 end as polarity
 from l2 left outer join dictionary d on l2.word = d.word;

 create table tweets_sentiment stored as orc as select
  id,
  case
    when sum( polarity ) > 0 then 'positive'
    when sum( polarity ) < 0 then 'negative'
    else 'neutral' end as sentiment
 from l3 group by id;


- Put everything back together and re-number sentiment



CREATE TABLE tweetsbi
STORED AS ORC
AS
SELECT
  t.*,
  case s.sentiment
    when 'positive' then 2
    when 'neutral' then 1
    when 'negative' then 0
  end as sentiment
FROM tweets_clean t LEFT OUTER JOIN tweets_sentiment s on t.id = s.id;
```





**REFERENCES**
==============
- http://hortonworks.com/hadoop-tutorial/how-to-refine-and-visualize-sentiment-data/
- http://bigdatabloggin.blogspot.it/2012/08/trending-topics-in-hive-ngrams.html
- https://cwiki.apache.org/confluence/display/Hive/StatisticsAndDataMining

