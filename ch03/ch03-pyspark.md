
# Chapter 3

## Referrence
- ALS 알고리즘 설명 : https://ggoals.github.io/Spark_ALS_Algorithm_tuning/
- Towards data science : https://towardsdatascience.com/build-recommendation-system-with-pyspark-using-alternating-least-squares-als-matrix-factorisation-ebe1ad2e7679
- Spark tutorial : https://spark.apache.org/docs/latest/ml-collaborative-filtering.html

## Download data


```python
!wget https://storage.googleapis.com/aas-data-sets/profiledata_06-May-2005.tar.gz
!tar xvf ./data/profiledata_06-May-2005.tar.gz
```


## Spark session


```python
import gc
import logging
import subprocess
from datetime import datetime
from pathlib import Path

import pyspark.sql.functions as F
import pyspark.sql.types as T
from pyspark.sql import Row
from pyspark.sql import SparkSession
from pyspark.sql.window import Window
# from pytz import timezone
# from pytz import utc
```


```python
EXECUTOR_MEMORY = "2g"
EXECUTOR_CORES = 2
EXECUTORE_INSTANCES = 3
DRIVER_MEMORY = "1g"
DRIVER_MAX_RESULT_SIZE = "1g"
```


```python
spark = (
    SparkSession.builder.appName(f"Advanced analytics with SPARK - Chapter 3")
    .config("spark.executor.memory", EXECUTOR_MEMORY)
    .config("spark.executor.cores", EXECUTOR_CORES)
    .config("spark.executor.instances", EXECUTORE_INSTANCES)
    .config("spark.driver.memory", DRIVER_MEMORY)
    .config("spark.driver.maxResultSize", DRIVER_MAX_RESULT_SIZE)
    .config("spark.kryoserializer.buffer.max", "1024m")
#     .config("spark.sql.warehouse.dir", "/user/bigdata/members/shyeon/advanced-spark/data")
    .enableHiveSupport()
    .getOrCreate()
)

spark.sparkContext.getConf().getAll()
```




    [('spark.executor.memory', '2g'),
     ('spark.executor.instances', '3'),
     ('spark.driver.host', '0b63c6cfbaf6'),
     ('spark.driver.extraJavaOptions',
      '"-Dio.netty.tryReflectionSetAccessible=true"'),
     ('spark.app.id', 'local-1611128012143'),
     ('spark.driver.port', '33195'),
     ('spark.kryoserializer.buffer.max', '1024m'),
     ('spark.executor.id', 'driver'),
     ('spark.driver.maxResultSize', '1g'),
     ('spark.driver.memory', '1g'),
     ('spark.executor.cores', '2'),
     ('spark.executor.extraJavaOptions',
      '"-Dio.netty.tryReflectionSetAccessible=true"'),
     ('spark.sql.catalogImplementation', 'hive'),
     ('spark.rdd.compress', 'True'),
     ('spark.app.name', 'Advanced analytics with SPARK - Chapter 3'),
     ('spark.serializer.objectStreamReset', '100'),
     ('spark.master', 'local[*]'),
     ('spark.submit.pyFiles', ''),
     ('spark.submit.deployMode', 'client'),
     ('spark.ui.showConsoleProgress', 'true')]



## Load dataset and Preprocessing


```python
!pwd
data_path = '/home/jovyan/work/ch03/profiledata_06-May-2005/'
import os
```

    /home/jovyan/work/ch03
    


```python
!cat '/home/jovyan/work/ch03/profiledata_06-May-2005/README.txt'
```

    Music Listening Dataset
    Audioscrobbler.com
    6 May 2005
    --------------------------------
    
    This data set contains profiles for around 150,000 real people
    The dataset lists the artists each person listens to, and a counter
    indicating how many times each user played each artist
    
    The dataset is continually growing; at the time of writing (6 May 2005) 
    Audioscrobbler is receiving around 2 million song submissions per day
    
    We may produce additional/extended data dumps if anyone is interested 
    in experimenting with the data. 
    
    Please let us know if you do anything useful with this data, we're always
    up for new ways to visualize it or analyse/cluster it etc :)
    
    
    License
    -------
    
    This data is made available under the following Creative Commons license:
    http://creativecommons.org/licenses/by-nc-sa/1.0/
    
    
    Files
    -----
    
    user_artist_data.txt
        3 columns: userid artistid playcount
    
    artist_data.txt
        2 columns: artistid artist_name
    
    artist_alias.txt
        2 columns: badid, goodid
        known incorrectly spelt artists and the correct artist id. 
        you can correct errors in user_artist_data as you read it in using this file
        (we're not yet finished merging this data)
        
        
    Contact Info
    ------------
    rj@audioscrobbler.com
    irc://irc.audioscrobbler.com/audioscrobbler
    

### user_artist_data

- empty space로 컬럼 구분
- id, count 모두 integer type로 구성됨


```python
!head /home/jovyan/work/ch03/profiledata_06-May-2005/user_artist_data.txt
```

    1000002 1 55
    1000002 1000006 33
    1000002 1000007 8
    1000002 1000009 144
    1000002 1000010 314
    1000002 1000013 8
    1000002 1000014 42
    1000002 1000017 69
    1000002 1000024 329
    1000002 1000025 1
    


```python
user_artist_schema = T.StructType([
    T.StructField("userid", T.IntegerType(), True),
    T.StructField("artistid", T.IntegerType(), True),
    T.StructField("playcount", T.IntegerType(), True),
])

user_artist_df = (
    spark
    .read.format("csv")
    .option("header", False)
    .option("sep", " ")
    .schema(user_artist_schema)
    .load(data_path + "user_artist_data.txt")
)

user_artist_df.show(5)
```

    +-------+--------+---------+
    | userid|artistid|playcount|
    +-------+--------+---------+
    |1000002|       1|       55|
    |1000002| 1000006|       33|
    |1000002| 1000007|        8|
    |1000002| 1000009|      144|
    |1000002| 1000010|      314|
    +-------+--------+---------+
    only showing top 5 rows
    
    

### artist_data


```python
!head /home/jovyan/work/ch03/profiledata_06-May-2005/artist_data.txt
```

    1134999	06Crazy Life
    6821360	Pang Nakarin
    10113088	Terfel, Bartoli- Mozart: Don
    10151459	The Flaming Sidebur
    6826647	Bodenstandig 3000
    10186265	Jota Quest e Ivete Sangalo
    6828986	Toto_XX (1977
    10236364	U.S Bombs -
    1135000	artist formaly know as Mat
    10299728	Kassierer - Musik für beide Ohren
    


```python
artist_schema = T.StructType([
    T.StructField("artistid", T.IntegerType(), True),
    T.StructField("artistname", T.StringType(), True),
])

artist_df = (
    spark
    .read.format("csv")
    .option("header", False)
    .option("sep", "\t")
    .schema(artist_schema)
    .load(data_path + "artist_data.txt")
)

artist_df.show(5, False)
```

    +--------+----------------------------+
    |artistid|artistname                  |
    +--------+----------------------------+
    |1134999 |06Crazy Life                |
    |6821360 |Pang Nakarin                |
    |10113088|Terfel, Bartoli- Mozart: Don|
    |10151459|The Flaming Sidebur         |
    |6826647 |Bodenstandig 3000           |
    +--------+----------------------------+
    only showing top 5 rows
    
    

### artist_alias


```python
!head /home/jovyan/work/ch03/profiledata_06-May-2005/artist_alias.txt
```

    1092764	1000311
    1095122	1000557
    6708070	1007267
    10088054	1042317
    1195917	1042317
    1112006	1000557
    1187350	1294511
    1116694	1327092
    6793225	1042317
    1079959	1000557
    


```python
artist_alias_schema = T.StructType([
    T.StructField("badid", T.IntegerType(), True),
    T.StructField("goodid", T.IntegerType(), True),
])

artist_alias_df = (
    spark
    .read.format("csv")
    .option("header", False)
    .option("sep", "\t")
    .schema(artist_alias_schema)
     .load(data_path + "artist_alias.txt")
)

artist_alias_df.show(5, False)
```

    +--------+-------+
    |badid   |goodid |
    +--------+-------+
    |1092764 |1000311|
    |1095122 |1000557|
    |6708070 |1007267|
    |10088054|1042317|
    |1195917 |1042317|
    +--------+-------+
    only showing top 5 rows
    
    

- user_artist_df의 artistid 필드를 artist_df를 참조하여 badid를 goodid로 교체
- broadcase 함수 적용
- cache 함수를 적용하면 Storage 탭에서 메모리 사용량을 알 수 있음


```python
new_user_artist_df = (
    user_artist_df
    .join(F.broadcast(artist_alias_df), user_artist_df.artistid == artist_alias_df.badid, "left")
    .withColumn("artistid", F.when(F.col("badid").isNull(), F.col("artistid")).otherwise(F.col("goodid")))
    .where(F.col("badid").isNotNull())
    .cache()
)
```


```python
new_user_artist_df.show()
```

    +-------+--------+---------+-------+-------+
    | userid|artistid|playcount|  badid| goodid|
    +-------+--------+---------+-------+-------+
    |1000002| 1000518|       89|1000434|1000518|
    |1000002| 1001514|        1|1000762|1001514|
    |1000002|     721|        1|1001220|    721|
    |1000002| 1034635|        5|1001410|1034635|
    |1000002|    3066|        1|1002498|   3066|
    |1000002| 6691692|        1|1003377|6691692|
    |1000002| 1237611|        1|1003633|1237611|
    |1000002| 1034635|        4|1006102|1034635|
    |1000002| 1001172|        1|1007652|1001172|
    |1000002| 1008391|        2|1010219|1008391|
    |1000002| 2006683|        1|1017405|2006683|
    |1000002| 1000840|        2|1059598|1000840|
    |1000002| 2058809|        2|   3197|2058809|
    |1000002| 1066440|       76|   5702|1066440|
    |1000002| 2003588|        2|    709|2003588|
    |1000019| 1239413|        1|1000287|1239413|
    |1000019| 2001739|        1|1000586|2001739|
    |1000019| 1247540|        6|1000943|1247540|
    |1000019| 1049809|        4|1001379|1049809|
    |1000019|    4377|        1|1002143|   4377|
    +-------+--------+---------+-------+-------+
    only showing top 20 rows
    
    

## Build Model (Spark Tutorial)


```python
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.recommendation import ALS 

als = ALS(seed=42,
          implicitPrefs=False, # Explicit vs Implicit
          rank=10,
          regParam=0.01,
          alpha=1.0,
          maxIter=5,
          userCol="userid", itemCol="artistid", ratingCol="playcount",
          coldStartStrategy="drop")

(train, test) = new_user_artist_df.randomSplit([0.8, 0.2])
als_model = als.fit(new_user_artist_df)
```


```python
predictions = als_model.transform(test)
evaluator = RegressionEvaluator(metricName="rmse", labelCol="playcount", predictionCol="prediction")
rmse = evaluator.evaluate(predictions)
print("Root-mean-square error = " + str(rmse))
```

    Root-mean-square error = 11.475695888786678
    


```python
# Generate top 10 movie recommendations for each user
userRecs = als_model.recommendForAllUsers(10)
# Generate top 10 user recommendations for each movie
movieRecs = als_model.recommendForAllItems(10)

# Show recomeadations
# userRecs.show(n=10, truncate=False)
# movieRecs.show(n=10, truncate=False)
```


```python
# Generate top 10 movie recommendations for a specified set of users
users = new_user_artist_df.select(als.getUserCol()).distinct().limit(3)
userSubsetRecs = als_model.recommendForUserSubset(users, 10)

users.show(n=3, truncate=False)
userSubsetRecs.show(n=3, truncate=False)
```

    +-------+
    |userid |
    +-------+
    |2130958|
    |2131738|
    |2132906|
    +-------+
    
    +-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    |userid |recommendations                                                                                                                                                                                                            |
    +-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    |1000484|[[1279475, 509.51498], [4423, 285.59818], [1254487, 198.77986], [1002128, 174.14009], [2134114, 156.99306], [1308254, 120.996124], [1061239, 114.24648], [1086065, 110.887115], [1240919, 110.20026], [1082068, 110.14881]]|
    |1000584|[[6899306, 180.5835], [1279475, 120.61687], [1123047, 113.81703], [1280614, 112.68537], [1313603, 97.26634], [1055411, 94.64791], [1101322, 92.81742], [6846841, 79.280426], [10073912, 78.5072], [4371, 77.58006]]        |
    |1000509|[[1017916, 633.2707], [2043183, 617.6902], [6799188, 440.63922], [1101322, 329.008], [1001017, 271.35806], [2036225, 232.64966], [1248506, 231.83801], [7034181, 212.84283], [7018959, 210.13376], [1237708, 204.71098]]   |
    +-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    
    


```python
# Generate top 10 user recommendations for a specified set of movies
movies = new_user_artist_df.select(als.getItemCol()).distinct().limit(3)
movieSubSetRecs = als_model.recommendForItemSubset(movies, 10)
```


```python
movies.show(n=3, truncate=False)
movieSubSetRecs.show(n=3, truncate=False)
```

    +--------+
    |artistid|
    +--------+
    |1239413 |
    |1001129 |
    |1239554 |
    +--------+
    
    +--------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    |artistid|recommendations                                                                                                                                                                                                           |
    +--------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    |1239413 |[[2088690, 452.4639], [2027150, 441.07208], [1036787, 136.33311], [1060553, 127.61982], [1031451, 123.03507], [2288164, 117.47447], [1077099, 117.15578], [1047508, 110.3871], [2188843, 106.95569], [1070204, 106.48883]]|
    |1239554 |[[1077099, 957.29675], [2213427, 686.50806], [1031723, 643.5707], [2019846, 614.0945], [1058542, 552.22235], [1073616, 512.73346], [1064240, 511.27615], [1010177, 495.19293], [2042801, 492.02536], [2248675, 471.95676]]|
    |1001129 |[[1041675, 6512.148], [2027150, 6130.94], [1036787, 4110.981], [2009645, 2701.3748], [1041919, 2336.6821], [2045193, 2287.1392], [1001440, 1945.2133], [2017087, 1920.5879], [1058542, 1863.7944], [1047972, 1838.2716]]  |
    +--------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    
    

## Build Model (Text Book)                                                                                 


```python
als_model.userFactors.show(1, truncate=False) # Rank 10
```

    +---+-----------------------------------------------------------------------------------------------------------------------------+
    |id |features                                                                                                                     |
    +---+-----------------------------------------------------------------------------------------------------------------------------+
    |90 |[-0.028082313, 7.6311367E-4, 0.2491423, -0.5567669, -0.13148631, -0.3197596, 4.7639868E-4, -0.64081705, 0.6159623, -0.577028]|
    +---+-----------------------------------------------------------------------------------------------------------------------------+
    only showing top 1 row
    
    


```python
userID = 1004666

existing_artist_ids = (
    new_user_artist_df.where(F.col("userid") == userID)
    .select(F.col("artistid").cast(T.IntegerType()))
    .collect()
)
existing_artist_ids = [row.artistid for row in existing_artist_ids]

artist_df.where(F.col("artistid").isin(existing_artist_ids)).show(n=30, truncate=False)
```

    +--------+----------------------------------+
    |artistid|artistname                        |
    +--------+----------------------------------+
    |1302232 |郭富城                            |
    |1020    |The Dave Brubeck Quartet          |
    |1328360 |鄧麗君                            |
    |1034635 |[unknown]                         |
    |1338195 |山崎まさよし                      |
    |6796568 |Les Petits Chanteurs de Saint-Marc|
    |9988765 |伊藤多喜雄                        |
    |1300816 |相川七瀬                          |
    |1003579 |LeAnn Rimes                       |
    |1280437 |倉木麻衣                          |
    |1345189 |米米CLUB                          |
    |1349540 |渡辺美里                          |
    |3066    |Nat King Cole                     |
    |1029324 |TM Network                        |
    |1020059 |Young M.C.                        |
    |1230410 |Billy Paul Williams               |
    |1300525 |氣志團                            |
    |2061677 |渡辺香津美                        |
    |1266817 |Stan Getz & João Gilberto         |
    |1028104 |Intenso Project                   |
    |2003588 |KoЯn                              |
    |1261516 |The Bobby Fuller Four             |
    |1271892 |Zigo                              |
    |2007793 |Luiz Bonfá                        |
    |1330911 |大黒摩季                          |
    |1328587 |木村弓                            |
    |1307741 |少年隊                            |
    |2065806 |柏原芳恵                          |
    |6834961 |大森俊之                          |
    |1235439 |The Murmurs                       |
    +--------+----------------------------------+
    only showing top 30 rows
    
    


```python
def makeRecomendations(model, userid, howmany):
    to_recommend = (
        model.itemFactors.select(F.col("id").alias("artistid"))
        .withColumn("userid", F.lit(userid))
    )

    return model.transform(to_recommend).select("artistid", "prediction").orderBy(F.col("prediction").desc()).limit(howmany)

recomendation = makeRecomendations(als_model, userID, 3)
recomendation.show()
```

    +--------+----------+
    |artistid|prediction|
    +--------+----------+
    |    4423| 488.04785|
    | 1279475| 458.93817|
    | 2134114|  413.4322|
    +--------+----------+
    
    


```python
recomendation
```




    DataFrame[artistid: int, prediction: float]



### Build Model (towards data science)


```python
# Import the required functions
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.recommendation import ALS
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
from pyspark.ml.evaluation import RegressionEvaluator

# Create ALS model
als = ALS(
         userCol="userid", 
         itemCol="artistid",
         ratingCol="playcount", 
         nonnegative = True, 
         implicitPrefs = False,
         coldStartStrategy="drop"
)

# Add hyperparameters and their respective values to param_grid
param_grid = (
    ParamGridBuilder()
    .addGrid(als.rank, [5, 30])
    .addGrid(als.regParam, [4.0, 0.0001])
    .addGrid(als.alpha, [1.0, 40.0])    
    .build()
)

# Define evaluator as RMSE and print length of evaluator
evaluator = RegressionEvaluator(
    metricName="rmse", 
    labelCol="playcount", 
    predictionCol="prediction") 

# Build cross validation using CrossValidator
cv = CrossValidator(estimator=als, estimatorParamMaps=param_grid, evaluator=evaluator, numFolds=5)
```


```python
# Fit cross validator to the 'train' dataset
model = cv.fit(train)

# Extract best model from the cv model above
best_model = model.bestModel

# View the predictions
test_predictions = best_model.transform(test)
RMSE = evaluator.evaluate(test_predictions)
print(RMSE)

print("**Best Model**")
print("  Rank:", best_model._java_obj.parent().getRank())
print("  MaxIter:", best_model._java_obj.parent().getMaxIter())
print("  RegParam:", best_model._java_obj.parent().getRegParam())
```


```python
# Generate n Recommendations for all users
recommendations = best_model.recommendForAllUsers(5)
recommendations.show(10, False)
```


```python
nrecommendations = recommendations\
    .withColumn("rec_exp", F.explode("recommendations"))\
    .select("userid", "rec_exp.artistid", "rec_exp.rating")
    
nrecommendations.limit(10).show()
```

    +-------+--------+---------+
    | userid|artistid|   rating|
    +-------+--------+---------+
    |1000144| 2043183|61.646343|
    |1000144| 1273059|59.896374|
    |1000144| 1027760| 38.36873|
    |1000144| 9985060| 37.61595|
    |1000144| 2091861| 36.61011|
    |1000465| 6672069|21.925694|
    |1000465| 1032434|21.773518|
    |1000465| 6812406|21.544062|
    |1000465| 2091861|  21.4468|
    |1000465| 1337692|20.663816|
    +-------+--------+---------+
    
    

## Prediction vs Real Data 


```python
nrecommendations.join(artist_df, on="artistid").filter('userid = 1000190').sort(F.col("rating").desc()).show()
```

    +--------+-------+---------+-----------------+
    |artistid| userid|   rating|       artistname|
    +--------+-------+---------+-----------------+
    | 6672069|1000190|32.698563|           hiro:n|
    |    2513|1000190|30.011713|          Merzbow|
    | 2091861|1000190|  29.3715|Purified in Blood|
    | 2043183|1000190|  29.2197|       中川幸太郎|
    | 1167516|1000190|27.088203|       Putsch '79|
    +--------+-------+---------+-----------------+
    
    


```python
(
    new_user_artist_df.select("userid", "artistid", "playcount")
    .join(artist_df, on="artistid")
    .filter('userid = 1000190')
    .sort(F.col("playcount").desc())
).show()
```

    +--------+-------+---------+--------------------+
    |artistid| userid|playcount|          artistname|
    +--------+-------+---------+--------------------+
    |     754|1000190|       20|           Sigur Rós|
    | 6715171|1000190|        9|        The '89 Cubs|
    | 1283231|1000190|        6|The Les Claypool ...|
    | 1290488|1000190|        4|The Nation of Uly...|
    | 1146220|1000190|        1|   Animal Collective|
    | 1013111|1000190|        1|  Murder City Devils|
    | 1004758|1000190|        1|         Silver Jews|
    +--------+-------+---------+--------------------+
    
    
