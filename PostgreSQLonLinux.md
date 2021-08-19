# PostgreSQL with EXPRESSCLUSTER X on Linux

## About This Guide

This guide describes how to setup PostgreSQL with EXPRESSCLUSTER X (ECX).

## Configuration

2 nodes mirror disk type cluster will be created. The database that is saved on the ECX data partition will be replicated between 2 servers.

In this guide, mirror disk type cluster is focused, but shared disk type clusters can also be crated.
The tested versions and configurations are as follows.

### Software versions
- CentOS Linux release 7.8.2003 (Core)
  - Kernel version 3.10.0-1127.el7.x86_64
- PostgreSQL 13.3/10.6
- EXPRESSCLUSTER X 4.3 for Linux (internal version: 4.3.0-1)

#### Cluster configuration
- Group resource
	- Floating IP address resource
	- Mirror disk resource
	- Exec resource

- Monitor resource
	- Floating IP monitor resource
	- Mirror connect monitor resource
	- Mirror disk monitor resource
    - PostgreSQL monitor resource
	- User-mode monitor resource

## Setup

### Setup a basic cluster
##### On both servers
1. Install ECX.
1. Register licenses.
    - EXPRESSCLUSTER X for Linux
    - EXPRESSCLUSTER X Replicator for Linux
    - EXPRESSCLUSTER X Database Agent for Linux
1. Reboot the server.

##### On Primary server
1. Create a cluster.
1. Add one failover group with fip and md resources.
    - failover group
        - fip
        - md
            - Mount Point: /mnt/md1
            - Data Partition: /dev/sdb2
            - Cluster Partition: /dev/sdb1
- Apply the configuration.
- Start the failover group on Primary server.

### User settings for PostgreSQL
##### On both servers
1. Create a user for PostgreSQL.
    - **The user ID must be the same on both servers.** You can specify the user ID with **-u** option.

        ```
        # useradd -u 26 postgres
        ```
1. Confirm that the user ID, group ID and group name is the same on both servers.

    ```
    # id postgres
    ```
1. Set the password of the user.

    ```
    # passwd postgres
    ```

### Install PostgreSQL

If PostgreSQL has already been installed on both servers, you can skip.

##### On both servers
1. Download PostgerSQL rpm file and copy them to the server.
    - https://yum.postgresql.org/rpmchart/
    - e.g.
        - postgresql13-13.3-1PGDG.rhel7.x86_64
        - postgresql13-contrib-13.3-1PGDG.rhel7.x86_64
        - postgresql13-libs-13.3-1PGDG.rhel7.x86_64
        - postgresql13-server-13.3-1PGDG.rhel7.x86_64
1. Install the rpms.

    ```
    # rpm -ivh postgresql13-libs-13.3-1PGDG.rhel7.x86_64
    # rpm -ivh postgresrpm/postgresql13-13.3-1PGDG.rhel7.x86_64.rpm
    # rpm -ivh postgresrpm/postgresql13-server-13.3-1PGDG.rhel7.x86_64.rpm
    # rpm -ivh postgresrpm/postgresql13-contrib-13.3-1PGDG.rhel7.x86_64.rpm
    ```

### PostgreSQL setup
##### On Primary server
1. Create a directory for the database on the mirror disk data partition.

    ```
    # mkdir -p /mnt/md1/pgsql/data
    ```
1. If you are already using PostgreSQL and the data is in default directory, you need to move the data to ECX mirror disk.
    1. Check the current/existing location of PostgreSQL Data Directory.

        ```
        > psql
        postgres=# SHOW data_directory;
           data_directory
        ---------------------
        /var/lib/pgsql/13/data
        (1 row)
        ```
    1. Stop the database server

        ```
        systemctl stop postgresql-13
        ```
        or
        ```
        $ /usr/pgsql-13/bin/pg_ctl stop -D /var/lib/pgsql/13/data -m fast
        ```
    1. Copy the data to ECX mirror disk.
        
        ```
        cp -r /var/lib/pgsql/13/data/* /mnt/md1/pgsql/data
        ```
    1. Update the data location in postgresql.conf if you were setting it.

        /mnt/md1/pgsql/data/postgresql.conf
        ```
        data_directory = '/mnt/md1/pgsql/data/'
        ```
1. Grant the full control of the data directory to the user for PostgreSQL.

    ```
    # chown -R postgres:postgres /mnt/md1/pgsql/data
    # chmod 700 /mnt/md1/pgsql/data
    ```
1. Login as **postgres** user.
1. Create a database cluster. **(If this is the first time you use PostgreSQL.)**

    ```
    $ /usr/pgsql-13/bin/initdb -D /mnt/md1/pgsql/data -E UTF8 --no-locale -W
    .
    .
    Enter new superuser password:
    Enter it again:
    .
    .
    Success. You can now start the database server using:

        /usr/pgsql-13/bin/pg_ctl -D /mnt/md1/pgsql/data -l logfile start
    ```
1. Edit postgresql.conf in the database location. (e.g. /mnt/md1/pgsql/data)
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
1. Edit pg_hba.conf in the database location. (e.g. /mnt/md1/pgsql/data)
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
1. Start the database server.

    ```
    $ /usr/pgsql-13/bin/pg_ctl start -D /mnt/md1/pgsql/data -l /dev/null
    ```
1. Create a database. **(If this is the first time you use PostgreSQL.)**

    ```
    $ /usr/pgsql-13/bin/createdb -h localhost -U postgres db1
    ```
1. Test the database connection.

    ```
    $ /usr/pgsql-13/bin/psql -h localhost -U postgres db1
    ```
1. Stop the database server.

    ```
    $ /usr/pgsql-13/bin/pg_ctl stop -D /mnt/md1/pgsql/data -m fast
    ```
1. Check the startup type of postgresql daemon.

    ```
    $ systemctl list-unit-files postgresql-13.service
    ```

    If STATE is enabled, login as root user and disable it.

    ```
    # systemctl disable postgresql-13.service
    ```

##### On Secondary server
1. Move the failover group to the secondary server.
1. Check the startup type of postgresql daemon.

    ```
    $ systemctl list-unit-files postgresql-13.service
    ```

    If STATE is enabled, login as root user and disable it.

    ```
    # systemctl disable postgresql-13.service
    ```
1. Login as **postgres** user.
1. Start the database server.

    ```
    $ /usr/pgsql-13/bin/pg_ctl start -D /mnt/md1/pgsql/data -l /dev/null
    ```
1. Test the database connection.

    ```
    $ /usr/pgsql-13/bin/psql -h localhost -U postgres db1
    ```
1. Stop the database server.

    ```
    $ /usr/pgsql-13/bin/pg_ctl stop -D /mnt/md1/pgsql/data -m fast
    ```

### Add resources for PostgreSQL to the cluster
##### On Primary server
1. Add one exec resource to the failover group.
    - [start.sh](#startsh)
        - Edit parameters.

            |Parameter|Value|Description|
            |----|----|----|
            |SUUSER|postgres|PostgreSQL user name|
            |PGINST|/usr/pgsql-13|PostgreSQL Install directory|
            |PGDATA|/mnt/sdb2/pgsql/data|Database directory|
            |PGPORT|5432|Database port number|
    - [stop.sh](#stopsh)
        - Edit parameters.

            |Parameter|Value|Description|
            |----|----|----|
            |SUUSER|postgres|PostgreSQL user name|
            |PGINST|/usr/pgsql-13|PostgreSQL Install directory|
            |PGDATA|/mnt/sdb2/pgsql/data|Database directory|
1. Add one PostgreSQL monitor resource to the cluster. (Database Agent license is required.)
    - Target Resource: exec resource
    - Database Name: db1
    - IP Address: 127.0.0.1
    - Port Number: 5432
    - User Name: postgres
    - Password: The password of postgres
    - Table: psqlwatch
    - Library Path: /usr/pgsql-13/lib/libpq.so.5.13
    - Recovery Target: exec resource
1. **(In case that you don't have Database Agent license)** Add one customer monitor resource instead of PostgreSQL monitor resource.

    This is an alternative option for the user that doesn't have Database Agent license. Please note that PostgreSQL monitor resource has a better monitoring capability than this customer monitor resource.
    
    - Target Resource: exec resource
    - [genw.sh](#genwsh)
        - Edit parameters.

            |Parameter|Value|Description|
            |----|----|----|
            |SUUSER|postgres|PostgreSQL user name|
            |PGINST|/usr/pgsql-13|PostgreSQL Install directory|
            |PGDATA|/mnt/sdb2/pgsql/data|Database directory|
    - Recovery Target: exec resource
1. Apply the configuration.
1. Start the exec resource.

### Sample scripts
#### `start.sh`
```sh
#! /bin/sh
#***************************************
#*              start.sh               *
#***************************************

ulimit -s unlimited

#
# PostgreSQL's Parameters
#
SUUSER="postgres"               # PG user name(OS user)
PGINST="/usr/pgsql-13"          # PG Install directory
PGDATA="/mnt/md1/pgsql/data"    # Database directory
PGPORT="5432"                   # Database Port number

if [ -f ${PGDATA}/postmaster.pid ]
then
    rm -f ${PGDATA}/postmaster.pid
fi

if [ -f /tmp/.s.PGSQL.${PGPORT} ]
then
    rm -f /tmp/.s.PGSQL.${PGPORT}
fi

if [ -f /var/run/postgresql/.s.PGSQL.${PGPORT} ]
then
    rm -f /var/run/postgresql/.s.PGSQL.${PGPORT}
fi


if [ "$CLP_EVENT" = "START" ]
then
    if [ "$CLP_DISK" = "SUCCESS" ]
    then
        echo "NORMAL1"
        if [ "$CLP_SERVER" = "HOME" ]
        then
            echo "NORMAL2"
        else
            echo "ON_OTHER1"
        fi
        date +"%Y/%m/%d %T"
        echo "PG Database start"

        su - ${SUUSER} -c "${PGINST}/bin/pg_ctl start -D ${PGDATA} -l /dev/null"
    else
        echo "ERROR_DISK from START"
        exit 1
        
    fi
elif [ "$CLP_EVENT" = "FAILOVER" ]
then
    if [ "$CLP_DISK" = "SUCCESS" ]
    then
        echo "FAILOVER1"
        if [ "$CLP_SERVER" = "HOME" ]
        then
            echo "FAILOVER2"
        else
            echo "ON_OTHER2"
        fi
        date +"%Y/%m/%d %T"
        echo "PG Database start"

        su - ${SUUSER} -c "${PGINST}/bin/pg_ctl start -D ${PGDATA} -l /dev/null"
    else
        echo "ERROR_DISK from FAILOVER"
        exit 1
    fi
else
    echo "NO_CLP"
    exit 1
fi
echo "EXIT"
exit 0
```

#### `stop.sh`
```sh
#! /bin/sh
#***************************************
#*               stop.sh               *
#***************************************

ulimit -s unlimited

#
# PostgreSQL's Parameters
#
SUUSER="postgres"               # PG user name(OS user)
PGINST="/usr/pgsql-13"          # PG Install directory
PGDATA="/mnt/md1/pgsql/data"    # Database directory

if [ "$CLP_EVENT" = "START" ]
then
	if [ "$CLP_DISK" = "SUCCESS" ]
	then
		echo "NORMAL1"
		if [ "$CLP_SERVER" = "HOME" ]
		then
			echo "NORMAL2"
		else
			echo "ON_OTHER1"
		fi
		date +"%Y/%m/%d %T"
        echo "PG Database stop"

        su - ${SUUSER} -c "${PGINST}/bin/pg_ctl stop -D ${PGDATA} -m fast"
	else
		echo "ERROR_DISK from START"
		exit 1
	fi
elif [ "$CLP_EVENT" = "FAILOVER" ]
then
	if [ "$CLP_DISK" = "SUCCESS" ]
	then
		echo "FAILOVER1"
		if [ "$CLP_SERVER" = "HOME" ]
		then
			echo "FAILOVER2"
		else
			echo "ON_OTHER2"
		fi
		date +"%Y/%m/%d %T"
        echo "PG Database stop"
        
        su - ${SUUSER} -c "${PGINST}/bin/pg_ctl stop -D ${PGDATA} -m fast"
	else
		echo "ERROR_DISK from FAILOVER"
		exit 1
	fi
else
	echo "NO_CLP"
	exit 1
fi
echo "EXIT"
exit 0
```

#### `genw.sh`
```sh
#! /bin/sh
#***********************************************
#*                   genw.sh                   *
#***********************************************

ulimit -s unlimited

#
# PostgreSQL's Parameters
#
SUUSER="postgres"               # PG user name(OS user)
PGINST="/usr/pgsql-13"          # PG Install directory
PGDATA="/mnt/md1/pgsql/data"    # Database directory

su - ${SUUSER} -c "${PGINST}/bin/pg_ctl status -D ${PGDATA}"

exit $?
```