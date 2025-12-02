EXPRESSCLUSTER X with PostgreSQL 18.0-2 on Windows Platform Quick Start Guide
===

About this guide
---
This guide describes how to setup PostgreSQL 18.0-2 with EXPRESSCLUSTER X 5.3.

For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html) .


## **System Overview**

### **System Requirement**
- 2 servers
  - IP reachable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
    - Cluster partition size depends on ECX version.
    - X 5.3 or later: 1024MB
    - Data partition size depends on Database sizing.
- PostgreSQL Database are installed on local partition on each server.

### **System Configuration**
- Windows Server 2025 Standard
- PostgreSQL 18.0-2
- EXPRESSCLUSTER X 5.3

        Sample configuration
		<LAN>
		 |  +--------------------------+
		 |  | Primary Server           |
		 |  | - Winecxcl01 (Win2k25)   |
		 |  | - PostgreSQL18.0-2       |
		 |  | - EXPRESSCLUSTER X 5.3   |
		 |  | IP Address:127.0.0.1     |
		 |  | RAM   : 6GB              |
		 |  | Disk 0: 50GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 30GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +-----------+--------------+
		 |              |
		 |              | Mirroring
		 |              |
		 |  +-----------+--------------+
		 |  | Secondary Server         |
		 |  | - Winecxcl02 (Win2k25)   |
		 |  | - PostgreSQL18.0-2       |
		 |  | - EXPRESSCLUSTER X 5.3   |
		 |  | IP Address:127.0.0.2     |
         |  | RAM   : 6GB              |
		 |  | Disk 0: 50GB OS          |
		 |  |      C: local partition  |
		 |  | Disk 1: 30GB mirror disk |
		 |  |      E: cluster partition|
		 |  |      D: mirror partition |
		 |  +--------------------------+
		 |

#### Cluster configuration
- Network Partition Resolution resource (PING method) : `pingnp1`

- failover Group: `PostgreSQLfailover`
	- Group resource
		- FIP                       : Floating IP address resource
		- MD                        : mirror disk resource
		- PostgreSQL_service        : service resource for PostgreSQL

- Monitor resource

	- Fipw1             : Floating IP monitor resource
	- mdw1              : mirror disk monitor resource
	- servicew1         : service monitor resource for PostgreSQL_service
	- userw             : user-mode monitor resource


## **Basic cluster setup on Primary and Secondary servers**


 ### **1. Install EXPRESSCLUSTER X (ECX)**
 ### **2. Register ECX licenses**
- EXPRESSCLUSTER X 5.3 for Windows
- EXPRESSCLUSTER X Replicator 5.3 for Windows

### **3. Create a ECX cluster with failover group On Primary server**
    - Network partition: 
        pingnp1: PING method
    - Group:
        Postgresqlfailover
        Fip: Floating IP resource
        md : mirror disk resource
        
### **4. Start group on Primary server**

      
       +----------------------+-----------------------------+
       | Parameter            | Value                       |
       +----------------------+-----------------------------+
       | Name                 | PostgreSqlFailover          |
       | Startup Server       | Winecxcl01 -> Winecxcl02    |
       | Floating IP Address  | 127.0.0.3                   |
       | Mirror Disk resource | D:\PostgreSQL               |
       +----------------------+-----------------------------+

 Please refer to EXPRESSCLUSTER X 5.3 for Windows Installation and Configuration Guide if want to know how to add the  resources. (https://www.nec.com/en/global/prod/expresscluster/en/doc/manuals/W53_IG_EN_02.pdf) .

 After you add failover group and execute apply the configuration file, you start failover group on Node1 (Winecxcl01).  
     
### **5. Install PostgreSQL on both servers**

- Download and Install PostgreSQL18.0-2 on both the servers.
    
    - For download procedure please refer to [this site](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads).
        - postgresql-18.0-2-1-windows-x64.exe

### Install PostgreSQL on both servers

1. Login as an user with Administrator privilege and Execute the .exe file on the server.
1. Installation Directory
    - Specify the local directory (C: drive) as **Installation Directory**.
    - e.g. *C:\Program Files\PostgreSQL\18*
1. Select Components
    - Check all components.
1. Data Directory
    - Specify the local directory (C: drive) as **Data Directory**.
    - e.g. **C:\Program Files\PostgreSQL\18\data**
1. Password
    - Input the password to login to PostgreSQL
1. Port
    - e.g. **5432**
1. Database Superuser
    - e.g. **postgres**
1. Pre Installation Summary
    - Click **Next**
1. Ready to Install
    - Click **Next**
1. Completing the PostgreSQL Setup Wizard
    - Uncheck the check box and click **Finish**
    - If you need to install additional tools, please keep it checked and click Finish.


### **6. PostgreSQL Configuration for Mirror disk (Node1)**

- Create the database directory.
- Create data directory in mirror drive e.g. D:\PostgreSQL
- Stop postgresql-x64-18 service from services.msc.
- Copy PostgreSQL data from default directory (C:\Program Files\PostgreSQL\18\data) to Mirror disk    (D:\PostgreSQL)
- Right Click on the newly created folder and make sure that it has all type permissions for local postgres system user.      

### **7. Perform below steps on both the Nodes after moving group failover.**

 - Open Windows Registry Editor and go to below path of system's registry.
    > “Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\postgresql-x64-18”.  

 - Double click on “ImagePath” and change the default path of the data Directory after”-D” 
    > “C:\Program Files\PostgreSQL\18\bin\pg_ctl.exe" runservice -N "postgresql-x64-18" -D "C:\Program Files\PostgreSQL\18\data" -w”.
    
 - Close all the open windows and restart the PostgreSQL Service.Validate new location using below command.

 - Click on Start Tab and Open the SQL Shell (psql).
      - Enter password:- root@123 (Administrative Credential)
      - Press enter five times to connect to the DB
      - Enter the command Verify the location of database 

    ```
    > postgres=# show data_directory;
        - Data Directory.
        - D:/PostgreSQL/data
    ```

 
### **8. PostgreSQL Setup (Node1)**
        
- Start the MySQL service to check it is working fine.
    
        - From services.msc start the PostgreSQL Service.
        - From services.msc stop the PostgreSQL Service.

### **9. Change the startup type of PostgreSQL Service (Node1 & Node2)**

- Open the Windows Service Manager     
  - On Command Prompt
        > services.msc
- Change the startup type of PostgreSQL Service to Manual.

### **10. Configure PostgreSQL Service in ECX Cluster**
 
  - Add the service resource to control PostgreSQL and configure in Config mode of the Cluster WebUI.  
- In Config mode of the Cluster WebUI, execute Apply the Configuration File.

    
      
### **11. Testing Of PostgreSQL Database on ECX Cluster**

- PostgreSQL Setup (Node1)
        
    - Click on Start Tab and Open the SQL Shell.
      - Enter password:- root@123 (Administrative Credential)
      - Press enter five times to connect to the DB
      - Enter the command Verify the location of database 
         
```
    postgres=#show data_directory;
    +--------------------+
    | data_directory     |
    +--------------------+
    | D:/PostgreSQL/data |
    +--------------------+
    1 row
```

-   Enter command \l to get a list of all databases

```

postgres=# \l
                                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |          Collate           |           Ctype            | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+----------------------------+----------------------------+--------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           |
 template0 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
(3 rows)
--------------------------------------------------------------------------------------------------------------------
```         
 
            
- **Create Database in Postgres SQL shell for testing database on Node1.**

 ```
postgres=# create database demo1;
CREATE DATABASE
```

- **Now Enter command \l to verify the created database in postgreSQL.**

```
postgres=# \l
                                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |          Collate           |           Ctype            | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+----------------------------+----------------------------+--------+-----------+-----------------------
 demo1     | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           |
 postgres  | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           |
 template0 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
 (4 rows)																	                       |
--------------------------------------------------------------------------------------------------------------------
```    


### **12. Move failover group on the Node2 for testing postgreSQL database failover.**

         
- Check and Confirm the database on postgreSQL which we have created on node1.

```
    postgres=#show data_directory;
    +--------------------+
    | data_directory     |
    +--------------------+
    | D:/PostgreSQL/data |
    +--------------------+
    1 row

 postgres=# \l
                                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |          Collate           |           Ctype            | Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+----------------------------+----------------------------+--------+-----------+-----------------------
 demo1     | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           |
 postgres  | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           |
 template0 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | English_United States.1252 | English_United States.1252 |        |           | =c/postgres          +
           |          |          |                 |                            |                            |        |           | postgres=CTc/postgres
 (4 rows)																	                       |
--------------------------------------------------------------------------------------------------------------------
```

   - The data is replicated from primary to secondary server successfully and vise versa.  
                 
### **13. Verification**

- Confirm that we can access the database where it failover group is running.
