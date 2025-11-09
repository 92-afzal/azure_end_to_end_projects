# Fix Summary: Databricks ADLS2 Mount & External Location Issue

## Problem
You were unable to mount or connect Azure Data Lake Storage Gen2 (ADLS2) to Databricks. The main issues were:
- `dbutils.fs.mount` failed with `Py4JSecurityException` because you were on a shared Unity Catalog cluster.
- `IllegalArgumentException: Unsupported Azure Scheme: abfss` errors occurred when using the wrong context.
- Unity Catalog’s default external location (`intech_dbw`) could not be edited and returned errors about breaking managed storage.

---

## Root Causes
1. **Wrong cluster type:** You were using a shared (non-dedicated) cluster, which doesn’t support DBFS mounts.
2. **Unity Catalog protection:** The default external location `intech_dbw` was tied to Unity Catalog’s system-managed storage and couldn’t be changed.
3. **Permission issues:** The managed identity initially lacked the proper Azure roles on your ADLS2 account.
4. **Wrong storage path:** The Unity Catalog storage account (`dbstoragespjyvhjmypfbs`) was mistakenly used instead of your actual data storage account (`intechstg`).

---

## Step-by-Step Fix

### **1. Verify Storage Account and Folder**
- Your real data was in the storage account: `intechstg`
- Inside the container: `bronze`
- Folder path: `SalesLT`

Correct ADLS URI:
```
abfss://bronze@intechstg.dfs.core.windows.net/SalesLT/
```

---

### **2. Check Azure Role Assignments**
In the Azure Portal → `intechstg` → **Access Control (IAM)**:

Assign the following roles:
- **Role:** `Storage Blob Data Contributor`
- **Assigned to:** The managed identity connected to Databricks Unity Catalog
  - Name: `unity-catalog-access-connector`

Other identities like `dbmanagedidentity`, `intech-s`, or your user account can remain, but they’re not required for Databricks access.

---

### **3. Confirm the Managed Identity**
- Verified Managed Identity: `dbmanagedidentity` (Linked to resource group `databricks-rg-intech-dbw-tk3bargyb6zys`)
- Confirmed subscription and region matched your Databricks workspace.

---

### **4. Create a New External Location in Databricks**
You could not modify the existing one (`intech_dbw`), so you created a new one.

**In Databricks → Catalog → External Data → External Locations → Create:**

| Field | Value |
|-------|--------|
| **External location name** | `bronze_saleslt` |
| **Storage type** | Azure Data Lake Storage |
| **URL** | `abfss://bronze@intechstg.dfs.core.windows.net/SalesLT/` |
| **Storage credential** | `intech_dbw` (Managed Identity) |

✅ **Validation result:** All checks passed (`Read`, `List`, `Write`, `Delete`, `Path Exists`, `Hierarchical Namespace Enabled`).

---

### **5. Test Connection and Validate Access**
You got all green checks confirming the managed identity had full access.

Databricks now recognized your external location properly.

---

### **6. (Optional) Read Data from the External Location**
You can now use either SQL or PySpark:

```python
df = spark.read.parquet("abfss://bronze@intechstg.dfs.core.windows.net/SalesLT/")
display(df)
```

or

```sql
CREATE TABLE main.default.saleslt
USING PARQUET
LOCATION 'abfss://bronze@intechstg.dfs.core.windows.net/SalesLT/';
```

---

## ✅ Final State
- Cluster: Dedicated / Single-user (with Unity Catalog enabled)
- External location: `bronze_saleslt`
- Storage credential: `intech_dbw` (Managed Identity)
- Validation: **All permissions confirmed**

You can now safely access, query, and register your ADLS2 data in Databricks Unity Catalog without mount errors or permission failures.

