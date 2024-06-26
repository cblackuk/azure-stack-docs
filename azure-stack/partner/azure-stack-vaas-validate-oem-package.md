---
title: Validate OEM packages
titleSuffix: Azure Stack Hub
description: Learn how to validate OEM packages with Azure Stack Hub validation as a service.
author: sethmanheim
ms.topic: tutorial
ms.date: 12/16/2020
ms.author: sethm
ms.reviewer: johnhas
ms.lastreviewed: 11/11/2019
ROBOTS: NOINDEX

# Intent: As a < type of user >, I want < what? > so that < why? >
# Keyword: Azure Stack keyword

---

# Validate OEM packages

[!INCLUDE [Azure_Stack_Partner](./includes/azure-stack-partner-appliesto.md)]

You can test a new original equipment manufacturer (OEM) package when there has been a change to the firmware or drivers for a completed solution validation. When your package has passed the test, it's signed by Microsoft. Your test must contain the updated OEM extension package with the drivers and firmware that have passed Windows Server logo and PCS tests.

[!INCLUDE [azure-stack-vaas-workflow-validation-completion](includes/azure-stack-vaas-workflow-validation-completion.md)]

> [!IMPORTANT]
> Before uploading or submitting packages, review [Create an OEM package](azure-stack-vaas-create-oem-package.md) for the expected package format and contents.

## Managing packages for validation

When using the **Package Validation** workflow to validate a package, you need to provide a URL to an **Azure Storage blob**. This blob is the test signed OEM package installed as part of the update process. Create the blob using the Azure Storage account you created during setup (see [Set up your validation as a service [VaaS] resources](azure-stack-vaas-set-up-resources.md)).

### Prerequisite: Provision a storage container

Create a container in your storage account for package blobs. This container can be used for all your Package Validation runs.

1. In the [Azure portal](https://portal.azure.com), go to the storage account created in [Set up your VaaS resources](azure-stack-vaas-set-up-resources.md).

2. On the left blade under **Blob Service**, select **Containers**.

3. Select **+ Container** from the menu bar.
    1. Provide a name for the container. For example, `vaaspackages`.
    1. Select the desired access level for unauthenticated clients such as VaaS. For details on how to grant VaaS access to packages in each scenario, see [Handling container access level](#handling-container-access-level).

### Upload package to storage account

1. Prepare the package you want to validate. This is a `.zip` file whose contents must match the structure described in [Create an OEM package](azure-stack-vaas-create-oem-package.md).

    > [!NOTE]
    > Please ensure that the `.zip` contents are placed at the root of the `.zip` file. There should be no sub-folders in the package.

1. In the [Azure portal](https://portal.azure.com), select the package container and upload the package by selecting **Upload** in the menu bar.

1. Select the package `.zip` file to upload. Keep defaults for **Blob type** (that is, **Block Blob**) and **Block size**.

### Generate package blob URL for VaaS

When creating a **Package Validation** workflow in the VaaS portal, you need to provide a URL to the Azure Storage blob containing your package. Some *interactive* tests, including **Monthly Azure Stack Hub Update Verification** and **OEM Extension Package Verification**, also require a URL to package blobs.

#### Handling container access level

The minimum access level required by VaaS depends on whether you're creating a Package Validation workflow or scheduling an *interactive* test.

In the case of **Private** and **Blob** access levels, you must temporarily grant access to the package blob by giving VaaS a [shared access signature](/azure/storage/common/storage-dotnet-shared-access-signature-part-1) (SAS). The **Container** access level doesn't require you to generate SAS URLs but allows unauthenticated access to the container and its blobs.

|Access level | Workflow requirement | Test requirement |
|---|---------|---------|
|Private | Generate a SAS URL per package blob ([Option 1](#option-1-generate-a-blob-sas-url)). | Generate a SAS URL at the account level and manually add the package blob name ([Option 2](#option-2-construct-a-container-sas-url)). |
|Blob | Provide the blob URL property ([Option 3](#option-3-grant-public-read-access)). | Generate a SAS URL at the account level and manually add the package blob name ([Option 2](#option-2-construct-a-container-sas-url)). |
|Container | Provide the blob URL property ([Option 3](#option-3-grant-public-read-access)). | Provide the blob URL property ([Option 3](#option-3-grant-public-read-access)).

The options for granting access to your packages are ordered from least access to greatest access.

#### Option 1: Generate a blob SAS URL

Use this option if the access level of your storage container is set to **Private**, where the container doesn't enable public read access to the container or its blobs.

> [!NOTE]
> This method won't work for *interactive* tests. See [Option 2: Construct a container SAS URL](#option-2-construct-a-container-sas-url).

1. In the [Azure portal](https://portal.azure.com/), go to your storage account and navigate to the `.zip` containing your package.

2. Select **Generate SAS** from the context menu.

3. Select **Read** from **Permissions**.

4. Set **Start time** to the current time, and **End time** to at least 48 hours from **Start time**. If you'll be creating other workflows with the same package, consider increasing **End time** for the length of your testing.

5. Select **Generate blob SAS token and URL**.

Use the **Blob SAS URL** when providing package blob URLs to the portal.

#### Option 2: Construct a container SAS URL

Use this option if the access level of your storage container is set to **Private** and you need to supply a package blob URL to an *interactive* test. This URL can also be used at the workflow level.

1. [!INCLUDE [azure-stack-vaas-sas-step_navigate](includes/azure-stack-vaas-sas-step_navigate.md)]

1. Select **Blob** from **Allowed Services options**. Deselect any remaining options.

1. Select **Container** and **Object** from **Allowed resource types**.

1. Select **Read** and **List** from **Allowed permissions**. Deselect any remaining options.

1. Select **Start time** as current time and **End time** to at least 14 days from **Start time**. If you'll be running other tests with the same package, consider increasing **End time** for the length of your testing. Any tests scheduled through VaaS after **End time** will fail and a new SAS will need to be generated.

1. [!INCLUDE [azure-stack-vaas-sas-step_generate](includes/azure-stack-vaas-sas-step_generate.md)]
    The format should look like this:
    `https://storageaccountname.blob.core.windows.net/?sv=2016-05-31&ss=b&srt=co&sp=rl&se=2017-05-11T21:41:05Z&st=2017-05-11T13:41:05Z&spr=https`

1. Modify the generated SAS URL to include the package container, `{containername}`, and the name of your package blob, `{mypackage.zip}`. Like this:
    `https://storageaccountname.blob.core.windows.net/{containername}/{mypackage.zip}?sv=2016-05-31&ss=b&srt=co&sp=rl&se=2017-05-11T21:41:05Z&st=2017-05-11T13:41:05Z&spr=https`

    Use this value when providing package blob URLs to the portal.

#### Option 3: Grant public read access

Use this option if it's acceptable to allow unauthenticated clients access to individual blobs or, in the case of *interactive* tests, the container.

> [!CAUTION]
> This option opens up your blob(s) for anonymous read-only access.

1. Set the access level of the package container to **Blob** or **Container**. For more information, see [Grant anonymous users permissions to containers and blobs](/azure/storage/storage-manage-access-to-resources#grant-anonymous-users-permissions-to-containers-and-blobs).

    > [!NOTE]
    > If you're providing a package URL to an *interactive* test, you must grant **full public read access** to the container to proceed with testing.

1. In the package container, select the package blob to open the properties pane.

1. Copy the **URL**. Use this value when providing package blob URLs to the portal.

## Create a Package Validation workflow

1. Sign in to the [VaaS portal](https://azurestackvalidation.com).

2. [!INCLUDE [azure-stack-vaas-workflow-step_select-solution](includes/azure-stack-vaas-workflow-step_select-solution.md)]

3. Select **Start** on the **Package Validation** tile.

    ![Package validations workflow tile](media/tile_validation-package.png)

4. [!INCLUDE [azure-stack-vaas-workflow-step_naming](includes/azure-stack-vaas-workflow-step_naming.md)]

5. Enter the Azure Storage blob URL to the test signed OEM package requiring a signature from Microsoft. For instructions, see [Generate package blob URL for VaaS](#generate-package-blob-url-for-vaas).

6. Copy the Azure Stack Hub update package folder to a local directory on the DVM. Enter the path to the **folder that contains the package zip file and metadata file** for 'AzureStack update package folder path'.

7. Copy the OEM package folder created above to a local directory on the DVM. Enter the path to the **folder that contains the package zip file and metadata file** for 'OEM update package folder path'.

    > [!NOTE]
    > Copy the Azure Stack Hub update and OEM update to **2 separate** directories.

8. `RequireDigitalSignature` - provide the value **true** if you need the package to be Microsoft signed (Running OEM Validation workflow). If you're validating a Microsoft signed package on the latest Azure Stack Hub update, make this value false (running monthly Azure Stack Hub update verification).

9. [!INCLUDE [azure-stack-vaas-workflow-step_test-params](includes/azure-stack-vaas-workflow-step_test-params.md)]

    > [!NOTE]
    > Environment parameters can't be modified after creating a workflow.

10. [!INCLUDE [azure-stack-vaas-workflow-step_tags](includes/azure-stack-vaas-workflow-step_tags.md)]

11. [!INCLUDE [azure-stack-vaas-workflow-step_submit](includes/azure-stack-vaas-workflow-step_submit.md)]
    You'll be redirected to the tests summary page.

## Required tests

The following tests are required to be run for OEM package validation:

- OEM Validation Workflow

## Run Package Validation tests

1. In the **Package Validation tests summary** page, you'll run a subset of the listed tests appropriate to your scenario.

    In the validation workflows, **scheduling** a test uses the workflow-level common parameters that you specified during workflow creation (see [Workflow common parameters for Azure Stack Hub validation as a service](azure-stack-vaas-parameters.md)). If any of test parameter values become invalid, you must resupply them as instructed in [Modify workflow parameters](azure-stack-vaas-monitor-test.md#change-workflow-parameters).

    > [!NOTE]
    > Scheduling a validation test over an existing instance will create a new instance in place of the old instance in the portal. Logs for the old instance will be retained but aren't accessible from the portal.<br><br>
    > Once a test has completed successfully, the **Schedule** action becomes disabled.

2. Select the agent that will run the test. For information about adding local test execution agents, see [Deploy the local agent](azure-stack-vaas-local-agent.md).

3. To schedule the test run, select **Schedule** from the context menu to open a prompt for scheduling the test instance.

4. Review the test parameters and then select **Submit** to schedule the test.

5. Review the results for the **required** tests.

To submit a package signing request, contact Microsoft Support with the solution name and package validation name associated with this run.

## Next steps

- [Monitor and manage tests in the VaaS portal](azure-stack-vaas-monitor-test.md)
