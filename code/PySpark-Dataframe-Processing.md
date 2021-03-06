## Step 1: Load Charlotte Sample Twitter Gnip Dataset

This step imports pySpark functions, reads in the Charlotte Gnip json file (Dec-Feb 2016 Geolocated Charlotte Tweets) and creates an RDD-file called tweets.

Last, the `printSchema()` function will print the schema so you can see the structure of the dataset.

```{python}
from pyspark.sql import SQLContext
sqlContext = SQLContext(sc)

# Create an RDD called tweets
tweets = sqlContext.read.json("/twitter/gnip/public/social/CharlotteProtest.json")

# Count the number of items in the RDD
tweets.count() 

# Print the schema for tweets
tweets.printSchema()
```

## Step 2: Basic GroupBy & Count Functions

Next, let's explore two fields to groupby and count the tweets.

First, we groupBy the `verb` field which corresponds to Tweets (post) and Retweets (share). Notice, this dataset only includes posts as the filtering rules for this dataset excluded Retweets.

Next, we groupBy the `geo.type` and `location.geo.type` fields to identify whether the geolocated Tweets are points, places (polygons) or no location (NULL).

```{python}
# Retweets (share) vs Original Content Posts (post)
tweets.groupBy("verb").count().show()

# Geolocated Points vs non-Geolocated Points
tweets.groupBy("location.geo.type","geo.type").count().show()
```

## Step 3: groupBy, filter and orderBy functions

We can also include the filter and orderBy functions too to limit or sort our data.

```{python}
from pyspark.sql.functions import col

# by top 20 locations
tweets.groupBy("actor.location.displayName")/
    .count()/
    .orderBy(col("count").desc())/
    .show()

# Tweets by users with more than 500k Followers Ordered by Tweet Time (postedTime)
tweets.filter(tweets['actor.followersCount'] > 500000)/
    .select("actor.preferredUsername","body","postedTime","actor.followersCount")/
    .orderBy("postedTime")/
    .show()
```

## Step 4: If-then Statements Using When

You can import in the `when` function that is used like a case when (if-else) statement.

```{python}
from pyspark.sql.functions import when

# Create new column called "geolocation" based on when statement
tweets = tweets.withColumn("geolocation", (when(col("geo.type") == "Point", "Point")\
                                           .when(col("location.geo.type") == "Polygon", "Polygon")\
                                           .otherwise("Place")))

tweets.groupBy("geolocation").count().show()
```

## Step 5: Registering the Dataframe as a Table

Alternatively to creating RDD's, you can create a Spark DataFrame by registering the dataframe as a table.

```{python}
tweets.registerTempTable('tweets')

df = sqlContext.sql("SELECT id, postedTime, body, actor.id FROM tweets")
```

## Step 6: Sampling and Exporting JSON file

This step will take a sample of your dataframe and allow you to save the dataframe as a smaller JSON file.

Please note that you will need to save the file to your own personal folder within the `/user/` directory.

```{python}
# sampling function: sample(withReplacement, fraction, seed=None)
df = df.sample(False, 0.01, 42)

df.coalesce(1).toJSON().saveAsTextFile("/user/rwesslen/savedjson/")
```

Currently, there's not an easy way to save CSV files. Spark 2.0 does offer the [spark-csv](https://github.com/databricks/spark-csv); however, this capability is not yet set up on SOPHI.

For more details, consider this [StackOverflow page](http://stackoverflow.com/questions/31385363/how-to-export-a-table-dataframe-in-pyspark-to-csv).
