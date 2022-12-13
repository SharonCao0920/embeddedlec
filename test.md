```python
!pip install https://github.com/aaren/notedown/tarball/master
```

```python
!notedown /content/drive/MyDrive/Colab_Notebooks/CS570/CS522_Week9_HW2_Yixin_Cao_19536.ipynb --to markdown --strip > test.md
```

### **Step 1: Go to the correct directory**

```python
%cd drive/MyDrive/Colab Notebooks/CS570
!pwd
```

### **Step 2: Convert the .txt data file to .csv file**

```python
import pandas as pd
# covert txt to cvs
read_file = pd.read_csv ("movies.txt")
read_file.to_csv ("movies.csv", index=None)

read_file = pd.read_csv ("ratings.txt")
read_file.to_csv ("ratings.csv", index=None)

read_file = pd.read_csv ("tags.txt")
read_file.to_csv ("tags.csv", index=None)
```

### **Step 3: Install pyspark**

```python
!pip install pyspark
```

### **Step 4: Import libraries**

```python
import pandas as pd
from pyspark.sql.functions import col, explode
from pyspark import SparkContext
```

### **Step 5: Initiate spark session**

```python
from pyspark.sql import SparkSession
sc = SparkContext
# sc.setCheckpointDir('checkpoint')
spark = SparkSession.builder.appName('Recommendations').getOrCreate()
```

### **Step 6: Load data**

```python
%cd drive/MyDrive/Colab Notebooks/CS570
movies = spark.read.csv("./drive/MyDrive/Colab Notebooks/CS570/movies.csv",header=True)
ratings = spark.read.csv("./drive/MyDrive/Colab Notebooks/CS570/ratings.csv",header=True)
```

```python
ratings.show()
```

```python
ratings.printSchema()
```

```python
ratings = ratings.\
    withColumn('userId', col('userId').cast('integer')).\
    withColumn('movieId', col('movieId').cast('integer')).\
    withColumn('rating', col('rating').cast('float')).\
    drop('timestamp')
ratings.show()
```

### **Step 7: Calculate sparsity**

```python
# Count the total number of ratings in the dataset
numerator = ratings.select("rating").count()

# Count the number of distinct userIds and distinct movieIds
num_users = ratings.select("userId").distinct().count()
num_movies = ratings.select("movieId").distinct().count()

# Set the denominator equal to the number of users multiplied by the number of movies
denominator = num_users * num_movies

# Divide the numerator by the denominator
sparsity = (1.0 - (numerator *1.0)/denominator)*100
print("The ratings dataframe is ", "%.2f" % sparsity + "% empty.")
```

### **Step 8: Interpret ratings**

```python
# Group data by userId, count ratings
userId_ratings = ratings.groupBy("userId").count().orderBy('count', ascending=False)
userId_ratings.show()
```

```python
# Group data by userId, count ratings
movieId_ratings = ratings.groupBy("movieId").count().orderBy('count', ascending=False)
movieId_ratings.show()
```

### **Step 9: Build Out An ALS Model**

```python
# Import the required functions
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.recommendation import ALS
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator
```

```python
# Create test and train set
(train, test) = ratings.randomSplit([0.8, 0.2], seed = 1234)

# Create ALS model
als = ALS(userCol="userId", itemCol="movieId", ratingCol="rating", nonnegative = 
          True, implicitPrefs = False, coldStartStrategy="drop")

# Confirm that a model called "als" was created
type(als)
```

### **Step 10: Tell Spark how to tune your ALS model**

```python
# Import the requisite items
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.tuning import ParamGridBuilder, CrossValidator

# Add hyperparameters and their respective values to param_grid
param_grid = ParamGridBuilder() \
            .addGrid(als.rank, [10, 50, 100, 150]) \
            .addGrid(als.regParam, [.01, .05, .1, .15]) \
            .build()
            #             .addGrid(als.maxIter, [5, 50, 100, 200]) \

           
# Define evaluator as RMSE and print length of evaluator
evaluator = RegressionEvaluator(metricName="rmse", labelCol="rating", predictionCol="prediction") 
print ("Num models to be tested: ", len(param_grid))
```

### **Step 11: Build your cross validation pipeline**

```python
# Build cross validation using CrossValidator
cv = CrossValidator(estimator=als, estimatorParamMaps=param_grid, evaluator=evaluator, numFolds=5)

# Confirm cv was built
print(cv)
```

### **Step 12: Best Model and Best Model Parameters**

```python
#Fit cross validator to the 'train' dataset
model = cv.fit(train)
#Extract best model from the cv model above
best_model = model.bestModel

```

```python
# # Print best_model
# print(type(best_model))

# Complete the code below to extract the ALS model parameters
print("**Best Model**")

# # Print "Rank"
print("  Rank:", best_model._java_obj.parent().getRank())

# Print "MaxIter"
print("  MaxIter:", best_model._java_obj.parent().getMaxIter())

# Print "RegParam"
print("  RegParam:", best_model._java_obj.parent().getRegParam())
```

```python
# View the predictions
test_predictions = best_model.transform(test)
RMSE = evaluator.evaluate(test_predictions)
print(RMSE)
```

```python
test_predictions.show()
```

### **Step 13: Make Recommendations**

```python
# Generate n Recommendations for all users
nrecommendations = best_model.recommendForAllUsers(10)
nrecommendations.limit(10).show()
```

```python
nrecommendations = nrecommendations\
    .withColumn("rec_exp", explode("recommendations"))\
    .select('userId', col("rec_exp.movieId"), col("rec_exp.rating"))

nrecommendations.limit(10).show()
```

## **Do the recommendations make sense?**

```python
nrecommendations.join(movies, on='movieId').filter('userId = 100').show()
```

```python
ratings.join(movies, on='movieId').filter('userId = 100').sort('rating', ascending=False).limit(10).show()
```
