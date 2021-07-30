# PostgreSQL with EXPRESSCLUSTER X on Windows

## About This Guide

This guide describes how to setup PostgreSQL with EXPRESSCLUSTER X (ECX).

## Configuration

2 nodes mirror disk type cluster will be created. The database that is saved on the ECX data partition will be replicated between 2 servers.

In this guide, mirror disk type cluster is focused, but shared disk type clusters can also be crated.
The tested versions and configurations are as follows.

### Software versions
- Windows Server 2019
- PostgreSQL 13.3/10.6
- EXPRESSCLUSTER X 4.3 for Windows (internal version: 12.30)

#### Cluster configuration
- Group resource
	- Floating IP address resource
	- Mirror disk resource
	- Service resource

- Monitor resource
	- Floating IP monitor resource
	- Mirror connect monitor resource
	- Mirror disk monitor resource
    - PostgreSQL monitor resource
    - Service monitor resource
	- User-mode monitor resource

## Setup

### Setup a basic cluster
##### On both servers
1. Install ECX.
1. Register licenses.
    - EXPRESSCLUSTER X for Windows
    - EXPRESSCLUSTER X Replicator for Windows
    - EXPRESSCLUSTER X Database Agent for Windows
1. Reboot the server.

##### On Primary server
1. Create a cluster.
1. Add one failover group with fip and md resource.
    - failover group
        - fip
        - md
            - Data Partition: F
            - Cluster Partition: E
- Apply the configuration.
- Start the failover group on Primary server.

### Install PostgreSQL
##### On both servers
1. Download PostgerSQL exe file and copy them to the servers.
    - https://www.enterprisedb.com/downloads/postgres-postgresql-downloads
    - e.g.
        - postgresql13.3-2-windows-x64.exe
1. Login as an user with Administrator privilege.
1. Execute the exe file.
1. Installation Directory
    - Specify the local directory (C drive) as **Installation Directory**.
    - e.g. *C:\\Program Files\\PostgreSQL\\13*
1. Select Components
    - Check all components.
1. Data Directory
    - Specify the local directory (C drive) as **Data Directory**.
    - e.g. *C:\\Program Files\\PostgreSQL\\13\\data*
1. Password
    - Input the password to login to PostgreSQL
1. Port
    - e.g. *5432*
1. Advanced Options
    - e.g. *C*
1. Pre Installation Summary
    - Click **Next**
1. Ready to Install
    - Click **Next**
1. Completing the PostgreSQL Setup Wizard
    - Uncheck the check box and click **Finish**
    - If you need to install additional tools, please keep it checked and click Finish.
1. After PostgreSQL installation, open Service Manager.

    On Command Prompt
    ```
    > services.msc
    ```
1. Stop PostgreSQL service.
1. Change the startup type of PostgreSQL to **Disabled**.

### PostgreSQL setup
##### On Primary server
1. Create a directory for the database on the mirror disk data partition.
    - e.g. *F:\\data*
1. Create a database cluster.

    ```
    >"C:\Program Files\PostgreSQL\13\bin\initdb" -D F:\data -E UTF8 --no-locale -U postgres -W
    .
    .
    Enter new superuser password:
    Enter it again:
    .
    .
    Success. You can now start the database server using:

        ^"C^:^\Program^ Files^\PostgreSQL^\13^\bin^\pg^_ctl^" -D ^"F^:^\data^" -l logfile start
    ```
1. Edit postgresql.conf in the database location. (e.g. F:\\data)
    - By default, the database can be connected from localhost only. If you want to allow connection from other servers, edit as follows.
    
        Before
        ```
        #listen_addresses = 'localhost'         # what IP address(es) to listen on;
        ```
        After
        ```
        listen_addresses = '*'         # what IP address(es) to listen on;
        ```
    - By default, the database listens for connections on port 5432. If you change, edit the following line.

        ```
        #port = 5432                            # (change requires restart)
        ```
1. Edit pg_hba.conf in the database location. (e.g. F:\\data)
    - ECX PostgreSQL monitor resource connects to the database with md5 authentication method. You need to edit this file as follows.

        Before
        ```
        # IPv4 local connections:
        host    all             all             127.0.0.1/32            trust
        ```
        After
        ```
        # IPv4 local connections:
        host    all             all             127.0.0.1/32            md5
        ```
    - Add lines if you have any clients that connect to the database.

        e.g.
        ```
        host    all             all             192.168.1.0/24          md5
        ```
1. Create a PostgreSQL service.
    1. If you are opening Service Manager, close it.
    1. Execute the pg_ctl register command on Command Prompt.

        e.g.
        ```
        > "C:¥Program Files¥PostgreSQL¥13¥bin¥pg_ctl" register –D F:¥data –N pgsql-13 –S demand –U "NT AUTHORITY¥NetworkService"
        ```
    1. Check the service property. It should be as follows.
        - Path to executable: *"C:¥Program Files¥PostgreSQL¥13¥bin¥pg_ctl.exe" runservice -N "pgsql-13" -D "F:¥data" -w*
        - Startup Type: *Manual*
1. Grant the access permission of the database directory to NETWORK SERVICE account.
    1. Right-click the database directory (e.g. F:\\data) on File Explorer, and open the **Properties**.
    1. In **Security** tab, click **Edit**.
    1. Add **NETWORK SERVICE** account.
    1. Grant **Full Control** permission to **NETWORK SERVICE**.
1. Start the PostgreSQL service on Service Manager.
    - e.g. pgsql-13
1. Create a database.

    e.g.
    ```
    > "C:\Program Files\PostgreSQL\13\bin\createdb" -h localhost -U postgres db1
    ```
1. Stop the PostgreSQL service on Service Manager.

##### On Secondary server
1. Move the failover group to Secondary server.
1. Create a PostgreSQL service.
    1. If you are opening Service Manager, close it.
    1. Execute the pg_ctl register command on Command Prompt.

        e.g.
        ```
        > "C:¥Program Files¥PostgreSQL¥13¥bin¥pg_ctl" register –D F:¥data –N pgsql-13 –S demand –U "NT AUTHORITY¥NetworkService"
        ```
    1. Check the service property. It should be as follows.
        - Path to executable: *"C:¥Program Files¥PostgreSQL¥13¥bin¥pg_ctl.exe" runservice -N "pgsql-13" -D "F:¥data" -w*
        - Startup Type: *Manual*

### Add resources for PostgreSQL to the cluster
##### On both servers
1. *(If you will use PostgreSQL monitor resource,)* Add the path of **libpq.dll** to **Path** of **Environment Variables: System Variables**.
    - e.g. *C:\Program Files\PostgreSQL\13\bin* and *C:\Program Files\PostgreSQL\13\lib*

##### On Primary server
1. *(If you will use PostgreSQL monitor resource,)* Stop the cluster on WebUI.
1. Add one service resource to the failover group.
    - Service Name: pgsql-13
1. Add one PostgreSQL monitor resource to the cluster.
    - Target Resource: The service resource for PostgreSQL
    - Database Name: db1
    - IP Address: 127.0.0.1
    - Port Number: 5432
    - User Name: postgres
    - Password: The password of PostgreSQL
    - Monitor Table Name: PSQLWATCH
    - Recovery Target: The service resource for PostgreSQL
1. Apply the configuration.
1. Start the cluster or resource on WebUI.

### Database connection test

On Command Prompt
```
> "C:¥Program Files¥PostgreSQL¥13¥bin¥psql" -h <IP address or hostname> -U postgres db1
```