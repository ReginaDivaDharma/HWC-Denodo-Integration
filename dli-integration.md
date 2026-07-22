# Connecting Denodo to Huawei Cloud DLI — Integration Guide

This guide describes how to establish bidirectional connectivity between **Denodo** (a data virtualization platform) and **Huawei Cloud Data Lake Insight (DLI)**. It covers the networking foundation, resource setup, driver configuration, and read/write operations in both directions. Each concept is briefly explained so the guide remains accessible to readers new to the Huawei Cloud ecosystem.

---

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Networking Setup](#step-1--networking-setup)
4. [Step 2 — Setting Up DLI Resources](#step-2--setting-up-dli-resources)
5. [Step 3 — VPC Peering and Authorization](#step-3--vpc-peering-and-authorization)
6. [Step 4 — Routing Configuration](#step-4--routing-configuration)
7. [Step 5 — Verify Connectivity](#step-5--verify-connectivity)
8. [Step 6 — Create an OBS Bucket](#step-6--create-an-obs-bucket)
9. [Step 7 — Upload the Denodo JDBC Driver](#step-7--upload-the-denodo-jdbc-driver)
10. [Step 8 — Write a PySpark Job to Query Denodo](#step-8--write-a-pyspark-job-to-query-denodo)
11. [Step 9 — Create an IAM Agency to Access OBS](#step-9--create-an-iam-agency-to-access-obs)
12. [Step 10 — Submit the Spark Job](#step-10--submit-the-spark-job)
13. [Step 11 — Review the Output](#step-11--review-the-output)
14. [Step 12 — Connecting a DLI Table Back into Denodo](#step-12--connecting-a-dli-table-back-into-denodo)
15. [Step 13 — Modifying DLI Tables via Denodo (CRUD)](#step-13--modifying-dli-tables-via-denodo-crud)

---

## Overview

The objective is to enable Denodo and DLI to communicate so that data can be queried and managed from a single interface. Three components are involved:

```
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│   DENODO    │ ◀──────▶ │  Huawei DLI │ ◀──────▶ │  OBS        │
│ (Query      │          │ (Processing │          │ (Data       │
│  interface) │          │   engine)   │          │  storage)   │
└─────────────┘          └─────────────┘          └─────────────┘
```

- **Denodo** is the query and data virtualization layer.
- **DLI (Data Lake Insight)** is Huawei Cloud's data processing engine, which executes the queries.
- **OBS (Object Storage Service)** is where the underlying data files are stored.

---

## Prerequisites

**Important:** Confirm that you have permission to create an **Agency in IAM** (Identity and Access Management) before you begin. If you do not, coordinate with your Master Account administrator in advance. Resolving this later in the process causes significant delays.

You will need:

- A Huawei Cloud account with DLI and OBS enabled
- A running Denodo instance with a public IP address
- Administrator or equivalent IAM permissions
- Basic familiarity with cloud consoles (no coding is required until Step 8)

---

## Step 1 — Networking Setup

### Purpose

By default, DLI operates within an isolated network and cannot reach the public internet or your Denodo instance. To enable outbound connectivity, you will configure a **NAT Gateway** as the network bridge.

### Procedure

**A. Create an Enterprise Project**

An Enterprise Project groups related resources for organization and tracking. Create one before provisioning any other resources.

**B. Create a VPC (Virtual Private Cloud)**

A VPC is your private network on Huawei Cloud. Create a standard VPC with a name of your choosing.

**C. Set Up a NAT Gateway**

The NAT Gateway provides DLI with outbound internet access:

1. Purchase a **NAT Gateway** within your VPC.
2. Bind an **EIP (Elastic IP)** to the gateway. This is the public IP address DLI will use for outbound traffic.
3. Add an **SNAT Rule** to enable outbound internet traffic.

**Reference:** The EIP is a fixed public address. The SNAT Rule authorizes the network to initiate outbound connections to the internet.

---

## Step 2 — Setting Up DLI Resources

This step provisions the compute environment in which your jobs will run.

### A. Purchase a Resource Pool

A Resource Pool provides the compute capacity DLI uses to execute jobs.

- For testing, **16 CU** is sufficient and cost-effective.
- Ensure it is created in the **same Enterprise Project** as the VPC from Step 1.

### B. Purchase a DLI Queue

A Queue is a dedicated execution lane for your jobs. Begin with the **smallest configuration** for initial testing.

### C. Associate the Queue with the Resource Pool

1. Open the **Resource Pool** page.
2. Select **More → Associate Queue**.
3. Choose the queue you created.

The Resource Pool supplies the compute capacity; the Queue is the target to which jobs are submitted. The two must be linked.

---

## Step 3 — VPC Peering and Authorization

This step connects DLI's network to your VPC so the two can communicate.

### A. Set Up Agency Authorization

An **Agency** grants one Huawei Cloud service permission to act on behalf of another.

1. Open **DLI Settings**.
2. Locate the **Agency** section.
3. Confirm that all three agency settings are authorized.

![Agency Settings](https://github.com/user-attachments/assets/9a954bc3-1f86-45e8-98c8-2ed0c7b4aada)

### B. Set Up VPC Peering

VPC Peering establishes a direct private connection between DLI's network and your VPC.

1. Open **Datasource Connection** in DLI.
2. Create a new connection.
3. Enter your **NAT Gateway VPC details**.
4. Ensure your **Resource Pool** is selected in the form.

![VPC Peering Setup](https://github.com/user-attachments/assets/b7fb7e68-a528-4f63-88da-9d1c7eef3bd5)

---

## Step 4 — Routing Configuration

**Official Huawei documentation for this step:**
https://support.huaweicloud.com/eu/bestpractice-dli/dli_05_0061.html

Even after VPC Peering is configured, the DLI Queue does not automatically know how to reach the internet or the Denodo instance. The routing below provides the required directions.

### A. Add a Second SNAT Rule for the DLI Queue

Return to the **NAT Gateway** and add a new **SNAT Rule** specific to your DLI Queue subnet:

| Field | Value |
|---|---|
| **Scenario** | Direct Connect / Cloud Connect |
| **Subnet** | The subnet where your DLI queue resides |
| **EIP** | The EIP bound to your NAT Gateway |

![SNAT Configuration](https://github.com/user-attachments/assets/ea7339ba-6433-41fe-b536-a23e24e37420)

### B. Add a Custom Route

This defines the path from the DLI Queue to the Denodo instance.

1. Open the **DLI Dashboard**.
2. Select **Manage Route**.
3. Select **Add Custom Route**.
4. Enter the **public IP address of your Denodo instance**.

Without this route, DLI has internet access but no defined path to the Denodo instance specifically.

---

## Step 5 — Verify Connectivity

Confirm that the network path is functioning before proceeding.

1. In the DLI Queue list, select **More → Test Address Connectivity**.
2. Enter the **public IP address of your Denodo instance**.
3. A successful result confirms the connection is established.

![Connectivity Test](https://github.com/user-attachments/assets/bea764fd-cec1-4273-9ebf-ea8647f2c1d2)

The networking stage is now complete.

---

## Step 6 — Create an OBS Bucket

OBS (Object Storage Service) is Huawei Cloud's file storage service. You will use an OBS bucket to store:

- The Spark job script (the query code)
- The Denodo JDBC driver (which enables DLI to communicate with Denodo)
- Job logs

**To create the bucket:**

1. Open the OBS console.
2. Select **Create Bucket**.
3. Choose the **Standard** storage type (recommended for frequently accessed data).
4. Record the bucket path; it is required in later steps.

![OBS Bucket](https://github.com/user-attachments/assets/349188f4-868b-47b0-9b27-54f141e3bd33)

---

## Step 7 — Upload the Denodo JDBC Driver

### About JDBC Drivers

A **JDBC driver** is a `.jar` file that acts as a translator between two systems. DLI requires the Denodo driver to communicate with Denodo.

### Procedure

**1. Download the driver**

Download the JDBC driver from the Denodo community portal:
https://community.denodo.com/drivers/jdbc/9

**2. Upload it to OBS**

Upload the `.jar` file to your OBS bucket, ideally within a clearly named folder such as `/drivers/`.

![Driver in OBS](https://github.com/user-attachments/assets/933f219b-ac54-42ef-80d0-066842da53ee)

**3. Register it in DLI**

1. Open **DLI Dashboard → Package Management**.
2. Select **Create Package**.
3. Set the path to the `.jar` file in OBS.
4. Set **Type** to `JAR`.
5. For **Group**, select **Do not use**.

![Package Management](https://github.com/user-attachments/assets/024a4873-8ce9-4e03-baaa-9812b9c5393c)

---

## Step 8 — Write a PySpark Job to Query Denodo

### About PySpark

**PySpark** is Python code that runs on Apache Spark, a distributed data processing engine. Because DLI runs on Spark, you provide a Python script to define the job.

### The Script

Create a file named `query_denodo.py` with the following content:

```python
from pyspark.sql import SparkSession

# Start a Spark session (the entry point for all Spark jobs)
spark = SparkSession.builder \
    .appName("denodo_query") \
    .getOrCreate()

# Connect to Denodo and read a table
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:denodo://YOUR_DENODO_IP:9999/admin") \  # Replace with your Denodo IP
    .option("dbtable", "your_table_name") \                       # Replace with your table name
    .option("user", "your_username") \                            # Replace with your username
    .option("password", "your_password") \                        # Replace with your password
    .option("driver", "com.denodo.vdp.jdbc.Driver") \
    .load()

# Print the result
df.show()

spark.stop()
```

**Note:** One script is required per table. Copy this template and change the `dbtable` value for each table you intend to query.

Upload the script to OBS once complete. A dedicated `/jobs/` folder is recommended.

---

## Step 9 — Create an IAM Agency to Access OBS

### Purpose

DLI requires explicit permission to read files from your OBS bucket. Without it, the Spark job will fail because it cannot access the driver file or the script.

### Step A — Obtain Your AK/SK Credentials

The **AK (Access Key)** and **SK (Secret Key)** function as credentials for Huawei Cloud APIs and are used to authenticate.

Locate them under **My Account → Access Keys**.

### Step B — Store Your AK/SK Securely in DEW

**DEW (Data Encryption Workshop)** is Huawei's secure secret storage service.

1. Open the **DEW Console → Cloud Secret Management Service → Secrets**.
2. Select **Create Secret**.
3. Add two key-value pairs:
   - Key 1: Your Access Key (AK)
   - Key 2: Your Secret Access Key (SK)

![DEW Secret](https://github.com/user-attachments/assets/2704005e-3fdd-4bc4-aa1d-9e3856e30bc3)

### Step C — Create the IAM Agency

1. Open the **IAM Console → Agencies → Create Agency**.
2. Complete the fields:

| Field | Value |
|---|---|
| **Agency Name** | `dli_dew_agency_access` (or a name of your choosing) |
| **Agency Type** | Cloud service |
| **Cloud Service** | Data Lake Insight (DLI) |
| **Validity Period** | Unlimited |

3. Select **Next → Permissions tab → Authorize → Create Policy**.
4. Set **Format** to JSON and paste the following:

```json
{
    "Version": "1.1",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "csms:secretVersion:get",
                "csms:secretVersion:list",
                "kms:dek:decrypt"
            ]
        }
    ]
}
```

5. Select **Next**. On the **Select Scope** page, search for **OBS** and select all the required permissions.

![OBS Permissions 1](https://github.com/user-attachments/assets/460501fc-2852-41b0-89bd-5919640c2823)
![OBS Permissions 2](https://github.com/user-attachments/assets/95891811-5dec-4240-b4cf-aea5a61f1576)

6. Select **OK**.

**Note:** Permissions may take **15–30 minutes** to take effect.

---

## Step 10 — Submit the Spark Job

1. Open the **DLI Console → Job Management → Spark Jobs → Create Job**.
2. Complete the job configuration:

| Field | Value |
|---|---|
| **Queue** | The queue you created |
| **Spark Version** | `3.3.1` (required for DEW agent support) |
| **Application** | The path to your `.py` script in OBS |
| **Agency** | The IAM agency created in Step 9 |
| **Jar Dependencies** | The path to your Denodo JDBC `.jar` file in OBS |

**Important:** When adding the Jar dependency, **enter the OBS path manually rather than using the dropdown**. A known issue causes the dropdown selection to produce an error even when it appears correct.

![Job Configuration](https://github.com/user-attachments/assets/afdcafc6-bd13-49e4-9762-955981150b7c)

3. Select **Execute** in the top-right corner.
4. Accept the privacy agreement and select **OK**.

---

## Step 11 — Review the Output

Once the job completes:

1. Select your Spark job.
2. Open the **More** dropdown.
3. Select **Driver Log**.
4. Scroll down to view the queried table data in the log output.

![Result](https://github.com/user-attachments/assets/a916e50a-1b3b-464c-af4b-fc60a89de79c)

The Denodo data is now accessible through DLI.

---

## Step 12 — Connecting a DLI Table Back into Denodo

This section covers the reverse direction: making a DLI table queryable from Denodo.

### A. Obtain Your AK/SK Credentials

You will need your **Access Key (AK)** and **Secret Key (SK)**, as in Step 9A.

### B. Construct the DLI JDBC Connection String

Denodo connects to DLI via JDBC. Use the following template to build the connection URL:

```
jdbc:dli://{dli-endpoint}/{project-id}?regionname={region};authenticationmode=aksk;databasename={database};queuename={queue}
```

**Example:**

```
jdbc:dli://dli.ap-southeast-4.myhuaweicloud.com/abc123?regionname=ap-southeast-4;authenticationmode=aksk;databasename=dummy_data;queuename=queue_spark
```

**Note:** The full list of endpoints is available here:
https://console-intl.huaweicloud.com/apiexplorer/#/endpoint

### C. Install the DLI JDBC Driver in Denodo

**1. Import the driver**

In Denodo Design Studio:

1. Open **File → Extension Management**.
2. Select **Library → Import**.
3. Set **Resource Type** to `JDBC_other`.
4. Provide a name and select **OK**.

**2. Configure the driver class**

Open **Configuration → Advanced** and set the **Driver Class** as shown below.

![Denodo Driver Class Configuration](https://github.com/user-attachments/assets/5a544a6f-f59e-4c5f-ac20-dd1c352adefe)

**3. Create a Base View**

1. Open your new DLI data source in Denodo.
2. Select **Create Base View**.
3. Select your database and table.
4. Create the view. The DLI table is now queryable from Denodo.

---

## Step 13 — Modifying DLI Tables via Denodo (CRUD)

To avoid switching between consoles, you can manage data directly through Denodo. This section describes how to perform **CRUD** (Create, Read, Update, Delete) operations from Denodo against Huawei Cloud DLI.

### Prerequisites

To modify data, the DLI table must support **ACID transactions**.

- **Required format:** Use **Hudi** (recommended for DLI).
- **The `_rt` rule:** Hudi maintains two table versions. Always use the **Real-Time (`_rt`)** table (for example, `food_rt`) in Denodo to reflect updates immediately.

**Viewing the full table:** DLI requires a partition filter to return data. To view the entire table, use a non-empty condition on the partition column:

```sql
WHERE your_partition_column != ''
```

Example:

```sql
WHERE city != ''
```

### 1. Insert (Create)

Adding new data uses a standard SQL statement.

**VQL command:**

```sql
INSERT INTO bv_food_rt (id, name, category, price, city)
VALUES (3, 'Pisang Goreng', 'Snack', 20000.0, 'Jakarta');
```

**Verification:** Checking both DLI and Denodo confirms the data is successfully ingested.

![Insert Verification](https://github.com/user-attachments/assets/0b2a7dcc-9937-4b00-88fb-7689509b94c4)

### 2. Update

When updating, always include the **partition key** (for example, `city`) so DLI can locate the correct record.

**VQL command:**

```sql
UPDATE bv_food_rt
SET price = 45000.0
WHERE id = 1 AND city = 'Jakarta';
```

**Verification:** The price update is reflected in the DLI console.

![Update Verification](https://github.com/user-attachments/assets/29a1cfb1-a5aa-480b-a9f2-ae5af8d0024e)

### 3. Delete

Removing a row also requires the **partition key** to satisfy DLI's constraints.

**VQL command:**

```sql
DELETE FROM bv_food_rt
WHERE id = 10 AND city = 'Bali';
```

**Verification:** After execution, the row is permanently removed from the data lake.

![Delete Verification](https://github.com/user-attachments/assets/b3cd1e2c-66a2-4a3b-a572-dc6affdcc4d4)

---

## Final Result

The data in Denodo now aligns with the corresponding DLI table.

![Final Result](https://github.com/user-attachments/assets/0dce9f13-2c84-4090-ae5b-1b4dd2f7f82e)

---

*Last updated: April 2026*

*For questions, contact regina.diva333@gmail.com*
