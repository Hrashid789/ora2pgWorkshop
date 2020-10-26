**Contents**

- [Migrating Oracle to PostgreSQL before the hands-on lab setup guide](#migratingoracletoazuresql-andpostgresql-before-the-hands-on-lab-setup-guide)
  - [Requirements](#requirements)
  - [Before the hands-on lab](#before-the-hands-on-lab)
    - [Task 1: Provision a resource group](#task-1-provision-a-resource-group)
    - [Task 2: Create lab virtual machine](#task-2-create-lab-virtual-machine)
    - [Task 4: Register the Microsoft DataMigration resource provider](#task-3-register-the-microsoft-datamigration-resource-provider)
    - [Task 5  (Migrate to PostgreSQL): Create Azure Database Migration Service for an Oracle to PostgreSQL Migration](#task-4-migrate-to-postgresql-create-azure-database-migration-service-for-an-oracle-to-postgresql-migration)

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

2. Select "Compute" under Azure Marketplace and then select "Virtual Machines"

![create a virtual machine](/Media/migrate6.jpg)

3. On the Create a virtual machine Basics tab, set the following configuration:

- Project Details:

        - **Subscription**: Select the subscription you are using for this hands-on lab.
        - **Resource Group**: Select the hands-on-lab-SUFFIX resource group from the list of existing resource groups.

    - Instance Details:

        - **Virtual machine name**: Enter LabVM.
        - **Region**: Select the region you are using for resources in this hands-on lab.
        - **Availability options**: Select no infrastructure redundancy required.
        - **Image**: Centos-based 8.2 - Gen1
        - **Size**: Accept the default size, Standard D2 v2.
        
 - Administrator Account:
        - **Authentication type**: Password
        - **Username**: demouser
        - **Password**: Password.1!!

    - Inbound Port Rules:

        - **Public inbound ports**: Choose Allow selected ports.
        - **Select inbound ports**: Select SSH (22) in the list.
        
        ![Screenshot of the Basics tab, with fields set to the previously mentioned settings.](/Media/migrate7.jpg "Create a virtual machine Basics tab")

4. Select **Review + create**.

5. On the **Review + create** tab, ensure the Validation passed message is displayed, and then select **Create** to provision the virtual machine.

    ![The Review + create tab is displayed, with a Validation passed message.](/Media/migrate8.jpg "Create a virtual machine Review + create tab")

6. It may take 10+ minutes for the virtual machine to complete provisioning. You can move on to the next tasks while waiting for the lab VM to provision.


### Task 3: Install PostgreSQL binaries

In this task, you will install postgres binaries to be able to connect to PostgreSQL PaaS service and deploy migrated code and load data.

1. Connect to your Azure VM

2. Switch to root and install postgres binaries:

        dnf module enable postgresql:12
        dnf install postgresql


### Task 4: Register the Microsoft DataMigration resource provider

In this task, you will register the `Microsoft.DataMigration` resource provider with your subscription in Azure.

1. In the [Azure portal](https://portal.azure.com/), navigate to the Home page and then select **Subscriptions** from the Navigate list found midway down the page.

    ![Subscriptions is highlighted in the Navigate menu.](/Media/migrate9.jpg "Navigate menu")

2. Select the subscription you are using for this hands-on lab from the list, select **Resource providers**, enter "migration" into the filter box, and then select **Register** next to **Microsoft.DataMigration**.

    ![The Subscription blade is displayed, with Resource providers selected and highlighted under Settings. On the Resource providers blade, migration is entered into the filter box, and Register is highlighted next to Microsoft.DataMigration.](/Media/migrate10.png "Resource provider registration")

### Task 5: (Migrate to PostgreSQL): Create Azure Database Migration Service for an Oracle to PostgreSQL Migration

In this task, you will provision an instance of the Azure Database Migration Service (DMS) for use with an *online* Oracle to PostgreSQL migration. This requires that we implement the **Premium** tier.

1. In the [Azure portal](https://portal.azure.com/), select the **Show portal menu** icon and then select **+Create a resource** from the menu.

    ![The Show portal menu icon is highlighted and the portal menu is displayed. Create a resource is highlighted in the portal menu.](/Media/migrate11.png "Create a resource")

2. Enter "database migration" into the Search the Marketplace box, select **Azure Database Migration Service** from the results, and select **Create**.

    !["Database migration" is entered into the Search the Marketplace box. Azure Database Migration Service is selected in the results.](/Media/migrate12.png "Create Azure Database Migration Service")

3. On the Create Migration Service blade, enter the following:

    - **Subscription**: Select the subscription you are using for this hands-on lab.
    - **Resource Group**: Select the hands-on-lab-SUFFIX resource group from the list of existing resource groups.
    - **Migration service name**: Enter **wwi-dms-SUFFIX**.
    - **Location**: Select the location you are using for resources in this hands-on lab.
    - **Pricing tier**: Select Premium: 4 vCores (you will need to select *Configure tier* to view this option).

    > **Note**: If you see the message `Your subscription doesn't have proper access to Microsoft.DataMigration`, refresh the browser window before proceeding. If the message persists, verify you successfully registered the resource provider, and then you can safely ignore this message.

   ![The Create Migration Service blade is displayed, with the values specified above entered into the appropriate fields.](/Media/migrate13.png "Create Migration Service")

4. Select **Next: Networking**.

5. On the Network tab, select the **hands-on-lab-SUFFIX-vnet/default** virtual network. This will place the DMS instance into the same VNet as your SQL Server and Lab VMs.

    ![The hands-on-lab-SUFFIX-vnet/default is selected in the list of available virtual networks.](/Media/migrate14.png "Create migration service")

6. Select **Review + create**.
  
7. Select **Create**.

>**Note**: It can take 15 minutes to deploy the Azure Data Migration Service.

You should follow all steps provided *before* performing the Hands-on lab.
