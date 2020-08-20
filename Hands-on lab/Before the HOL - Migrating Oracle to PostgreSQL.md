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

    ![Resource groups is highlighted in the Azure services list.](media/azure-services-resource-groups.png "Azure services")

2. On the Resource groups blade, select **+Add**.

    ![+Add is highlighted in the toolbar on Resource groups blade.t](media/resource-groups-add.png "Resource groups")

3. Enter the following in the Create an empty resource group blade:

    - **Subscription**: Select the subscription you are using for this hands-on lab.
    - **Resource group**: Enter hands-on-lab-SUFFIX.
    - **Region**: Select the region you would like to use for resources in this hands-on lab. Remember this location so you can use it for the other resources you'll provision throughout this lab.

    ![Add Resource group Resource groups is highlighted in the navigation pane of the Azure portal, +Add is highlighted in the Resource groups blade, and "hands-on-labs" is entered into the Resource group name box on the Create an empty resource group blade.](./media/create-resource-group.png "Create resource group")

4. Select **Review + Create**.

5. On the Review + Create tab, select **Create** to provision the resource group.

### Task 2: Create lab virtual machine

In this task, you will provision a virtual machine (VM) in Azure. The VM image used will have Visual Studio Community 2019 installed.

1. In the [Azure portal](https://portal.azure.com/), select the **Show portal menu** icon and then select **+Create a resource** from the menu.

### Task 3:

### Task 4:
[Download ora2pg image](https://hub.docker.com/r/georgmoser/ora2pg-docker)
### Task 5:
[Download Oracle image](https://hub.docker.com/r/araczkowski/oracle-apex-ords)
### Task 6:

### Task 7:
