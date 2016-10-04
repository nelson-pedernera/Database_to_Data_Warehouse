# Database_to_Data_Warehouse
Data Pipeline built for transferring data from RDS to Redshift in the AWS cloud<br />
Data scientists need to use data warehouse to generate data dashboards and run machine learning algorithms. But usually the source of user activities are saved in relational database or MySQL. Since a lot of companies are hosted on AWS, this project tried to build up a stable transfer pipeline from Amazon RDS to Redshift for everyday usage to update the Redshift with the latest data created in RDS.<br />
<img width="485" alt="background" src="https://cloud.githubusercontent.com/assets/17118374/19009766/e8eb1bfc-872b-11e6-9aa1-08ed35146502.png"><br />
This project tried to construct a stable data pipeline that transfers data from RDS to Redshift even with schema changes
The challenges are:<br />
1. Transfer the data in a manner that involved with minimal amount of data (instead of droping and copying entire tables each time)<br />
2. Transfer the data in a faster distributed way (use more than one threads)<br />
3. Handle the schema changes from RDS without breaking the pipeline (COPY will fail if schema is different from source)<br />
4. Solve the date format issue that Sqoop Avro casted datetime to long type <br />

Amazon S3 is used as the transfer station between RDS and Redshift, because Redshift currently only allow users to copy data from S3 or DynamoDB. S3 is simpler to use as compared to DynamoDB for me.<br />
<img width="565" alt="pipeline" src="https://cloud.githubusercontent.com/assets/17118374/19009850/ed06a43a-872c-11e6-90f2-b1c376e1c547.png"><br />
From RDS to S3<br />
<br />
I spinned up EMR Cluster (m3.xlarge 1 core) to use Sqoop to transfer data from RDS to S3. <br />
"-m" parameter in Sqoop gives the option of using multiple workers for the load. By this way, there will be multiple files in the S3 buckets and make sure the next COPY step will use all of them as inputs.<br />
After sqoop, the local folder where you call the commands will have a file "QueryResult.avsc" with the schema information. We copy this temparary file to a permanant location with the corrent information of table in the file name. <br />
MySQL JDBC driver is needed for the connection to RDS server. Place the downloaded jar file to EMR path: /usr/lib/sqoop/<br />
<br />
Step 1: <br />
script file: 1_schema_load.py <br />
This script did a minimal query (just one row for each table) by Sqoop Avro and the purpose is to get the new schemas in Avro format. Now the date had been casted to long type, we will fix them in part 2 when we actually load the RDS data.<br />

Step 2: <br />
script file: 2_data_load.py <br />
This script did a query that incremental import the new data (data had been updated since last import) from the RDS.<br />
In this step, we scan the schema information created in step 1 and cast the date to Java String format. <br />
To achieve this, "--map-column-java name=String" parameter is used, in which name is the column name we want to do casting.<br />
Now after importing, we will generate a new temparary schema in QueryResult.avsc, move it to a permanant place for next steps. <br />
<br />
From S3 to Redshift <br />
<br />
Python package psycopg2 is used to connect to postgreSQL database hosted in Redshift. <br />
The scripts will use the schema files created in Step 2 and compare it with the version created by last import. <br />
If there is schema change, we will need to alter the Redshift schema to avoid COPY failure. <br />
However, dropping the entire table is unnecessary since the ALTER table commands provided by Amazon allows us add columns to Redshift with Default values. In addition, loading using Avro schema is flexible because the data doesn't need to follow the predefined order in the schema as long as the column names can match. <br />
Taking these factors into account, we can still use only the new updated since last import to update the Redshift in the situation of schema changes, so that to miniize the data flow. As far as I tested, create and COPY a ~3GB large new table from RDS to Redshift can take up to 20 minutes, while only merging 5% new data takes less than a minute.<br />
<br />
Step 3: <br />
script file: 3_schema_check.py <br />
To handle the schema change, we load the avro schema .avsc files as json object and return a map of (columnName, dataType), which we can use to alter Redshift table in Step 4 <br />

Step 4: <br />
script file: 4_redshift_update.py <br />
We have 3 situations: <br />
  . The table is new for Redshift. So we scan the avro schema .avsc file for the table and use "create table" commands to initilize the table with all the columns. Remeber to handle the date type can cast the Java String back to timestamp in Redshift. Then we call COPY commands to unload the S3 data to Redshift.<br />
  . The table exists but schema changed. So we use the step 3 results to generate the "alter table add columns" commands to adjust the Redshift schema and put default values in the new columns. And then it is no difference from the table merge without schema changes.<br />
  . Table exists and schema not changed. We create a temp table called stage in Redshift and COPY the S3 data into it. Then we delte the rows whose primary id is equal to the temp table's id. Finally we insert the temp table into the current Redshift table. This is a transaction process.<br />
 <img width="566" alt="schema_check" src="https://cloud.githubusercontent.com/assets/17118374/19009915/cdb5f4d6-872d-11e6-84eb-21900a5b270f.png"><br />

Presentation slide is at: http://bit.ly/2dGrfuG
