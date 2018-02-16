# Big_Data


## The list of commands to launch to retrieve the predict.csv treated by the machine learning algorithm and put it in mongoDB.
We use s3cmd (based on goto3) and mongoimport that come with the default mongoDB install.


##First time configuration : 
###s3cmd download and configure:
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
##Launch mongod  : 
```
mongod --dbpath ./data --smallfiles
```


##The script itself:

###Download the csv from the S3 bucket
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
