Using Cloudera Impala with the GDELT database  
--------------------------------------
This directory contains:
- Files to create the aggregate database made of the historical data files and of the daily updates, using Parquet compression
- Files to pull the daily update on a daily basis and append it to the aggregate database 

Files
-----------------------------
* gdelt_create.sh : calls the download engine to retrieve all the historical and daily updates files, uploads them to HDFS, and populates the Impala database.

* dl_engine.sh : a Bash download engine that is used to pull all the information from the servers at UT Dallas.
The engine assumes that the files are stored as ZIP files (.zip extension) on the server, and are all located under the same remote directory.
The engine has 4 [R]equired arguments and 4 [O]ptional arguments:
	- [R] The target directory in which to download the zip files (--targetdir | -t)
	- [R] The data directory in which the zip files will be extracted (--datadir | -d)
	- [R] The remote URL from where to download the zip files (--url | -u)
	- [R] A log file for the output of the script (--log)
	- [O] The number of retries if a file can not be downloaded at first (--nretry | -r)
	- [O] The number of processes to spawn when unzipping all the files 
	- [O] A verbose option to obtain more output from the command (--verbose | -v)
	- [O] A help message (--help | -h) 

The engine will check that the files are valid once downloaded, and will try to download them again if it failed. 

* hdfs_upload.sh : a Bash script to upload all the unzipped files to HDFS
The script has 3 [R]equired arguments and two [O]ptional arguments:
	- [R] The local directory where the unzipped files are located (--localdir)
	- [R] The HDFS directory in which the files will be uploaded (--hdfsdir)
	- [R] A log file to use (--log)
	- [O] A verbose option to obtain more output from the command (--verbose | -v)
	- [O] A help message (--help | -h) 

* skel/*.sql : Contains the skeleton of the Impala queries to create the aggregate query that will create the tables for historical data, daily updates, and aggregate data.
The script gdelt_create.sh copies the skeleton files to temporary SQL files, then populates them with the variables specified at the beginning of the script. It then creates a single query
file named "create_query.sql", which is then passed to Impala.

* skel/daily_update.skel.sh : Contains the skeleton of the daily_update.sh script which can be used to update the aggregate table if it is outdated. daily_update.sh is created 
during the execution of gdelt_create.sh, and is then populated with the correct variables.

Usage
-----------------------------
1. Edit the file gdelt_create.sh to reflect your local Hadoop cluster configuration
List of variables to edit (\* denotes HIST, DU, or AGG, which refers to HISTorical files, Daily Updates, or AGGregated data):
	- LOGDIR: log file directory location.
	- NRETRY: Number of download retries before a file is skipped.
	- NPROC: Number of processes used to unzip the data files.
	- IMPALA_HOST: Host on which the Impala daemon is running. This should be a data node.
	- KERBEROS: Flag to set Kerberos authentication for the Impala script.
	- PARTITION: Flag to set partitioning of the database by year
	- TMPDIR: A temporary directory is set to save intermediary Impala query files. It is removed once the script has been executed.
	- DB_NAME: The database name to use. If it does not exist, it will be created.
	- DB_LOC: The location of the database on HDFS.
	- *_URL: The remote location of the data files to download.
	- *_TDIR: Directory where the zip files will be saved.
	- *_DDIR: Directory where the zip files will be uncompressed.
	- *_HDFSDIR: HDFS location of the TSV files or saved tables. 
	- *_TBNAME: Table name in the database. IMPORTANT: If the table already exists, it will be dropped !

2. Run the script gdelt_create.sh
```shell
./gdelt_create.sh
```

3. A log file will be created at the location ```$LOGDIR/gdelt_create.log.YYYYMMDD.HHmmss```. You can track the progress of the script using 
```shell 
tail -f $LOGDIR/gdelt_create.log.YYYYMMDD.HHmmss
```

4. Once the script has finished, a new script daily_update.sh will be created in the current directory. This script is already populated with all the info pertaining to the
database, tables, and flags. You can then run the script daily_update.sh at any point, as it will compare the data set last update to the current update on the UT Dallas server,
and decide on what files to download and push into the aggregated table. To run the script, simply type in the command line:
```shell
./daily_update.sh
```

Notes
-----------------------------
* This has been tested with Impala v1.1.1 on the CDH distribution v.4.4.0 patch 39 
* Possible issues include:
	- Directory permissions not set up correctly.
	- Kerberos authentication errors.
	- If the directory in which the tables are stored are not empty, query results will be incorrect.

* Using partitioning can improve result times, as Impala will only load the required files. Only partitioning by year has been implemented so far. Other partitions can be considered, such as year AND month, or actors.
Partitioning is VERY expensive when inserting data during the first creation of the aggregate table. 


