# Big_Data

## Hadoop

### Installation et Configuration
Follow these 2 tutoriels to [install hadoop](https://www.digitalocean.com/community/tutorials/how-to-install-hadoop-in-stand-alone-mode-on-ubuntu-16-04) and [configure Pseudo-Distributed mode](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)

### Upload data files to HDFS
Once hadoop is well installed and configured, we can use the bash command to launch hadoop service and upload files into HDFS
```BASH
# Launch hadoop
$ $HADOOP_HOME/sbin/start-dfs.sh

# Upload data files
$ bin/hdfs dfs -put *data files path*/train.csv dataset
$ bin/hdfs dfs -put *data files path*/test.csv dataset
$ bin/hdfs dfs -put *data files path*/predict.csv dataset

```
### Send the data files from HDFS to AWS S3
For uploading files from HDFS to AWS we use Python script with 2 libraries:
* Pydoop (a Python interface to Hadoop)
* boto3 (Amazon Web Services (AWS) SDK for Python)

We could install these 2 tools by [pip](https://pypi.python.org/pypi/pip)
* For [installing Pydoop](https://crs4.github.io/pydoop/installation.html)
* For [installing boto3](https://boto3.readthedocs.io/en/latest/guide/quickstart.html#installation)

Here's the script we use to upload files:
```python
import pydoop.hdfs as hdfs
import boto3
import pprint

# Hadoop loading process
print "Loading train.csv ..."
data_train_csv = hdfs.open("/user/tghead/dataset/train.csv")

print "Loading test.csv ..."
data_test_csv = hdfs.open("/user/tghead/dataset/test.csv")

print "Loading predict.csv ..."
data_predict_csv = hdfs.open("/user/tghead/dataset/predict.csv")

# AWS upload process
s3 = boto3.resource('s3')

if(s3.Bucket('hadoop_data').creation_date == None):
    print "S3 Bucket 'hadoop_data' not find creating ..."
    bucket = s3.Bucket('hadoop_data').create('private', 'eu-west-2')
    print "S3 Bucket 'hadoop_data' created!"
else:
    print "Loading Bucket 'hadoop_data ...'"
    bucket = s3.Bucket('hadoop_data')

print "Uploading training data to the Bucket 'hadoop_data' ..."
s3.Bucket('hadoop_data').upload_fileobj(data_train_csv, 'train.csv')

print "Uploading test data to the Bucket 'hadoop_data' ..."
s3.Bucket('hadoop_data').upload_fileobj(data_test_csv, 'test.csv')

print "Uploading predict data to the Bucket 'hadoop_data' ..."
s3.Bucket('hadoop_data').upload_fileobj(data_predict_csv, 'predict.csv')

print "Upload finished! Here's the list of objects in bucket 'hadoop_data':"
s3_client = boto3.client('s3')

pprint.pprint(s3_client.list_objects(Bucket='hadoop_data'))
```
## Python script for ML on AWS EC2
Here's our script python which load data from S3 and resend the result to another S3 bucket after our predict algorithm:
```python
import boto3
import pandas as pd
from pandas import Series, DataFrame
from sklearn import preprocessing
from sklearn import tree

s3 = boto3.resource('s3')

print("Loading predict data from S3...")
s3.Object('hadoop_data','predict.csv').download_file('./dataset/predict.csv')

print("Loading test data from S3...")
s3.Object('hadoop_data','test.csv').download_file('./dataset/test.csv')

print("Loading train data from S3...")
s3.Object('hadoop_data','train.csv').download_file('./dataset/train.csv')

test = preprocessing.LabelEncoder()
test2 = preprocessing.LabelEncoder()
test3 = preprocessing.LabelEncoder()
#to convert into numbers



data= pd.read_csv('./dataset/train.csv')
frameTrain = DataFrame(data)

[oldName1,oldName2,oldName3,oldName4,oldName5,oldName6,oldName7,oldName8,oldName9] = frameTrain.columns
frameTrain.rename(columns={oldName1: 'Type', oldName2: 'Montant',oldName3: 'NameSource',oldName4: 'SourceB',oldName5: 'SourceA',oldName6: 'NameDest',oldName7: 'DestB',oldName8: 'DestA',oldName9: 'Label'}, inplace=True)
frameTrain.Type = test.fit_transform(frameTrain.Type)
frameTrain.NameSource = test.fit_transform(frameTrain.NameSource)
frameTrain.NameDest = test.fit_transform(frameTrain.NameDest)
print("training set imported")

data= pd.read_csv('./dataset/predict.csv')
framePredict = DataFrame(data)

[oldName1,oldName2,oldName3,oldName4,oldName5,oldName6,oldName7,oldName8] = framePredict.columns
framePredict.rename(columns={oldName1: 'Type', oldName2: 'Montant',oldName3: 'NameSource',oldName4: 'SourceB',oldName5: 'SourceA',oldName6: 'NameDest',oldName7: 'DestB',oldName8: 'DestA'}, inplace=True)
framePredict.Type = test.fit_transform(framePredict.Type)
framePredict.NameSource = test2.fit_transform(framePredict.NameSource)
framePredict.NameDest = test3.fit_transform(framePredict.NameDest)
print("training set imported")

model = tree.DecisionTreeClassifier(random_state=0)


X_train = frameTrain[['Type','Montant','SourceB','SourceA','DestB','DestA']]
y_train = frameTrain['Label']
X_test = framePredict[['Type','Montant','SourceB','SourceA','DestB','DestA']]

model.fit(X_train,y_train)
print("model trained")

X_predict = model.predict(X_test)
print("model predicted")

framePredict.Type = test.inverse_transform(framePredict.Type)
framePredict.NameSource = test2.inverse_transform(framePredict.NameSource)
framePredict.NameDest = test3.inverse_transform(framePredict.NameDest)
framePredict['Label'] = X_predict
# framePredict

framePredict.to_csv('out.csv', encoding='utf-8', index=False)
data = open('out.csv', 'rb')
print("Uploading result to S3..")
s3.Bucket('fraud-data-predict').put_object(Key='out.csv', Body=data)
```

## The list of commands to launch to retrieve the predict.csv treated by the machine learning algorithm and put it in mongoDB.
We use s3cmd (based on goto3) and mongoimport that come with the default mongoDB install.


## First time configuration : 
### s3cmd download and configure:
```BASH
apt-get update s3cmd
#S3 configuration
s3cmd --configure
Acces key : AKIAJHJEIGQMB4YFS2DQ
Secret key : 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
Default region : eu-west-3
Encryption password : 
Path to PGP program :
Use HTTPS protocol : No
no proxy
```
## Launch mongod  : 
```
mongod --dbpath ./data --smallfiles
```


## The script itself:

### Download the csv from the S3 bucket
```
s3cmd get s3://fraud-data-predict/out.csv
```

### Import the csv into mongodb
There is a header line in the csv so we use --headerline
mongoimport -d predictdb -c predictions --type csv --file out.csv --headerline 


### Connect to the db and list frauds : 
mongo
use predictdb
db.predictions.find({isFraud:1}).pretty()
