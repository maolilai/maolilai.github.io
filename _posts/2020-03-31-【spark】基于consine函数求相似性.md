---
layout: post
title:  "[spark]基于consine函数求相似性"
date:   2020-03-31 12:16:01
categories: 机器学习
---
# [spark]基于consine函数求相似性

余弦相似性，数值通过两个向量的夹角的余弦值来度量他们之间的相似性情况。 比如完全一致，夹角为0 ，余弦值为1
a*b = |A| * |B| * consine（sitar）


```
from pyspark.ml import Pipeline
from pyspark.ml.feature import  StringIndexer,StandardScaler,OneHotEncoder
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder
from pyspark.ml.evaluation import BinaryClassificationEvaluator

from pyspark.sql.functions import udf, col

def cos_sim(a,b):
    tt = np.zeros(90)
    #tt = []
    j = 0
    a=str(a).strip()
    a=filter(str.isdigit, a)
    a=a[0:90]
    for i in a:
        tt[j] =tt[j] + (float(i))
        j = j + 1
    a_float_new = Vectors.dense([float(i) for i in tt])  ###.astype(np.float)
    #print(a_float_new)
    return float(np.dot(a_float_new, b) / (np.linalg.norm(a_float_new) * np.linalg.norm(b)))



def random_dense_vector(length=90):
    return Vectors.dense([float(np.random.random()) for i in xrange(length)])


if __name__ == "__main__":

    # create a random static dense vector
    static_vector = random_dense_vector()


    user_name =  'xxx'
    password = 'xxx'
    db = 'xx'
    tbl = 'xx'
    tbl_result = ''

    spark  = SparkSession \
            .builder \
            .appName("clusterModel") \
            .getOrCreate()

    省略了一些代码

    static_vectorb = Vectors.dense([
       0.879306,
0.878479889, --- 修改下
 ])
    c = random_dense_vector(90)

    ### 自定义函数 ，如果是固定值要用lit 转化
    train_xiaohao = df.withColumn("cosin", udf(cos_sim, FloatType())(col("xxxx"),array([，如果是固定值要用lit 转化(v) for v in static_vectorb])))
    #train_xiaohao.show(30)
    train_xiaohao.printSchema()

    train_xiaohao.fillna(0).show(50)
    #print train_xiaohao.first()  

    # df_split = df.withColumn("s", split(df['score'], " ")).select('gid', 's')
    # df["temperatures"].cast(types.ArrayType(types.StringType())))

    #for name, index in attrs:
    #    df = df.withColumn(str(name),str(df['myCol']))
    tbl_result = 'tmp_maolilai_yangben_xiaohgao_project_20181106_bitmap_result3'

    tdw.saveToTable(train_xiaohao,tbl_result)  
    #spark.stop()
```
