# Informix Database Export

## 1.0. Challenge
A customer wants to move off an on-premise installation of IBM Informix Database Server (IDS) onto a cloud based platform and they are looking for the best technique to achieve this. In the short term, they want to replicate the data into a system like AWS RedShift and point power users to do their reporting off that system, but they need some help in determining the best way to do that. So not only replicate the data the first time but also keep the database server and the new system in sync (on a daily basis).

### 1.1. Objectives
1. The best way to do a full one-time data dump, with little or no database server downtime.   
2. The best approach to keep the exported data dump synchronized daily. 

### 1.2. Current System
* IBM Informix Database Server (IDS): v11.50
* Logical size: 1000+ tables & multiple views (across multiple dbspaces)
* Physical size:  ~1TB (spanning multiple raw disks)
* Concurrent users: 50


## 2.0. Requirements
The two (2) objectives mentionied in section 1.1. are really just asking for how to do a: *full backup* and an *incremental backup* on an ongoing basis with little or no downtime for other database users. 
**Figure 1** summarizes the different backup levels that are available in Informix.


##### Figure 1: Backup Levels
| Level | Description |
|-------|-------------|
|Level 0 | Backs up all used pages that contain data for the specified storage spaces.|
|Level 1 | Backs up only data that has changed since the last level-0 backup of the specified storage spaces.|
|Level 2 | Backs up only data that has changed since the last level-1 backup of the specified storage spaces.|


Based on **Figure 1**, a full backup can be thought of as a level-0 backup while an incremental backup can be thought of as either a level-1 or a level-2 backup. Level-1 and level-2 backups will typically have shorter backup windows than a level-0 backup. Since the client is requesting for _daily_ incremental backups, we could use the _backup plan_ in **Figure 2** below to closely model the client's requirements.


##### Figure 2: Backup Plan
|Backup Level| 	Backup Frequency| Backup Schedule |
|------------|-----------------|------------------|
|Full backup (level-0) | Monthly       | Last Sunday at 6pm |
|Incremental backup (level-1) | Weekly | On Sunday at 12am |
|Incremental backup (level-2) | Daily | At 12am |


In order to figure out how to make this backup plan work for the client, the next section will highlight and discuss different backup types; recommend a suitable backup type for each backup frequency; then recommend which backup utilities are best suited to realizing the backup plan. 


### 2.1. Backup Types
Broadly speaking, there are two (2) types of backups: physical backups and logical backups.

#### 2.1.1. Physical
Physical backups involve making raw copies of the directories and files that are used to store a database's data files. 
This means copying the actual data files -- in binary -- that constitute the database directly from disk. As a result, physical backups offer the fastest speeds for duplicating very large databases but they have a disadvantage of being machine and platform dependent -- the backup might only be recoverable on a machine running an identical version of the database server. Physical backups are also referred to as _cold_ backups.

#### 2.1.2. Logical
Logical backups involve reproducing a database's table structure and data (records) usually into a delimited-text file format. 
Textual backups are usually machine and platform (i.e. operating system) independent but they can take several hours for very large databases. Logical backups are also know as _hot_ backups.

#### 2.1.3. Characteristics
The table below summarizes the characteristics of physical backups when compared with logical backups.

|   | Characteristic | Physical | Logical|
|---|-------------|-------------|--------|
|1|Backup Data | Exact copies of files containing actual data. | Logical representation of data in storage.| 
|2|Backup Granularity | Storage spaces, logical logs, config files etc. | Database- and table-level granularity only.|
|3|Backup Mode | Offline: server may not accept connections. | Online: server may still accept connections.|
|4|Backup Output | Binary format. Compact. | Delimited-text format. Verbose.|
|5|Backup Portability | Machine and platform dependent. | Machine independent and highly portable across platforms.|
|6|Backup Speed | Directly from disk. Very fast. | Requires table locks. Slow for large databases.|

In order to identify which database backup utilities will be useful in accomplishing the backup plan, the next section will present a quick summary of several Informix database utilities, then use the characteristics above to highlight differences between them.


### 2.2. Database Utilities
Informix offers several data migration utilites which can be used to perform backups as listed [here](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.mig.doc/ids_mig_246.htm) but there are a few other utilities like `High Performance Unload (HPL)` and `ontape` which will be included in this review. 

1. [dbexport/dbimport](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.mig.doc/ids_mig_113.htm) - `dbexport` and `dbimport` utilities import and export a database and its schema to disk or tape. 
`dbexport` locks the database in exclusive mode. To execute `dbexport`, no users are allowed to be connected to the database.

2. [onunload/onload](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.mig.doc/ids_mig_191.htm?lang=en) - `onunload` and `onload` utilities provide the fastest way to move data between computers that use identical IDS versions. 
The `onunload` and `onload` utilities are faster than `dbimport`, `dbload`, or LOAD but are much less flexible and do not permit modifications to the database schema nor does it allow mixing and matching between different OS or database server versions. 

3. [High Performance Unload (HPL)](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.hpl.doc/hpl.htm) -
HPL is similar to the `dbexport` utility with the additional benefit of allowing data compression during the export to conserve disk space. The main difference with `dbexport` is locks -- HPL does not lock out other users from the database, as a result, HPL does not force consistency of the exported data because no exclusive lock is set. The only way to guarantee consistency is to manually prevent the data from being updated while the unload is running.

4. [onbar](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.bar.doc/ids_bar_169.htm?lang=en) - `onbar` can back up and verify storage spaces -- dbspaces, blobspaces, and sbspaces (physical backup) and logical-log files (logical backup). `onbar`, does not have direct control of the backup data unlike other utilities. Instead, it delegates read and writes to a Storage Manager (SM) which manages storage devices that might not be as fast as disks. Many Storage Managers will allow backups to disk in place of a tape device. 

5. [ontape](http://www.ibm.com/support/knowledgecenter/SSGU8G_11.70.0/com.ibm.bar.doc/ids_bar_170.htm) - `ontape` utility supports backing up of storage spaces (physical backup) and logical-log files (logical backup). Although `ontape` was designed to use tapes for archiving backups, it equally supports backing up to disks.


## 3.0. Proposal
In order to implement the full and incremental backups in the backup plan, a suitable tool must support the creation of backups at different [backup levels](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.bar.doc/ids_bar_515.htm?lang=en). Level-0 and level-1/level-2 backups, in most cases correspond to the creation of physical and logical backups respectively. 

Out of the utilities reviewed, only two (2) utilities support the backup creation at different backup levels: `onbar` and `ontape`, some feature highlights along with a comparison between them is in order.

### 3.1. onbar
In addition to supporting different backup levels, the "v" command line option of the `onbar` utility: [onbar -v](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.bar.doc/ids_bar_231.htm) can be used to verify a whole-system or physical-only backup. In other words, it provides *backup integrity* checking of physical backups but it cannot be used for verifying logical logs. 

The `onbar -v` command runs the `archecker` utility to verify that all pages required to restore a backup exist on the media in the correct form.

For instance, this command verifies the backed-up storage spaces that are listed in the file `bkup1`:
```
onbar -v -f /usr/backups/bkup1
```

### 3.2. ontape
Starting from IDS version 11.70, `ontape` supports *"back up and restore Informix database data to or from cloud storage"*. In other words, it can be used to back up and restore data to or from the Amazon Simple Storage Service (S3). 

If the client is willing to consider upgrading IDS from 11.5 to 11.7, they will be able take advantage of a vendor supported mechanism of [exporting natively to S3](http://www.ibm.com/support/knowledgecenter/SSGU8G_11.70.0/com.ibm.bar.doc/ids_bar_509.htm), which is essentially replication to AWS with fewer moving parts. 

For instance, you can use the `ifxbkpcloud.jar` utility to create and name a storage device in the region where you intend to store Informix data.
```
java -jar ifxbkpcloud.jar CREATE_DEVICE amazon mytapedevice US_Standard

# Set the TAPEDEV and LTAPEDEV configuration parameters in the onconfig file to the cloud storage location. 
TAPEDEV '/opt/IBM/informix/tapedev_dir, keep = yes, cloud = amazon,
        url = https://mytapedevice.s3.amazonaws.com'
LTAPEDEV '/opt/IBM/informix/ltapedev_dir, keep = yes, cloud = amazon, 
        url = https://mylogdevice.s3.amazonaws.com'

# Back up data to the cloud storage device by using the ontape utility.
ontape -s -L 0
```

The exported data in [AWS S3 can be imported](http://docs.aws.amazon.com/redshift/latest/dg/t_Loading-data-from-S3.html) using the [COPY command](http://docs.aws.amazon.com/redshift/latest/dg/r_COPY.html) which appears to be the most efficient way to load data into tables on AWS Redshift.


### 3.3. Comparison
Most of the information here comes directly from the IBM documentation that [compares](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.bar.doc/ids_bar_177.htm?lang=en) `onbar` with `ontape` but note that the backup that `ontape` and `onbar` produce are not compatible. You cannot create a backup with `ontape` and restore it with `onbar`, or vice versa.

**onbar**
    Backs up and restores storage spaces (dbspaces) and logical files, by using a storage manager to track backups and storage media. Use this utility when you need to:

        Select specific storage spaces
        Back up to a specific point in time
        Perform separate physical and logical restores
        Back up and restore different storage spaces in parallel
        Use multiple tape drives concurrently for backups and restores
        Perform imported restores
        Perform external backups and restores

**ontape**
    Logs, backs up, and restores data, and enables you to change the logging status of a database. It does not use a storage manager. Use this utility when you need to:

        Back up and restore data without a storage manager
        Back up without selecting storage spaces
        Change the logging mode for databases

**[Comprehensive Differences](https://www.ibm.com/support/knowledgecenter/SSGU8G_12.1.0/com.ibm.bar.doc/ids_bar_177.htm?lang=en)**
