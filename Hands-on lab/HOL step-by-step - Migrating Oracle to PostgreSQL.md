**Contents**
- [Migrating Oracle to PostgreSQL hands-on lab step-by-step](#migratingoracletopostgresql-hands-on-lab-step-by-step)
  - [Requirements](#requirements)
  - [Exercise 1: Setup Oracle 11g Express Edition](#exercise-1-setup-oracle-11g-express-edition)
    - [Task 1: Download ora2pg image](#task-1-download-ora2pg-image)
    - [Task 2: Download oracle image](#task-2-download-oracle-image)
  - [Exercise 2: Prepare to Migrate the Oracle database to PostgreSQL](#exercise-3-prepare-to-migrate-the-oracle-database-to-postgresql)
    - [Task 1: Create Azure Resources](#task-1-create-azure-resources)
    - [Task 2: Configure the PostgreSQL server instance](#task-2-configure-the-postgresql-server-instance)
    - [Task 3: Install pgAdmin](#task-3-install-pgadmin)
    - [Task 4: Install ora2pg](#task-4-install-ora2pg)
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

### Task 1: Download ora2pg image

[Download ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)

1. Connect to your Lab VM. 

2. Switch to root user:

       sudo -i
3. From root user run the following commands:

        dnf update -y
        dnf install podman -y

        podman pull georgmoser/ora2pg-docker

        mkdir /data

        
### Task 2: Download oracle image
[Download Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)

There are two ways of doing this 1) Own docker image, with custom password 2) Get the prebuilt image from docker hub. In this lab, we will use the option 2.

1. Installation:

        podman pull araczkowski/oracle-apex-ords
2. Run the container based on prebuild image from docker with 8080, 1521, 22 ports opened:

         podman run -d --name oracle -p 49160:22 -p 8080:8080 -p 1521:1521 araczkowski/oracle-apex-ords    
         
3. Connect to Oracle VM to check the ip (password: secret):

         `ssh root@localhost -p 49160`
         
4. Run ip a command:

        `ip a`
        
        ![Checking ip of Oracle VM.](/Media/OracleIP.png "IPA")

Connect database with the following settings:

        hostname: localhost
        port: 1521
        sid: xe
        username: system
        password: secret
4. Once you login you will get an error that the password will exprire in 7 days so you have to change the password:

        alter user system identified by secret;

5. Create a migration project:

        ora2pg --project_base /data --init_project myproject
        
6. Now, change the ORACLE_DSN project:

        vi /data/myproject/config/ora2pg.conf

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

### Task 3: Install pgAdmin

PgAdmin greatly simplifies database administration and configuration tasks by providing an intuitive GUI. Hence, we will be using it to create a new application user and test the migration.

1. You will need to navigate to <https://www.pgadmin.org/download/pgadmin-4-windows/> to obtain the latest version of pgAdmin 4, which, at the time of writing, is **v4.22**. Select the link to the installer, as shown below.

    ![The screenshot shows the correct version of the pgAdmin utility to be installed.](/Media/migrate22.PNG "pgAdmin 4 v4.22")

2. Download the **pgadmin4-4.22-x86.exe** file, **not** the one with the **.asc** extension.

    ![Screenshot shows the correct installer to be used.](/Media/migrate23.png "pgAdmin version to be installed, pgadmin4-4.22-x86.exe")

3. Once the installer launches, accept all defaults. Complete the installation steps.

4. The pgAdmin utility will open automatically in the browser. To open manually, use the Windows Search utility. Type `pgAdmin`.

   ![The screenshot shows pgAdmin in the Windows Search text box.](/Media/migrate24.png "Find pgAdmin manually in Windows Search bar")

5. PgAdmin will prompt you to set a password to govern access to database credentials. Enter a password. Confirm your choice. For now, our configuration of pgAdmin is complete.

### Task 4: Install ora2pg

**Ora2pg** is the tool we will use to migrate database objects and data. Microsoft's Data Migration Team has greatly simplified the process of obtaining this tool by providing the **installora2pg.ps1** script. You can download using the link below:

 **Download**: <https://github.com/microsoft/DataMigrationTeam/blob/master/IP%20and%20Scripts/PostgreSQL%20Migration%20and%20Assessment%20Tools/installora2pg.ps1>.

1. Copy Microsoft's script to the `C:\handsonlab\MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL\Hands-on lab\lab-files\starter-project\Postgre Scripts` location.

2. Navigate to the location mentioned above and right-click `installora2pg.ps1`. Then, select **Run with PowerShell**.

    ![Screenshot to show process to install ora2pg.](/Media/migrate25.png "Installing ora2pg")

3. Install the ora2pg utility dependencies.

   - Install Perl. It will take five minutes.
   - Install the Oracle client library and SDK. To do this, you will first need to navigate to [Oracle Downloads](https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html). Then, scroll to **Version 12.2.X**. Select the installer for the **Basic Package**.
   - Download the zip file.

    ![Screenshot to show downloading the basic package.](/Media/migrate26.PNG "Basic package download")

4. On the same Oracle link as above under the **version** section, locate the **SDK Package** installer under the **Development and Runtime - optional packages** section. Keep the zipped file in the Downloads directory.

    ![Screenshot to show the SDK Package download.](/Media/migrate27.PNG "SDK package download")

5. Navigate to the directory where the zipped instant client packages reside.

    - For the basic package, right-click it, and select **Extract All...**.
    - When prompted to choose the destination directory, navigate to the `C:\` location.
    - Select **Extract**.
    - Repeat this process for the zipped SDK.

    ![Screenshot to show the process of installing client and SDK Packages.](/Media/migrate28.PNG "Client and SDK package downloads")

6. Return to the PowerShell script.

    - Press any key to terminate the script's execution.
    - Launch the script once more.
    - If the previous steps were successful, the script should be able to locate **oci.dll** under `C:\instantclient_12_2\oci.dll`.

7. Once ora2pg installs, you will need to configure PATH variables.

    - Search for **View advanced system settings** in Windows.
    - Select the result, and the **System Properties** dialog box should open.
    - By default, the **Advanced** tab should be showing, but if not, navigate to it.
    - Then, select **Environment Variables...**.

    ![Screenshot showing process to enter environment labels.](/Media/migrate29.png "Environment Variables selected")

8. Under **System variables**, select **Path**. Select **Edit...**.

    ![Screenshot to show editing the path variables.](/Media/migrate30.png "Selecting the PATH variables")

9. The **Edit environment variable** box should be displaying.

    - Select **New**.
    - Enter **C:\instantclient_12_2**.
    - Repeat this process, but enter **%%PATH%%** instead.

    ![Screenshot showing path variable configuration.](/Meida/migrate31.png "Path variable configuration")
    
    
### Task 5: Prepare the PostgreSQL instance

In this task, we will create the vi file in the linux server, create a PostgreSQL database and download the HR DDL scripts

1. Create a new file

        vi .pg_azure

2. Add the following parameters.

        export PGDATABASE=ORA2PG
        export PGHOST=HOSTNAME.postgres.database.azure.com
        export PGUSER=user@hostname
        export PGPASSWORD=password
        export PGSSLMODE=require.

3. ![Download the HR DDL scripts](https://www.oracle.com/technetwork/developer-tools/datamodeler/hr-30-ddl-246035.zip)

4. Create Database

        psql postgres
        createdb ora2pg
        psql
        psql -f schema/tables/table.sql
        psql -f schema/sequences/sequence.sql
        psql -f schema/views/view.sql 
        psql -f schema/procedures/procedure.sql
        psql -f schema/triggers/trigger.sql
        
 >**Note**: If you get an error (SELECT COUNT(*) FROM v$archived_log;) that there are no archive log run this command in oracle (ALTER SYSTEM ARCHIVE LOG CURRENT)
 
 ### Task 6: Create an ora2pg project structure
 
 **Ora2pg** allows database objects to be exported in multiple files so that is simple to organize and review changes. In this task, you will create the project structure that will make it easy to do this.  
 
### Task 7: Create a migration report

The migration report tells us the "man-hours" required to fully migrate to our application and components. The report will provide the user with a relative complexity value. In this task, we will retrieve the migration report for our migration.

## Exercise 3: Migrate the Database

In this exercise, we will begin the migration of the database and the application. This includes migrating database objects, the data, application code, and finally, deploying to Azure App Service.

### Task 1: Migrate the basic database table schema using ora2pg

In this task, we will migrate the database table schema, using ora2pg and psql, which is a command-line utility that makes it easy to run SQL files against the database.

1. Exercise 3 covered planning and assessment steps.  To start the database migration, DDL statements must be created for all valid Oracle objects.

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

