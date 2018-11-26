## W205 - Project 3 - Understanding User Behavior Project

### Assignment 10 - Setup Pipline - Part 1

A mobile app make API calls to web services when user interacts with it.  The API server handles the actual 
business process and logs events to kafka.

In this assignment, we setup the pipeline and track two events from a mobile game: buy a sword and join guild. 

#### Flask with Kafka and Spark

As we had already created a directory ~/w205/flask-with-kafka-and-spark/, just cd into it.  Copy the docker-compose.yml file 
to the current directory.


```
cd ~/w205/flask-with-kafka-and-spark/

cp ~/w205/course-content/10-Transforming-Streaming-Data/docker-compose.yml .
```

Spin up the cluster
```
docker-compose up -d
```

Create a topic
```
docker-compose exec kafka kafka-topics --create --topic events --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181
```
The following is displayed.
```
Created topic "events".
```

Make the Web App generate more informative events.  Copy and paste the following code into game_api_with_json_events.py

```python
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')


def log_to_kafka(topic, event):
    producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"


@app.route("/join_a_guild")
def join_a_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nJoined a Guild!\n"

    
```

Run it by executing the docker-compose exec mids command below.
```

docker-compose exec mids env FLASK_APP=/w205/flask-with-kafka-and-spark/game_api_with_json_events.py flask run --host 0.0.0.0
```
Displays
``
 * Serving Flask app "game_api_with_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```
In another terminal window, test it by generating events.
```
docker-compose exec mids curl http://localhost:5000/
```
Displays
```
This is the default response!
``
Test for "purchase_a_sword"
```
docker-compose exec mids curl http://localhost:5000/purchase_a_sword
```
Displays
```
Sword Purchased!
```

Test again for "join_a_guild"
```
docker-compose exec mids curl http://localhost:5000/join_a_guild
```
Displays
```
Joined a Guild!
```

Switch back to the original window and we can see the following.

```
127.0.0.1 - - [22/Nov/2018 01:41:11] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:41:37] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:41:49] "GET /join_a_guild HTTP/1.1" 200 -
```

Read from kafka
```
docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
```
Displays
```
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "join_guild"}
% Reached end of topic events [0] at offset 3: exiting
```

Copy the following code and create game_api_with_extended_json_events.py

```
#!/usr/bin/env python
import json
from kafka import KafkaProducer
from flask import Flask, request

app = Flask(__name__)
producer = KafkaProducer(bootstrap_servers='kafka:29092')


def log_to_kafka(topic, event):
    event.update(request.headers)
    producer.send(topic, json.dumps(event).encode())


@app.route("/")
def default_response():
    default_event = {'event_type': 'default'}
    log_to_kafka('events', default_event)
    return "\nThis is the default response!\n"


@app.route("/purchase_a_sword")
def purchase_a_sword():
    purchase_sword_event = {'event_type': 'purchase_sword'}
    log_to_kafka('events', purchase_sword_event)
    return "\nSword Purchased!\n"


@app.route("/join_a_guild")
def join_a_guild():
    join_guild_event = {'event_type': 'join_guild'}
    log_to_kafka('events', join_guild_event)
    return "\nJoined a Guild!\n"

```

Generate more informative events  

```
docker-compose exec mids env FLASK_APP=/w205/flask-with-kafka-and-spark/game_api_with_extended_json_events.py flask run --host 0.0.0.0
```
Displays
```
 * Serving Flask app "game_api_with_extended_json_events"
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```
In another window test it

```
docker-compose exec mids curl http://localhost:5000/
```
Displays
```
This is the default response!
```

```
docker-compose exec mids curl http://localhost:5000/
```
Displays
```
This is the default response!
```
More testing

```
docker-compose exec mids curl http://localhost:5000/purchase_a_sword

	* Sword Purchased!

docker-compose exec mids curl http://localhost:5000/purchase_a_sword

	* Sword Purchased!

docker-compose exec mids curl http://localhost:5000/purchase_a_sword

	* Sword Purchased!

docker-compose exec mids curl http://localhost:5000/join_a_guild

	* Joined a Guild!

docker-compose exec mids curl http://localhost:5000/purchase_a_sword

	* Sword Purchased!

docker-compose exec mids curl http://localhost:5000/join_a_guild

	* Joined a Guild!

docker-compose exec mids curl http://localhost:5000/purchase_a_sword

	*Sword Purchased!
```

Switch back to the original window and the following is displayed.

```
127.0.0.1 - - [22/Nov/2018 01:53:31] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:53:34] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:53:41] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:53:46] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:53:48] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:53:56] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:54:04] "GET /purchase_a_sword HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:54:08] "GET /join_a_guild HTTP/1.1" 200 -
127.0.0.1 - - [22/Nov/2018 01:54:15] "GET /purchase_a_sword HTTP/1.1" 200 -
```
Read from kafka

```
docker-compose exec mids kafkacat -C -b kafka:29092 -t events -o beginning -e
```

Displays
```
{"event_type": "default"}
{"event_type": "purchase_sword"}
{"event_type": "join_guild"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "default", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "join_guild", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
{"Host": "localhost:5000", "event_type": "purchase_sword", "Accept": "*/*", "User-Agent": "curl/7.47.0"}
% Reached end of topic events [0] at offset 12: exiting
```


Bring up Spark
```
docker-compose exec spark pyspark
```
We see

```
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.2.0
      /_/

Using Python version 3.6.1 (default, May 11 2017 13:09:58)
SparkSession available as 'spark'.
>>>
```
Read from kafka
```
raw_events = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","events").option("startingOffsets", "earliest").option("endingOffsets", "latest").load()
```
Check the type of rav_events
```
raw_events
```
Displays
```
DataFrame[key: binary, value: binary, topic: string, partition: int, offset: bigint, timestamp: timestamp, timestampType: int]
```

PrintSchema
```
 raw_events.printSchema()
```
Displays
```
root
 |-- key: binary (nullable = true)
 |-- value: binary (nullable = true)
 |-- topic: string (nullable = true)
 |-- partition: integer (nullable = true)
 |-- offset: long (nullable = true)
 |-- timestamp: timestamp (nullable = true)
 |-- timestampType: integer (nullable = true)
```
See the data
```
 raw_events.show()
```
Displays
```
+----+--------------------+------+---------+------+--------------------+-------------+
| key|               value| topic|partition|offset|           timestamp|timestampType|
+----+--------------------+------+---------+------+--------------------+-------------+
|null|[7B 22 65 76 65 6...|events|        0|     0|2018-11-22 01:41:...|            0|
|null|[7B 22 65 76 65 6...|events|        0|     1|2018-11-22 01:41:...|            0|
|null|[7B 22 65 76 65 6...|events|        0|     2|2018-11-22 01:41:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     3|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     4|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     5|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     6|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     7|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     8|2018-11-22 01:53:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|     9|2018-11-22 01:54:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|    10|2018-11-22 01:54:...|            0|
|null|[7B 22 48 6F 73 7...|events|        0|    11|2018-11-22 01:54:...|            0|
+----+--------------------+------+---------+------+--------------------+-------------+
```

Explore the events
```
events = raw_events.select(raw_events.value.cast('string'))
import json
extracted_events = events.rdd.map(lambda x: json.loads(x.value)).toDF()
```
Displays
```
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:58:50 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
/spark-2.2.0-bin-hadoop2.6/python/pyspark/sql/session.py:351: UserWarning: Using RDD of dict to inferSchema is deprecated. Use pyspark.sql.Row instead
  warnings.warn("Using RDD of dict to inferSchema is deprecated. "
```
```
extracted_events.show()
```
Displays
```
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
18/11/22 01:59:05 WARN CachedKafkaConsumer: CachedKafkaConsumer is not running in UninterruptibleThread. It may hang when CachedKafkaConsumer's methods are interrupted because of KAFKA-1894
+--------------+
|    event_type|
+--------------+
|       default|
|purchase_sword|
|    join_guild|
|       default|
|       default|
|purchase_sword|
|purchase_sword|
|purchase_sword|
|    join_guild|
|purchase_sword|
|    join_guild|
|purchase_sword|
+--------------+

```
Check the type of events
```
 events
```
```
DataFrame[value: string]
```
PrintSchema
```
events.printSchema()
```
Displays
```
root
 |-- value: string (nullable = true)
```
Exit pyspark and tear down the cluster
```
exit()

docker-compose down
```


