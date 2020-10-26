**Contents**
- [Migrating Oracle to PostgreSQL hands-on lab step-by-step](#migratingoracletopostgresql-hands-on-lab-step-by-step)
  - [Requirements](#requirements)
  - [Exercise 1: Setup Oracle 11g Express Edition](#exercise-1-setup-oracle-11g-express-edition)
    - [Task 1: Download ora2pg image](#task-1-download-ora2pg-image)
    - [Task 2: Setup Oracle](#task-2-download-oracle-image)
  - [Exercise 2: Prepare to Migrate the Oracle database to PostgreSQL](#exercise-3-prepare-to-migrate-the-oracle-database-to-postgresql)
    - [Task 1: Create Azure Resources](#task-1-create-azure-resources)
    - [Task 2: Configure the PostgreSQL server instance](#task-2-configure-the-postgresql-server-instance)
    - [Task 3: Install pgAdmin](#task-3-install-pgadmin)
    - [Task 5: Prepare the PostgreSQL instance](#task-5-prepare-the-postgresql-instance)
    - [Task 6: Create an ora2pg project structure](#task-6-create-an-ora2pg-project-structure)
    - [Task 7: Create a migration report](#task-7-create-a-migration-report) 
  - [Exercise 3: Migrate the Database](#exercise-4-migrate-the-database)
    - [Task 1: Migrate the basic database table schema using ora2pg](#task-1-migrate-the-basic-database-table-schema-using-ora2pg)
    - [Task 2: Use Azure Database Migration Service to migrate table data](#task-2-use-azure-database-migration-service-to-migrate-table-data)
    - [Task 3: Finishing the table schema migration](#task-3-finishing-the-table-schema-migration)
    - [Task 4: Migrate Views](#task-4-migrate-views)
    - [Task 5: Migrate the Stored Procedure](#task-5-migrate-the-stored-procedure)
    - [Task 6: Create new Entity Data Models and update the application on the Lab VM](#task-6-create-new-entity-data-models-and-update-the-application-on-the-lab-vm)
  - [After the hands-on lab](#after-the-hands-on-lab)
    - [Task 1: Delete the resource group](#task-1-delete-the-resource-group)

# Migrating Oracle to PostgreSQL hands-on lab step-by-step

## Requirements

- Microsoft Azure subscription must be pay-as-you-go or MSDN.
  - Trial subscriptions will not work.
## Exercise 1: Setup Oracle 11g Express Edition

### Task 1: Setup ora2pg
![Setup ora2pg](/Media/SetupOra2pg.gif "Setup Ora2pg") 

[ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)

1. Connect to your Lab VM. 

2. Switch to root user:

       sudo -i
3. From root user run the following commands:

        dnf update -y
        dnf install podman -y

        podman pull georgmoser/ora2pg-docker

        mkdir /data
        
4. Get inside container:

        podman run -it --privileged -v /data:/data georgmoser/ora2pg-docker /bin/bash
        
5. **Ora2pg** allows database objects to be exported in multiple files so that is simple to organize and review changes. This command will create the project structure that will make it easy to do this: 

        ora2pg --project_base /data --init_project myproject
        
6. Log out from the container (Ctrl+d)
        

        
### Task 2: Setup Oracle
![Setup Oracle](/Media/Setup-Oracle.gif "Setup Oracle") 

[Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)

There are two ways of doing this 1) Own docker image, with custom password 2) Get the prebuilt image from docker hub. In this lab, we will use the option 2.

1. Download the image:

        podman pull araczkowski/oracle-apex-ords
2. Run the container based on prebuild image from docker with 8080, 1521, 22 ports opened:

         podman run -d --name oracle -p 49160:22 -p 8080:8080 -p 1521:1521 araczkowski/oracle-apex-ords    
         
3. Connect to Oracle VM to check the ip (password: secret):

         ssh root@localhost -p 49160
         
4. Check IP address of Oracle VM:

        ip a
        
![Checking ip of Oracle VM.](/Media/OracleIP.png "ip a")

Connect database with the following settings:

        hostname: <Output of ip a command>
        port: 1521
        sid: xe
        username: system
        password: secret
        
5. Log out of the Oracle VM and change ORACLE_DSN in ora2pg.conf file:

        vi /data/myproject/config/ora2pg.conf
        
![Changing Oracle DSN.](/Media/OracleConnection.png "Changing Oracle DSN")

6. Check if you are able to connect to Oracle database from ora2pg tool:

        podman run -it --privileged -v /data:/data georgmoser/ora2pg-docker ora2pg -c /data/myproject/config/ora2pg.conf -t SHOW_VERSION

![Test Connection.](/Media/OracleConnectTest.png "Test connection")       
        
7. Optionally you might want to change the password for system user since the current one will expire in 7 days:

        alter user system identified by secret;
        


## Exercise 2: Export Oracle schema and data to VM
Ora2pg provides a script that allows you to export database schema in one simple step. The script was generated into your migration
project.
1. From your test VM run ora2pg container:
        
        podman run -it --privileged -v /data:/data georgmoser/ora2pg-docker /bin/bash
        
2. Navigate to your project:
        
        cd /data/myproject/
        
3. When you list the content of the directory you will see export_schema.sh. Let's run the script:

        ./export_schema.sh
        
4. When schema export will finish ora2pg will display a command to migrate data. Just copy and paste it to the terminal:

        ora2pg -t COPY -o data.sql -b ./data -c ./config/ora2pg.conf
        
 You should see the following:
 
 ![Data Export.](/Media/DataExport.png "Data Export")         

5. Exit the container, for instance by using Ctrl+d shortcut.

6. /data/myproject directory contains now both - the migrated schema and data downloaded from Oracle and is ready to be 
deployed on Postgres.

![migrated Project.](/Media/migratedProject.png "Migrated Project")  
 
![Data Export.](/Media/countriesData.png "Data Export")  

## Exercise 2: Prepare to Migrate the Oracle database to PostgreSQL

In this exercise, you will configure Azure Database for PostgreSQL and Azure App Service, install and configure ora2pg and pgAdmin, and create an assessment report that outlines the difficulty of the migration process.

### Task 1: Create Azure Resources

We need to create a PostgreSQL instance and an App Service to host our application. Visual Studio integrates well with Microsoft Azure, simplifying application deployment.

1. Just as you configured resources in **Before the HOL**, you will need to navigate to the **New** page accessed by selecting **+ Create a resource**. Then, navigate to **Databases** under the **Azure Marketplace** section. Select **Azure Database for PostgreSQL**.

    ![Navigating Azure Marketplace to Azure Database for PostgreSQL, which has been highlighted.](/Media/migrate15.png "Azure Database for PostgreSQL")

2. There are two deployment options: **Single Server** and **Hyperscale (Citus)**. Single Server is best suited for traditional transactional workloads whereas Hyperscale is best suited for ultra-high-performance, multi-tenant applications. For our simple application, we will be utilizing a single server for our database.

    ![Screenshot of choosing the correct single server option.](/Media/migrate16.PNG "Single server")

3. Create a new Azure Database for PostgreSQL resource. Use the following configuration values:

   - **Resource group**: (same as Lab VM)
   - **Server name**: Enter a unique server name.
   - **Version**: 11
   - **Administrator username**: solldba
   - **Password**: (secure password)

    Select **Review + create** button once you are ready.

    ![Configuring the instance details.](/Media/migrate17.PNG "Project Details window with pertinent details")

### Task 2: Configure the PostgreSQL server instance

In this task, we will be modifying the PostgreSQL instance to fit our needs.

1. Storage Auto-growth is a feature in which Azure will add more storage automatically when required. We do not need it for our purposes so we will need to disable it. To do this, locate the PostgreSQL instance you created. Under the **Settings** tab, select **Pricing tier**.

    ![Changing the pricing tier in PostgreSQL instance.](/Media/migrate18.PNG "Pricing tier")

2. Find the **Storage Auto-growth** switch, and disable the feature. Select **OK** at the bottom of the page to save your change.

    ![Disabling storage auto-growth feature.](/Media/migrate19.PNG  "Storage auto-growth toggled to no")

3. Now, we need to implement firewall rules for the PostgreSQL database so we can access it. Locate the **Connection security** selector under the **Settings** tab.

    ![Configuring the Connection Security settings for the database.](/Media/migrate20.png "Connection security highlighted")

4. We will add an access rule. Since we are storing insecure test data, we can open the 0.0.0.0 to 255.255.255.255 range (all IPv4 addresses). Azure makes this option available. Press the **Save** button at the top of the page once you are ready.

    ![Adding IP addresses as an Access Rule](/Media/migrate21.png "IP Addresses highlighted")

    >**Note**: Do not use this type of rule for databases with sensitive data or in a production environment. You are allowing access from any IP address.

### Task 5: Prepare the file with libpq Environment Variables

In this task, we will create a file in our linux VM containing [environment variables] (https://www.postgresql.org/docs/current/libpq-envars.html) that will be used to select 
default connection parameter values to PostgreSQL PaaS instance. These are useful to be able to 
connect to postgres in a fast and convenient way without hard-coding connection string.

1. Go to the "Connection Strings" tab on the left hand side of the Azure Portal and copy psql connection string:

![Connection string](/Media/connectionString.png "Connection String")

2. From a terminal connect to Azure VM

3. Switch to root user:
        
        sudo -i
        
![Exporting libpq variables](/Media/exportingVariables.gif "Exporting libpq variables")

4. Create a new file

        vi .pg_azure

5. Add the following parameters.

        export PGDATABASE=ora2pg
        export PGHOST=HOSTNAME.postgres.database.azure.com
        export PGUSER=your_user@hostname
        export PGPASSWORD=your_password
        export PGSSLMODE=require

6. Read the content of the file in the current session:

        source .pg_azure
        
7. Create a new database in the postgres instance that will be our migration database:

        createdb ora2pg
        
8. Let's connect to our newly created Azure database with psql client:

        psql
        
        
 >**Note**: If you get an error (SELECT COUNT(*) FROM v$archived_log;) that there are no archive log run this command in oracle (ALTER SYSTEM ARCHIVE LOG CURRENT)
 
 
 
## Exercise 3: Migrate the Database

In this exercise, we will begin the migration of the database and the application. This includes migrating database objects, 
the data, application code, and finally, deploying to Azure App Service.

### Task 1: Migrate the basic database table schema using ora2pg

In this task, we will migrate the database table schema, using ora2pg and psql, which is a command-line utility that makes it easy to run SQL files against the database.

1. Exercise 3 covered planning and assessment steps.  To start the database migration, DDL statements must be created for all valid Oracle objects.

 
 
 
 
 
 
 
 
### Task 7: Create a migration report

The migration report tells us the "man-hours" required to fully migrate to our application and components. The report will provide the user with a relative complexity value. In this task, we will retrieve the migration report for our migration.


    >**Note**: In almost all migration scenarios, it is advised that table, index, and constraint schemas are kept in separate files. For data migration performance reasons, constraints should be applied to the target database only after tables are created and data copied. To enable this feature, open **config\ora2pg.conf** file. Set **FILE_PER_CONSTRAINT**, **FILE_PER_INDEX**, **FILE_PER_FKEYS**, and **FILE_PER_TABLE** to 1.


# Helpful Links:
[Full Windows-based workshop](https://github.com/microsoft/MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Migrating%20Oracle%20to%20PostgreSQL.md#task-4-install-ora2pg)

[Tutorial](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md)

[ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)

[Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)

[HR DDLScript - Import the HR sample](https://www.oracle.com/database/technologies/appdev/datamodeler-samples.html)

[Examples](https://www.darold.net/confs/ora2pg_the_hard_way.pdf)


# This lab was developed using CentOS 7

Setting up ora2pg:

```
sudo -i
```
From root user:
```
yum update -y
yum install docker -y

systemctl start docker
systemctl status docker

docker pull georgmoser/ora2pg-docker

mkdir /data
```
## Get inside container:
```
docker run -it --privileged -v /data:/data georgmoser/ora2pg-docker /bin/bash

ora2pg --version

apt-get update -y

apt-get install vim
```

Generate a migration project:

```
ora2pg --project_base /data --init_project myproject
```

change ORACLE_DSN in config file:

```
vi /data/myproject/config/ora2pg.conf

cd data/myproject/

```

### Discovery:

```
ora2pg -c config/ora2pg.conf -t SHOW_VERSION

ora2pg -c config/ora2pg.conf -t SHOW_SCHEMA 
```

Check which tables contain HR schema:

```
ora2pg -c config/ora2pg.conf -t SHOW_TABLE -n HR
```

List columns of table JOBS:

```
ora2pg -c config/ora2pg.conf -t SHOW_COLUMN -a 'TABLE[JOBS]' -n HR
```

### Generate report
```
ora2pg -c config/ora2pg.conf -t SHOW_REPORT --estimate_cost --dump_as_html -n HR > reports/report.html
```

## SCHEMA Migration
### Offline migration
Inside ```/data/myproject``` directory create a new directory:

```
mkdir offline
cd offline
```

Create a new file with following content:

```
vi countries.sql

CREATE TABLE COUNTRIES 
    ( 
     COUNTRY_ID CHAR (2 BYTE)  NOT NULL , 
     COUNTRY_NAME VARCHAR2 (40 BYTE) , 
     REGION_ID NUMBER 
    ) LOGGING 
;
```

now run ora2pg against this file:

```
root@13a8720887da:/data/myproject/offline# ora2pg -i countries.sql -t TABLE -c ../config/ora2pg.conf 
[========================>] 1/1 tables (100.0%) end of table export.

root@13a8720887da:/data/myproject/offline# ls
CONSTRAINTS_output.sql	INDEXES_output.sql  countries.sql  output.sql
```

Investigate the content of 3 files that has been created. 

### Online migration
Create online directory inside your project:
```
root@13a8720887da:/data/myproject/offline# mkdir /data/myproject/online
root@13a8720887da:/data/myproject/offline# cd /data/myproject/online
```

Get the sources of oracle procedures:
```
root@13a8720887da:/data/myproject/online# ora2pg -t PROCEDURE -o procedure.sql -c ../config/ora2pg.conf
[========================>] 2/2 procedures (100.0%) end of procedures export.
root@13a8720887da:/data/myproject/online# ls
ADD_JOB_HISTORY_procedure.sql  SECURE_DML_procedure.sql  procedure.sql
```

Investigate the content of generated files

Add ```-p``` to the command above to convert procedures to plpgsql:

```
root@13a8720887da:/data/myproject/online# ora2pg -p -t PROCEDURE -o procedure.sql -c ../config/ora2pg.conf
[========================>] 2/2 procedures (100.0%) end of procedures export.
root@13a8720887da:/data/myproject/online# ls
ADD_JOB_HISTORY_procedure.sql  SECURE_DML_procedure.sql  procedure.sql
```

The previous content has been overwritten. Check if conversion went well by exploring the content of the files.



### Schema export
Let run the export then!

change the name of the schema in ora2pg.conf file:

```
# Oracle schema/owner to use
SCHEMA  CHANGE_THIS_SCHEMA_NAME
```
So it looks as follows:

```
# Oracle schema/owner to use
SCHEMA  HR
```

Run the export. Make sure you are in the ```/data/myproject``` directory:

```
root@13a8720887da:/data/myproject# pwd
/data/myproject

root@13a8720887da:/data/myproject# ./export_schema.sh
```

Check the content of two directories:
* sources - where oracle sources are kept
* schema - with objects converted to PostgreSQL

## DATA Export
From ```/data/myproject``` run the following command:
```
ora2pg -t COPY -o data.sql -b ./data -c ./config/ora2pg.conf
```

Investigate the content of the ```data``` directory:
```
root@13a8720887da:/data/myproject# cd data/
root@13a8720887da:/data/myproject/data# ls
COUNTRIES_data.sql  DEPARTMENTS_data.sql  EMPLOYEES_data.sql  JOBS_data.sql  JOB_HISTORY_data.sql  LOCATIONS_data.sql  REGIONS_data.sql  data.sql
```

Direct export from Oracle to PostgreSQL
```
ora2pg -c config/ora2pg.conf -t COPY --pg_dsn "dbi:Pg:dbname=mydb;host=192.168.122.1;port=5432" --pg_user postgres
```



## PostgreSQL PaaS

* [Create an instance in Azure Database for PostgreSQL](https://docs.microsoft.com/azure/postgresql/quickstart-create-server-database-portal).
* Connect to the instance and create a database using the instruction in this [document](https://docs.microsoft.com/azure/postgresql/tutorial-design-database-using-azure-portal).

## Import to PostgreSQL PaaS
Create .pg_azure file with the credentials to your PaaS database:
```
vi .pg_azure

export PGDATABASE=ora2pg
export PGHOST=name.postgres.database.azure.com
export PGUSER=user@host
export PGPASSWORD=VeryBadPractice;)
```

Save the ```.pg_azure``` file and load it in the session:
```
source .pg_azure
```

Create a new database:
```
createdb ora2pg
```

From the ```/data/myproject``` directory load the files:
```
psql -f schema/tables/table.sql

psql -f schema/sequences/sequence.sql
psql -f schema/views/view.sql 
psql -f schema/procedures/procedure.sql
psql -f schema/triggers/trigger.sql
```

Check if all objects were correctly created in your new postgres database.

Import data:
```
psql -f data/data.sql
```
Create constraints:
```
psql -f schema/tables/INDEXES_table.sql
psql -f schema/tables/CONSTRAINTS_table.sql
psql -f schema/tables/FKEYS_table.sql
```

## Test the migration
In order to lack of DBD::Pg perl module we are not able count rows. If you install the module the command to check migration looks as follows:

```
ora2pg -c config/ora2pg.conf -t TEST
```
## Online data migration
For online data migration approach please refer [here](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md#when-the-postgresql-table-schema-already-exists)

# ./export_schema.sh
```
ora2pg -p -t TABLE -o table.sql -b ./schema/tables -c ./config/ora2pg.conf
ora2pg -p -t PACKAGE -o package.sql -b ./schema/packages -c ./config/ora2pg.conf
ora2pg -p -t VIEW -o view.sql -b ./schema/views -c ./config/ora2pg.conf
ora2pg -p -t SEQUENCE -o sequence.sql -b ./schema/sequences -c ./config/ora2pg.conf
ora2pg -p -t TRIGGER -o trigger.sql -b ./schema/triggers -c ./config/ora2pg.conf
ora2pg -p -t FUNCTION -o function.sql -b ./schema/functions -c ./config/ora2pg.conf
ora2pg -p -t PROCEDURE -o procedure.sql -b ./schema/procedures -c ./config/ora2pg.conf
ora2pg -p -t TABLESPACE -o tablespace.sql -b ./schema/tablespaces -c ./config/ora2pg.conf
ora2pg -p -t PARTITION -o partition.sql -b ./schema/partitions -c ./config/ora2pg.conf
ora2pg -p -t TYPE -o type.sql -b ./schema/types -c ./config/ora2pg.conf
ora2pg -p -t MVIEW -o mview.sql -b ./schema/mviews -c ./config/ora2pg.conf
ora2pg -p -t DBLINK -o dblink.sql -b ./schema/dblinks -c ./config/ora2pg.conf
ora2pg -p -t SYNONYM -o synonym.sql -b ./schema/synonyms -c ./config/ora2pg.conf
ora2pg -p -t DIRECTORY -o directorie.sql -b ./schema/directories -c ./config/ora2pg.conf
ora2pg -t PACKAGE -o package.sql -b ./sources/packages -c ./config/ora2pg.conf
ora2pg -t VIEW -o view.sql -b ./sources/views -c ./config/ora2pg.conf
ora2pg -t TRIGGER -o trigger.sql -b ./sources/triggers -c ./config/ora2pg.conf
ora2pg -t FUNCTION -o function.sql -b ./sources/functions -c ./config/ora2pg.conf
ora2pg -t PROCEDURE -o procedure.sql -b ./sources/procedures -c ./config/ora2pg.conf
ora2pg -t PARTITION -o partition.sql -b ./sources/partitions -c ./config/ora2pg.conf
ora2pg -t TYPE -o type.sql -b ./sources/types -c ./config/ora2pg.conf
ora2pg -t MVIEW -o mview.sql -b ./sources/mviews -c ./config/ora2pg.conf
```

# Issues
* DBD::Pg perl module isn't provided in the container. You might want to install it or use psql for import to PostgreSQL
* import_all.sh script isn't working in current version of ora2pg because of following lines: ```  pgdsn_defined=$(grep "^PG_DSN" config/ora2pg.conf | sed 's/.*dbi:Pg/dbi:Pg/')
  if [ "a$pgdsn_defined" = "a" ]; then
    if [ "a$DB_HOST" != "a" ]; then```

