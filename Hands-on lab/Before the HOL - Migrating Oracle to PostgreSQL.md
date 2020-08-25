**Contents**

- [Migrating Oracle to PostgreSQL before the hands-on lab setup guide](#migratingoracletoazuresql-andpostgresql-before-the-hands-on-lab-setup-guide)
  - [Requirements](#requirements)
  - [Before the hands-on lab](#before-the-hands-on-lab)
    - [Task 1: Provision a resource group](#task-1-provision-a-resource-group)
    - [Task 2: Create lab virtual machine](#task-2-create-lab-virtual-machine)
    - [Task 3: Connect to the Lab VM](#task-3-connect-to-the-lab-vm)
    - [Task 4: Download ora2pg image](#task-4-download-ora2pg-image)
    - [Task 5: Download Oracle image](#task-5-download-oracle-image)
    - [Task 6: Register the Microsoft DataMigration resource provider](#task-6-register-the-microsoft-datamigration-resource-provider)
    - [Task 7  (Migrate to PostgreSQL): Create Azure Database Migration Service for an Oracle to PostgreSQL Migration](#task-7-migrate-to-postgresql-create-azure-database-migration-service-for-an-oracle-to-postgresql-migration)

# Migrating Oracle to Azure SQL and PostgreSQL before the hands-on lab setup guide

## Requirements

- Microsoft Azure subscription must be pay-as-you-go or MSDN.
  - Trial subscriptions will not work.
## Before the hands-on lab

In the Before the hands-on lab exercise, you will set up your environment for use in the rest of the hands-on lab. You should follow all the steps provided in the Before the hands-on lab section to prepare your environment **before attending** the hands-on lab. Failure to do so will significantly impact your ability to complete the lab within the time allowed.

> **Important**: Most Azure resources require unique names. Throughout this lab you will see the word “SUFFIX” as part of resource names. You should replace this with your Microsoft alias, initials, or another value to ensure the resource is uniquely named.

### Task 1: Provision a resource group

In this task, you will create an Azure resource group for the resources used throughout this lab.

1. In the [Azure portal](https://portal.azure.com), select **Resource groups** from the Azure services list.

    ![Resource groups is highlighted in the Azure services list.](/Media/migrate2.jpg "Azure services")

2. On the Resource groups blade, select **+Add**.

    ![+Add is highlighted in the toolbar on Resource groups blade.t](/Media/migrate3.jpg "Resource groups")

3. Enter the following in the Create an empty resource group blade:

    - **Subscription**: Select the subscription you are using for this hands-on lab.
    - **Resource group**: Enter hands-on-lab-SUFFIX.
    - **Region**: Select the region you would like to use for resources in this hands-on lab. Remember this location so you can use it for the other resources you'll provision throughout this lab.

    ![Add Resource group Resource groups is highlighted in the navigation pane of the Azure portal, +Add is highlighted in the Resource groups blade, and "hands-on-labs" is entered into the Resource group name box on the Create an empty resource group blade.](/Media/migrate4.jpg "Create resource group")

4. Select **Review + Create**.

5. On the Review + Create tab, select **Create** to provision the resource group.

### Task 2: Create lab virtual machine

In this task, you will provision a virtual machine (VM) in Azure. The VM image used will have Visual Studio Community 2019 installed.

1. In the [Azure portal](https://portal.azure.com/), select the **Show portal menu** icon and then select **+Create a resource** from the menu.

  ![The Show portal menu icon is highlighted and the portal menu is displayed. Create a resource is highlighted in the portal menu.](/Media/migrate5.jpg "Create a resource")

### Task 3:

### Task 4: Download ora2pg image
[Download ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)

1. Setting up ora2pg:

       sudo -i
2. From root user run the following commands:

        yum update -y
        yum install docker -y

        systemctl start docker
        systemctl status docker

         docker pull georgmoser/ora2pg-docker

        mkdir /data
      > **Note**: You may receive an error "Installation of docker fails on CentOS 8 with Error – package containerd.io-1.2.10-3.2.el7.x86_64 is excluded" to overcome that you will need to run the below command:
              
               yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm

3. Get inside container:

        docker run -it --privileged -v /data:/data georgmoser/ora2pg-docker /bin/bash

        ora2pg --version

        apt-get update -y

        apt-get install vim

### Task 5:Download Oracle image
[Download Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)

There are two ways of doing this 1) Own docker image, with custom password 2) Get the prebuilt image from docker hub. In this lab, we will use the option 2.

1. Installation:

        docker pull araczkowski/oracle-apex-ords
2. Run the container based on prebuild image from docker with 8080, 1521, 22 ports opened:

         docker run -d --name <own-container-name> -p 49160:22 -p 8080:8080 -p 1521:1521 araczkowski/oracle-apex-ords    
         
3. Connect database with the following settings:

        hostname: localhost
        port: 1521
        sid: xe
        username: system
        password: secret
4. Once you login you will get an error that the password will exprire in 7 days so you have to change the password:

        alter user system identified by secret;
### Task 6:

In this task, you will register the `Microsoft.DataMigration` resource provider with your subscription in Azure.

1. In the [Azure portal](https://portal.azure.com/), navigate to the Home page and then select **Subscriptions** from the Navigate list found midway down the page.

    ![Subscriptions is highlighted in the Navigate menu.](media/azure-navigate-subscriptions.png "Navigate menu")

2. Select the subscription you are using for this hands-on lab from the list, select **Resource providers**, enter "migration" into the filter box, and then select **Register** next to **Microsoft.DataMigration**.

    ![The Subscription blade is displayed, with Resource providers selected and highlighted under Settings. On the Resource providers blade, migration is entered into the filter box, and Register is highlighted next to Microsoft.DataMigration.](media/azure-portal-subscriptions-resource-providers-register-microsoft-datamigration.png "Resource provider registration")

### Task 7:
In this task, you will provision an instance of the Azure Database Migration Service (DMS) for use with an *online* Oracle to PostgreSQL migration. This requires that we implement the **Premium** tier.

1. In the [Azure portal](https://portal.azure.com/), select the **Show portal menu** icon and then select **+Create a resource** from the menu.

    ![The Show portal menu icon is highlighted and the portal menu is displayed. Create a resource is highlighted in the portal menu.](media/create-a-resource.png "Create a resource")

2. Enter "database migration" into the Search the Marketplace box, select **Azure Database Migration Service** from the results, and select **Create**.

    !["Database migration" is entered into the Search the Marketplace box. Azure Database Migration Service is selected in the results.](media/create-resource-azure-database-migration-service.png "Create Azure Database Migration Service")

3. On the Create Migration Service blade, enter the following:

    - **Subscription**: Select the subscription you are using for this hands-on lab.
    - **Resource Group**: Select the hands-on-lab-SUFFIX resource group from the list of existing resource groups.
    - **Migration service name**: Enter **wwi-dms-SUFFIX**.
    - **Location**: Select the location you are using for resources in this hands-on lab.
    - **Pricing tier**: Select Premium: 4 vCores (you will need to select *Configure tier* to view this option).

    > **Note**: If you see the message `Your subscription doesn't have proper access to Microsoft.DataMigration`, refresh the browser window before proceeding. If the message persists, verify you successfully registered the resource provider, and then you can safely ignore this message.

   ![The Create Migration Service blade is displayed, with the values specified above entered into the appropriate fields.](media/create-premium-migration-service.png "Create Migration Service")

4. Select **Next: Networking**.

5. On the Network tab, select the **hands-on-lab-SUFFIX-vnet/default** virtual network. This will place the DMS instance into the same VNet as your SQL Server and Lab VMs.

    ![The hands-on-lab-SUFFIX-vnet/default is selected in the list of available virtual networks.](media/create-migration-service-networking-tab.png "Create migration service")

6. Select **Review + create**.
  
7. Select **Create**.

>**Note**: It can take 15 minutes to deploy the Azure Data Migration Service.

You should follow all steps provided *before* performing the Hands-on lab.
