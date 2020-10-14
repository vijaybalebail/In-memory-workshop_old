# Enable In-Memory

Oracle Database In-Memory option comes preinstalled with the Oracle Database 12c and does not require additional software installation or recompilation of existing database software. This is because the In-Memory option has been seamlessly integrated into the core of the Oracle Database software as a new component of the Shared Global Area (SGA), so when the Oracle Database is installed, Oracle Database In-Memory gets installed with it.
However, the IM column store is not enabled by default, but can be easily enabled via a few steps, as outlined in this lesson.
It is important to remember that after the In-Memory option is enabled at the instance level, you also have to specifically enable objects so they would be considered for In-Memory column store population.


## Step 1: Logging In and Enabling In-Memory

1.  All scripts for this lab are stored in the labs/inmemory folder and are run as the oracle user.  Let's navigate there now.  We recommend you type the commands to get a feel for working with In-Memory. But we will also allow you to copy the commands via the COPY button.

    ````
    <copy>
    sudo su - oracle
    cd ~/labs/inmemory
    ls
    </copy>
    ````

2. The In-Memory Area is a static pool within the SGA that holds the column format data (also referred to as the In-Memory Column Store). The size of the In-Memory Area is controlled by the initialization parameter INMEMORY\_SIZE (default is 0, i.e. disabled).
As the IM column store is a static pool, any changes to the INMEMORY\_SIZE parameter will not take effect until the database instance is restarted.
Note : It is also not impacted or controlled by Automatic Memory Management (AMM). The default install of database usually set a parameter MEMORY_TARGET which manages both SGA (System Global Area) and PGA(Process Global Area).


    ````
    . oraenv
    ORCL
    ````
     ![](images/step1num1.png)

    ````
    <copy>
    sqlplus / as sysdba
    show sga;
    show parameter inmemory;
    show parameter keep;
    </copy>
    ````

     ![](images/step1num2.png)

    Notice that the SGA is made up of Fixed Size, Variable Size, Database Buffers and Redo.  There is no In-Memory in the SGA.  Let's enable it.

3.  Enter the commands to enable In-Memory.  The database will need to be restarted for the changes to take effect.

    ````
    <copy>
    alter system set inmemory_size=2G scope=spfile;
    shutdown immediate;
    startup;
    </copy>
    ````
     ![](images/step1num3.png)


4.  Now let's take a look at the parameters.

    ````
    <copy>
    show sga;
    show parameter inmemory;
    show parameter keep;
    exit
    </copy>
    ````
     ![](images/step1num4.png)

## Step 2: Enabling In-Memory

The Oracle environment is already set up so sqlplus can be invoked directly from the shell environment. Since the lab is being run in a pdb called orclpdb you must supply this alias when connecting to the ssb account.

1.  Login to the pdb as the SSB user.  
    ````
    <copy>
    sqlplus ssb/Ora_DB4U@localhost:1521/orclpdb
    set pages 9999
    set lines 200
    </copy>
    ````
     ![](images/num1.png)

2.  The In-Memory area is sub-divided into two pools:  a 1MB pool used to store actual column formatted data populated into memory and a 64K pool to store metadata about the objects populated into the IM columns store.  V$INMEMORY_AREA shows the total IM column store.  The COLUMN command in these scripts identifies the column you want to format and the model you want to use.  

    ````
    <copy>
    column alloc_bytes format 999,999,999,999;
    column used_bytes format 999,999,999,999;
    column populate_status format a15;
    --QUERY

    select * from v$inmemory_area;
    </copy>
    ````
     ![](images/num2.png)

3.  
To check if the IM column store is populated with object, run the query below.

    ````
    <copy>
    column name format a30
    column owner format a20
    --QUERY

    select v.owner, v.segment_name name, v.populate_status status from v$im_segments v;
    </copy>
    ````
     ![](images/num3.png)   

4.  To add objects to the IM column store the inmemory attribute needs to be set.  This tells the Oracle DB these tables should be populated into the IM column store.   

    ````
    <copy>
    ALTER TABLE lineorder INMEMORY;
    ALTER TABLE part INMEMORY;
    ALTER TABLE customer INMEMORY;
    ALTER TABLE supplier INMEMORY;
    ALTER TABLE date_dim INMEMORY;
    </copy>
    ````
     ![](images/num4.png)   

5.  This looks at the USER_TABLES view and queries attributes of tables in the SSB schema.  

    ````
    <copy>
    set pages 999
    column table_name format a12
    column cache format a5;
    column buffer_pool format a11;
    column compression heading 'DISK|COMPRESSION' format a11;
    column compress_for format a12;
    column INMEMORY_PRIORITY heading 'INMEMORY|PRIORITY' format a10;
    column INMEMORY_DISTRIBUTE heading 'INMEMORY|DISTRIBUTE' format a12;
    column INMEMORY_COMPRESSION heading 'INMEMORY|COMPRESSION' format a14;
    --QUERY    

    SELECT table_name, cache, buffer_pool, compression, compress_for, inmemory,
        inmemory_priority, inmemory_distribute, inmemory_compression
    FROM   user_tables;
    </copy>
    ````
     ![](images/step2num5.png)   

By default the IM column store is only populated when the object is accessed.

6.  Let's populate the store with some simple queries.

    ````
    <copy>
    SELECT /*+ full(d)  noparallel (d )*/ Count(*)   FROM   date_dim d;
    SELECT /*+ full(s)  noparallel (s )*/ Count(*)   FROM   supplier s;
    SELECT /*+ full(p)  noparallel (p )*/ Count(*)   FROM   part p;
    SELECT /*+ full(c)  noparallel (c )*/ Count(*)   FROM   customer c;
    SELECT /*+ full(lo)  noparallel (lo )*/ Count(*) FROM   lineorder lo;
    </copy>
    ````
     ![](images/step2num6.png)

7. Background processes are populating these segments into the IM column store.  To monitor this, you could query the V$IM_SEGMENTS.  Once the data population is complete, the BYTES_NOT_POPULATED should be 0 for each segment.  

    ````
    <copy>
    column name format a20
    column owner format a15
    column segment_name format a30
    column populate_status format a20
    column bytes_in_mem format 999,999,999,999,999
    column bytes_not_populated format 999,999,999,999,999
    --QUERY

    SELECT v.owner, v.segment_name name, v.populate_status status, v.bytes bytes_in_mem, v.bytes_not_populated
    FROM v$im_segments v;
    </copy>
    ````

     ![](images/lab4step7.png)

8.  Now let's check the total space usage.

    ````
    <copy>
    column alloc_bytes format 999,999,999,999;
    column used_bytes      format 999,999,999,999;
    column populate_status format a15;
    select * from v$inmemory_area;
    exit
    </copy>
    ````

    ![](images/part1step8a.png)

    ![](images/part1step8b.png)

In this Step you saw that the IM column store is configured by setting the initialization parameter INMEMORY_SIZE. The IM column store is a new static pool in the SGA, and once allocated it can be resized dynamically, but it is not managed by either of the automatic SGA memory features.

You also had an opportunity to populate and view objects in the IM column store and to see how much memory they use. In this Lab we populated about 1471 MB of compressed data into the  IM column store, and the LINEORDER table is the largest of the tables populated with over 23 million rows.  Remember that the population speed depends on the CPU capacity of the system as the in-memory data compression is a CPU intensive operation. The more CPU and processes you allocate the faster the populations will occur.

Finally you got to see how to determine if the objects were fully populated and how much space was being consumed in the IM column store.

You may now proceed to the next lab.

## Acknowledgements

- **Author** - Andy Rivenes, Sr. Principal Product Manager, Oracle Database In-Memory
- **Last Updated By/Date** - Kay Malcolm, Director, DB Product Management, March 2020

See an issue?  Please open up a request [here](https://github.com/oracle/learning-library/issues).