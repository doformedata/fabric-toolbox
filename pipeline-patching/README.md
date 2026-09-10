# Set Pipeline Last Modified By Identity

A Microsoft Fabric notebook for updating the **Last Modified By** identity of Data Pipelines in a workspace.

The notebook uses the Microsoft Fabric REST API to patch pipeline metadata. The identity running the notebook becomes the `LastModifiedBy` identity of the pipelines that are updated.

This can be useful when you want pipelines to run under a specific identity, such as a **service principal** or **workspace identity**, instead of depending on an individual user's identity.

> **Note:** `LastModifiedBy` is separate from the Fabric item's **Owner**. This notebook changes the identity that last modified the pipeline. It does not transfer item ownership.

## How it works

The notebook can be used in two ways:

### Run the notebook directly

You can run the notebook as a standalone notebook.

When run interactively, the notebook uses your user identity to call the Fabric REST API. Your user will therefore become the `LastModifiedBy` identity of the pipelines that are patched.

### Run the notebook from a pipeline

My preferred approach is to create a small helper pipeline that runs this notebook.

This lets authentication be handled through a **Fabric connection** instead of storing service principal credentials, retrieving secrets from Azure Key Vault, or implementing authentication logic inside the notebook.

It also makes it possible to use a **Fabric workspace identity**.

The flow is:

```text
Helper Pipeline
      │
      │ Notebook activity
      │ Connection: desired identity
      ▼
Patch Pipeline Notebook
      │
      │ Fabric REST API
      ▼
Target Pipelines
```

The identity selected for the Notebook activity becomes the identity executing the notebook and calling the Fabric REST API. When the notebook patches the target pipelines, that identity becomes their `LastModifiedBy` identity.

## Recommended setup

### 1. Add the notebook to your workspace

Import the notebook into the Fabric workspace containing the pipelines you want to update.

The notebook automatically works against the current workspace.

### 2. Create a helper pipeline

Create a Data Pipeline that contains a **Notebook activity** pointing to this notebook.

I recommend keeping this pipeline specifically for setting the Last Modified By identity.

### 3. Create the `pipeline_names` parameter

Add a pipeline parameter named:

```text
pipeline_names
```

Pass this parameter to the Notebook activity using a dynamic value.

The notebook can then receive the pipelines you want to patch when the helper pipeline is triggered.

### 4. Select the connection

In the Notebook activity, select or create a connection using the identity you want to become the `LastModifiedBy` identity.

For example:

* Organizational account
* Service principal
* Workspace identity

The connection handles authentication for the notebook run, so no credentials need to be stored in the notebook itself.

### 5. Run the helper pipeline

When triggering the pipeline, you can:

* Pass one or more pipeline names to patch only those pipelines.
* Leave `pipeline_names` empty to patch all pipelines in the workspace except pipelines defined in the notebook's ignore list.

The notebook contains an ignore list that can be used to protect pipelines you never want to modify.

## Using multiple identities

If different pipelines should use different Last Modified By identities, I recommend creating a separate helper pipeline for each identity.

For example:

```text
patch-as-workspace-identity
    └── Notebook connection: Workspace Identity

patch-as-service-principal
    └── Notebook connection: Service Principal A

patch-as-service-principal-b
    └── Notebook connection: Service Principal B
```

Each helper pipeline can use the same notebook.

You only need to select a different Notebook activity connection and pass the pipelines that should use that identity.

This keeps identity configuration in Fabric connections instead of spreading authentication logic and credentials across notebooks.

## Permissions

The identity used to run the notebook needs sufficient permissions to perform the required operations.

At minimum:

* The identity needs **Contributor** access or higher in the workspace to modify Data Pipeline items.
* The identity must have permission to use any Fabric connections required by the pipelines when they run.
* The identity behind those connections must have the required permissions on the underlying data sources.

When using a **workspace identity**, make sure the workspace identity has been created for the workspace and has the required Fabric and data-source permissions.

Depending on your tenant configuration, service principal and workspace identity usage with Fabric APIs may also require the relevant tenant settings to be enabled.

## Important

Changing `LastModifiedBy` can change the security context used during pipeline execution.

Before applying the change to production pipelines, verify that the selected identity has access to everything the pipeline needs, including:

* Fabric workspace items
* Connections
* Source systems
* Destination systems
* Notebooks or other items invoked by the pipeline
* Any APIs or resources accessed during execution

A pipeline that previously worked under a user identity can fail after changing `LastModifiedBy` if the new identity does not have equivalent access.

## Disclaimer

This notebook is provided as an example and utility for working with Microsoft Fabric.

Test it in a non-production workspace or against test pipelines before using it with production workloads. Microsoft Fabric APIs and identity behavior can change over time.
