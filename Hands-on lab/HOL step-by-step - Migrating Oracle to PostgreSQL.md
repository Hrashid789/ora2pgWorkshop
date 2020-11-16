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
![Setup ora2pg](/Media/ex01_Task01_SetupOra2pg.gif "Setup Ora2pg") 

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
        
You will see that command prompt has changed. Now it looks similarly to this one:

        root@d36721adb825:/#
        
It consists of user name (root), at sign (@) and alphanumeric string which identifies the container that you are running (container id).  
        
5. **Ora2pg** allows database objects to be exported in multiple files so that is simple to organize and review changes. This command will create the project structure that will make it easy to do this: 

        ora2pg --project_base /data --init_project myproject
        
6. Log out from the container (Ctrl+d)
        

        
### Task 2: Setup Oracle
![Setup Oracle](/Media/ex01_Task02_SetupOracle.gif "Setup Oracle") 

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

Connection details for your Oracle instance will look as follows:

        hostname: <Output of ip a command>
        port: 1521
        sid: xe
        username: system
        password: secret
        
5. Log out of the Oracle VM and change ORACLE_DSN in ora2pg.conf file accordingly. Then find the line that begins with SCHEMA and change the string "CHANGE_THIS_SCHEMA_NAME" to HR:

        vi /data/myproject/config/ora2pg.conf
        
![Changing Oracle DSN.](/Media/OracleConnection.png "Changing Oracle DSN")
![Changing schema name to HR](/Media/schemaname.PNG "Changing Schema name")

Save changed configuration file.

6. Check if you are able to connect to Oracle database from ora2pg tool:

        podman run -it --privileged -v /data:/data georgmoser/ora2pg-docker ora2pg -c /data/myproject/config/ora2pg.conf -t SHOW_VERSION

![Test Connection.](/Media/OracleConnectTest.png "Test connection")       

If ora2pg displays the line with Oracle version: "Oracle Database 11g Express Edition Release 11.2.0.2.0" despite the warnings you might
see above it means that ora2pg is well configured and you are ready to start the migration.
        
7. Optionally you might want to change the password for system user since the current one will expire in 7 days. That will
also stop the warning about password expiration from being displayed at each call:

        alter user system identified by secret;
        


## Exercise 2: Export Oracle schema and data to VM
![Export Oracle schema and data to VM](/Media/ex02_ExportSchemaAndData.gif "Export Oracle schema and data to VM") 
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

## Exercise 3: Prepare to Migrate the Oracle database to PostgreSQL

In this exercise, you will create a new PaaS service - Azure Database for PostgreSQL.

### Task 1: Create Azure Resources
![Create Postgres PaaS](/Media/ex02-Task01Create%20Azure%20Resources.gif "Create Postgres PaaS") 

1. Just as you configured resources in **Before the HOL**, you will need to navigate to the **New** page accessed by selecting **+ Create a resource**. Then, navigate to **Databases** under the **Azure Marketplace** section. Select **Azure Database for PostgreSQL**.

    ![Navigating Azure Marketplace to Azure Database for PostgreSQL, which has been highlighted.](/Media/migrate15.png "Azure Database for PostgreSQL")

2. There are many deployment options including: **Single Server** and **Hyperscale (Citus)**. Single Server is best suited for traditional transactional workloads whereas Hyperscale is best suited for ultra-high-performance, multi-tenant applications. For our simple application, we will be utilizing a single server for our database.

    ![Screenshot of choosing the correct single server option.](/Media/migrate16.PNG "Single server")

3. Create a new Azure Database for PostgreSQL resource. Use the following configuration values:

   - **Resource group**: (same as Lab VM)
   - **Server name**: Enter a unique server name (we use **demopg**)
   - **Version**: 11
   - **Administrator username**: pgdba
   - **Password**: (secure password)

    Select **Review + create** button once you are ready.

    ![Configuring the instance details.](/Media/pgpaascreate.PNG "Project Details window with pertinent details")

### Task 5: Prepare the file with libpq Environment Variables

In this task, we will create a file in our linux VM containing [environment variables] (https://www.postgresql.org/docs/current/libpq-envars.html) that will be used to select 
default connection parameter values to PostgreSQL PaaS instance. These are useful to be able to 
connect to postgres in a fast and convenient way without hard-coding connection string.

1. Go to the "Connection Strings" tab on the left hand side of the Azure Portal and find psql connection string:

![Connection string](/Media/connectionString.png "Connection String")

2. From a terminal connect to Azure VM

3. Switch to root user:
        
        sudo -i
        
![Exporting libpq variables](/Media/ex03_libpq.gif "Exporting libpq variables")

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
 
 
## Exercise 4: Migrate the Database

All the data and schema are already on the LabVM. Now we need to deploy them to the previously created Postgres instance.

### Task 1: (Schema & Offline Data) Migration using ora2pg

In this task, we will migrate the database schema and data, previously loaded to LabVM, using a import_all.sh script provided by ora2pg.

![Deploy to Postgres](/Media/ex04_import_to_postgres.gif "Automatic migration using import_all.sh script")

1. Make sure that libpq variables, necessary to connect to postgres, are exported in current session:

        source .pg_azure

2. Navigate to the main project directory:

        cd /data/myproject

3. Run import_all.sh script with database name and username as the parameters:
        
        ./import_all.sh -d ora2pg -o pgdba@demopg
        
4. Answer NO for the first two questions, the user is created and the database was newly created so there is no need to drop it.

        Would you like to create the owner of the database pgdba@demopg?
        Would you like to create the database ora2pg?
        
5. Answer YES to all import questions till you see a question about importing synonyms. The test database that we use in this workshop contains
synonyms to the objects that we haven't migrated so we should skip import of the synonyms:

        Would you like to import TABLE from ./schema/tables/table.sql? [y/N/q]
        Would you like to import VIEW from ./schema/views/view.sql? [y/N/q]
        Would you like to import SEQUENCE from ./schema/sequences/sequence.sql? [y/N/q]
        Would you like to import TRIGGER from ./schema/triggers/trigger.sql? [y/N/q]
        Would you like to import PROCEDURE from ./schema/procedures/procedure.sql? [y/N/q]
        
6. (NO) Skip synonyms:

        Would you like to import SYNONYM from ./schema/synonyms/synonym.sql? [y/N/q]
        
7. (NO) Skip also importing all the constaints before loading the data:

        Would you like to process indexes and constraints before loading data? [y/N/q]
        
After loading the data the script will ask you again about the constraints.

8. (YES) Load the data:

        Would you like to import data from ./data/data.sql? [y/N/q]
        
9. (YES) Now we are safe to load the indexes and constraints:

        Would you like to import indexes from ./schema/tables/INDEXES_table.sql? [y/N/q]
        Would you like to import constraints from ./schema/tables/CONSTRAINTS_table.sql? [y/N/q]
        Would you like to import foreign keys from ./schema/tables/FKEYS_table.sql? [y/N/q]
        
10. Connect to postgres and run VACUUM ANALYZE. Running this command is crucial after every data load and migration:

        psql
        VACUUM ANALYZE;
        
11. Your database was successfully migrated. Congratulations!
        
        
        


 
 
 
 
 
 

# Helpful Links:
[Full Windows-based workshop](https://github.com/microsoft/MCW-Migrating-Oracle-to-Azure-SQL-and-PostgreSQL/blob/master/Hands-on%20lab/HOL%20step-by-step%20-%20Migrating%20Oracle%20to%20PostgreSQL.md#task-4-install-ora2pg)

[Tutorial](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/dms/tutorial-oracle-azure-postgresql-online.md)

[ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)

[Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)

[HR DDLScript - Import the HR sample](https://www.oracle.com/database/technologies/appdev/datamodeler-samples.html)

[Examples](https://www.darold.net/confs/ora2pg_the_hard_way.pdf)