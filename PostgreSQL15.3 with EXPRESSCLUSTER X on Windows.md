PostgreSQL 15.3 Quick Start Guide for EXPRESSCLUSTER X on Windows Platform
===

About this guide
---
This guide describes how to setup PostgreSQL 15.3 with EXPRESSCLUSTER X 5.1.

For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html) .


## **System Overview**

### **System Requirement**
- 2 servers
  - IP reachable each other
  - Having mirror disk
    - At least 2 partitions are required on each mirror disk.
    - Cluster partition size depends on ECX version.
    - X 5.1 or later: 1024MB
    - Data partition size depends on Database sizing.
- PostgreSQL Database are installed on local partition on each server.

### **System Configuration**
- Windows Server 2022 Standard
- PostgreSQL 15.3
- EXPRESSCLUSTER X 5.1

        Sample configuration
		<LAN>
		 |  +--------------------------+
		 |  | Primary Server           |
		 |  | - Wincluster7 (Win2k22)  |
		 |  | - PostgreSQL15.3         |
		 |  | - EXPRESSCLUSTER X 5.1   |
		 |  | IP Address:10.0.7.87     |
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
		 |  | - Wincluster8 (Win2k22)  |
		 |  | - PostgreSQL15.3         |
		 |  | - EXPRESSCLUSTER X 5.1   |
		 |  | IP Address:10.0.7.88     |
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
		- Fip                       : Floating IP address resource
		- md                        : mirror disk resource
		- PostgreSQL_service        : service resource for PostgreSQL

- Monitor resource

	- Fipw1             : Floating IP monitor resource
	- mdw1              : mirror disk monitor resource
	- servicew1         : service monitor resource for PostgreSQL_service
	- userw             : user-mode monitor resource


## **Basic cluster setup on Primary and Secondary servers**


 ### **1. Install EXPRESSCLUSTER X (ECX)**
 ### **2. Register ECX licenses**
- EXPRESSCLUSTER X 5.1 for Windows
- EXPRESSCLUSTER X Replicator 5.1 for Windows

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
       | Startup Server       | wincluster7 -> wincluster8  |
       | Floating IP Address  | 10.0.7.189                  |
       | Mirror Disk resource | D:\PostgreSQL               |
       +----------------------+-----------------------------+

 If want to know how to add the resource, please refer to "EXPRESSCLUSTER X 5.1 for Windows Installation and Configuration Guide". 

 After you add failver group and execute apply the configuration file, you start failover group by wincluster7.  
     
### **5. Install PostgreSQL on both servers**

- Download and Install PostgreSQL15.3 on both the servers.
    
    - For download procedure please refer to [this site](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads).
        - postgresql-15.3-1-windows-x64.exe

### Install PostgreSQL On both servers

1. Login as an user with Administrator privilege and Execute the .exe file on the server.
1. Installation Directory
    - Specify the local directory (C: drive) as **Installation Directory**.
    - e.g. *C:\\Program Files\\PostgreSQL\\15*
1. Select Components
    - Check all components.
1. Data Directory
    - Specify the local directory (C: drive) as **Data Directory**.
    - e.g. **C:\\Program Files\\PostgreSQL\\15\\data**
1. Password
    - Input the password to login to PostgreSQL
1. Port
    - e.g. **5432**
1. Advanced Options
    - e.g. **C: Drive**
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
- Stop postgresql-x64-15 service from services.msc.
- Copy MySQL data from default directory (C:\ProgramData\MySQL\MySQL Server 8.0\MySQL) to Mirror disk    (D:\PostgreSQL)
- Right Click on the newly created folder and make sure that it has all type permissions for local postgres system user.      

### **7. Perform below steps on both the Nodes after moving group failover.**

 - Open Windows Registry Editor and go to below path of system's registry.
    > “Computer\HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\postgresql-x64-15”.  

 - Double click on “ImagePath” and change the default path of the data Directory after”-D” 
    > “C:\Program Files\PostgreSQL\15\bin\pg_ctl.exe" runservice -N "postgresql-x64-15" -D "D:\PostgreSQL\data" -w”.
    
 - Close all the open windows and restart the PostgreSQL Service.Validate new location using below command.
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
Postgres-# \l
                                                        List of databases
Name|  Owner   | Encoding |      Collate       |       Ctype        | ICU Locale | Locale Provider |Access privileges
-----------+----------+----------+--------------------+--------------------+------------+-----------------+--------
 postgres  | postgres | UTF8     | English_India.1252 | English_India.1252 |     |  ibc  |					       |
 template0 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
 template1 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
(3 rows)																					                       |
--------------------------------------------------------------------------------------------------------------------
```         
 
            
- **Create Database in Postgres SQL shell for testing database on Node1.**

 ```
postgres=# create database testdemo;
CREATE DATABASE
```

- **Now Enter command \l to varify the created database in postgreSQL.**

```
Postgres-# \l
                                                        List of databases
Name|  Owner   | Encoding |      Collate       |       Ctype        | ICU Locale | Locale Provider |Access privileges
-----------+----------+----------+--------------------+--------------------+------------+-----------------+--------
 Testdemo  | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc                            |
 postgres  | postgres | UTF8     | English_India.1252 | English_India.1252 |     |  ibc  |					       |
 template0 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
 template1 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
(4 rows)																					                       |
--------------------------------------------------------------------------------------------------------------------
```    


### **12. You have to move failover group on the Node2 for testing postgreSQL database failover.**

         
- Check and Confirm the database on postgreSQL which we have created on node1.

```
    postgres=#show data_directory;
    +--------------------+
    | data_directory     |
    +--------------------+
    | D:/PostgreSQL/data |
    +--------------------+
    1 row

    Postgres-# \l
                                                        List of databases
Name|  Owner   | Encoding |      Collate       |       Ctype        | ICU Locale | Locale Provider |Access privileges
-----------+----------+----------+--------------------+--------------------+------------+-----------------+--------
 Testdemo  | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc                            |
 postgres  | postgres | UTF8     | English_India.1252 | English_India.1252 |     |  ibc  |					       |
 template0 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
 template1 | postgres | UTF8     | English_India.1252 | English_India.1252 |     | libc  | =c/postgres          +  |
           |          |          |                    |                    |     |       | postgres=CTc/postgres   |
(4 rows)																					                       |
--------------------------------------------------------------------------------------------------------------------
```

   - The data is replicated from primary to secondary server successfully.   
                 
### **13. Verification**

- Confirm that we can access the database where it failover group is running.
