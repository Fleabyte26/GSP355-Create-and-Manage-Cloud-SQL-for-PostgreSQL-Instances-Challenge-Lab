# GSP355: Create and Manage Cloud SQL for PostgreSQL Instances - Challenge Lab Guide

> **Lab Name:** Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab  
> **Course / Quest:** Manage PostgreSQL Databases on Cloud SQL  
> **Lab Code:** GSP355  
> **Estimated Completion Time:** ~12–15 minutes (using CLI automation)  
> **Target Score:** 100 / 100  

---

## 1. Executive Summary & "Why This Lab Fails"

This challenge lab assesses database migration and security operations on Google Cloud Platform:
1. Configuring a self-hosted PostgreSQL 14 instance on Compute Engine for logical replication.
2. Migrating the standalone database (`orders`) to Cloud SQL for PostgreSQL using Database Migration Service (DMS) with continuous CDC (Change Data Capture).
3. Promoting the Cloud SQL replica to a standalone instance.
4. Securing database tables using Cloud IAM database authentication.
5. Enabling Point-in-Time Recovery (PITR) and testing disaster recovery by cloning to a rollback timestamp.

### Why Attempting This via the GCP Web Console Fails:
- **The Engine Selection Trap:** The DMS creation wizard defaults to *MySQL to Cloud SQL for MySQL*. If missed, the UI silently rejects or hides existing PostgreSQL connection profiles.
- **The Step 4 VPC Peering Hang:** Step 4 of the DMS wizard frequently spins indefinitely while allocating the Service Networking VPC peering range on the `default` VPC, burning through lab time.
- **Silent CDC Failure:** Without `output_plugin_libraries = 'pglogical_output'` in `postgresql.conf`, DMS passes pre-checks but fails silently when transitioning to CDC.
- **Missing Primary Keys:** Tables without primary keys (specifically `inventory_items`) cannot be replicated by `pglogical` in CDC mode.
- **PITR Cloned Instance Lag:** Cloud SQL cloning can take 3–5 minutes while replaying WAL logs (`PENDING_CREATE`), leading learners to believe the command hung.

---

## 2. Lab Variables Reference Sheet

Before executing, gather your specific lab parameters from the Qwiklabs sidebar:

| Variable | Description | Example / Typical Value |
| :--- | :--- | :--- |
| `PROJECT_ID` | Active GCP Project ID | `qwiklabs-gcp-...` |
| `DEST_INSTANCE` | Target Cloud SQL Instance ID | `postgres10-...` |
| `REGION` | Cloud SQL & DMS Region | e.g., `us-east4` |
| `MIGRATION_USER` | PostgreSQL Replication User | `replication_user` |
| `MIGRATION_PASS` | Replication User Password | `DMS_1s_cool!` |
| `IAM_USER` | Qwiklabs Student IAM Email | `student-...@qwiklabs.net` |
| `SECURED_TABLE` | Table required for IAM auth | `inventory_items` |
| `PITR_DAYS` | Transaction log retention days | `3` |
| `CLONE_INSTANCE` | Pitr Clone Instance Name | `postgres-orders-pitr` |

---

## 3. Step-by-Step CLI Playbook

### Step 1: Destination Pre-Patching (Cloud Shell)
*Prevents the UI Step 4 freeze by pre-attaching VPC private networking.*

```bash
# Enable DMS and Service Networking APIs
gcloud services enable datamigration.googleapis.com servicenetworking.googleapis.com --quiet

# Pre-patch destination instance to default VPC
gcloud sql instances patch <DEST_INSTANCE>   --network=default   --no-assign-ip   --quiet
```

---

### Step 2: Source VM Configuration (SSH into `postgres-vm`)
*Installs pglogical, configures PostgreSQL flags, adds primary keys, and grants replication permissions.*

Connect via SSH to `postgres-vm` and run:

```bash
# 1. Install pglogical extension
sudo apt update && sudo apt install -y postgresql-14-pglogical

# 2. Append required network & replication settings
sudo su - postgres -c "gsutil cp gs://cloud-training/gsp355/pg_hba_append.conf . 2>/dev/null || gsutil cp gs://cloud-training/gsp918/pg_hba_append.conf ."
sudo su - postgres -c "gsutil cp gs://cloud-training/gsp355/postgresql_append.conf . 2>/dev/null || gsutil cp gs://cloud-training/gsp918/postgresql_append.conf ."
sudo su - postgres -c "cat pg_hba_append.conf >> /etc/postgresql/14/main/pg_hba.conf"
sudo su - postgres -c "cat postgresql_append.conf >> /etc/postgresql/14/main/postgresql.conf"

# Crucial parameters to prevent silent CDC replication failures:
echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf
echo "output_plugin_libraries = 'pglogical_output'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf

# Restart PostgreSQL service
sudo systemctl restart postgresql@14-main

# 3. Configure Database Permissions, Extensions & Primary Keys
sudo -u postgres psql << 'EOF'
\c postgres
CREATE EXTENSION IF NOT EXISTS pglogical;

\c orders
CREATE EXTENSION IF NOT EXISTS pglogical;
ALTER TABLE inventory_items ADD PRIMARY KEY (id);

CREATE USER replication_user WITH PASSWORD 'DMS_1s_cool!';
ALTER USER replication_user WITH SUPERUSER REPLICATION LOGIN;
ALTER DATABASE orders OWNER TO replication_user;

GRANT CONNECT ON DATABASE orders TO replication_user;
GRANT USAGE, CREATE ON SCHEMA public TO replication_user;
GRANT USAGE ON SCHEMA pglogical TO replication_user;

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO replication_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO replication_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA pglogical TO replication_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA pglogical TO replication_user;

ALTER TABLE public.distribution_centers OWNER TO replication_user;
ALTER TABLE public.inventory_items OWNER TO replication_user;
ALTER TABLE public.order_items OWNER TO replication_user;
ALTER TABLE public.products OWNER TO replication_user;
ALTER TABLE public.users OWNER TO replication_user;

\c postgres
GRANT USAGE, CREATE ON SCHEMA public TO replication_user;
GRANT USAGE ON SCHEMA pglogical TO replication_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO replication_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA pglogical TO replication_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA pglogical TO replication_user;
EOF

sudo systemctl restart postgresql@14-main
```

---

### Step 3: Create Connection Profile (Cloud Shell)

Run in **Cloud Shell**:

```bash
REGION=$(gcloud sql instances describe <DEST_INSTANCE> --format="value(region)")
VM_INTERNAL_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].networkIP)")

gcloud database-migration connection-profiles create postgresql vm-source     --region="$REGION"     --display-name="vmsource"     --host="$VM_INTERNAL_IP"     --port=5432     --username="replication_user"     --password="DMS_1s_cool!"     --no-async
```

---

### Step 4: Create, Start & Promote Continuous Migration Job

#### A. Create & Start the Migration Job (Cloud Console UI):

Go to Database Migration > Migration jobs and click + Create migration job.

Get started:

Job name: orders-migration

Source database engine: PostgreSQL

Destination database engine: Cloud SQL for PostgreSQL

Migration job type: Continuous

Define source: Select the vm-source profile you created.

Define destination: Select your active Cloud SQL instance (e.g., postgres50-...) and enter the root password (supersecret!).

Define connectivity: Select VPC peering and set the network to default.

Test and click Create & Start Job.

#### B. Monitor until CDC:
```bash
watch -n 5 "gcloud database-migration migration-jobs describe orders-migration --region=$REGION --format='table(state,phase)'"
```
* As soon as phase reaches **`CDC`**, hit `Ctrl + C` and promote:

```bash
gcloud database-migration migration-jobs promote orders-migration --region=$REGION --quiet
```

---

### Step 5: Implement Cloud IAM Database Authentication (Task 3)

#### A. In Cloud Shell:
```bash
VM_EXT_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].accessConfigs[0].natIP)")

gcloud sql instances patch <DEST_INSTANCE>   --authorized-networks="${VM_EXT_IP}/32"   --assign-ip   --database-flags=cloudsql.iam_authentication=on   --quiet

gcloud sql users create "<IAM_USER>"   --instance=<DEST_INSTANCE>   --type=CLOUD_IAM_USER
```

#### B. In `postgres-vm` SSH:
```bash
SQL_IP=$(gcloud sql instances describe <DEST_INSTANCE> --format="value(ipAddresses[0].ipAddress)")

PGPASSWORD='supersecret!' psql -h "$SQL_IP" -U postgres -d orders -c 'GRANT SELECT ON inventory_items TO "<IAM_USER>";'
```

---

### Step 6: Point-in-Time Recovery & Clone Testing (Task 4)

Run in **Cloud Shell**:

```bash
# 1. Enable PITR with specified retention days
gcloud sql instances patch <DEST_INSTANCE>   --backup-start-time 00:00   --enable-point-in-time-recovery   --retained-transaction-log-days 3   --quiet

# 2. Capture recovery point timestamp BEFORE insert
TIME_STAMP=$(date -u --rfc-3339=ns | sed -r 's/ /T/; s/\.([0-9]{3}).*/\.Z/')
echo "Recovery Point Timestamp: $TIME_STAMP"
```

#### Insert test row in `postgres-vm` SSH:
```bash
SQL_IP=$(gcloud sql instances describe <DEST_INSTANCE> --format="value(ipAddresses[0].ipAddress)")
PGPASSWORD='supersecret!' psql -h "$SQL_IP" -U postgres -d orders -c "INSERT INTO distribution_centers (name, latitude, longitude) VALUES ('Orlando Center', 28.5383, -81.3792);"
```

#### Trigger PITR clone in Cloud Shell:
```bash
gcloud sql instances clone <DEST_INSTANCE> postgres-orders-pitr   --point-in-time "$TIME_STAMP"

# Poll until clone instance is RUNNABLE
while [ "$(gcloud sql instances describe postgres-orders-pitr --format='value(state)' 2>/dev/null)" != "RUNNABLE" ]; do
  echo "Cloning instance from PITR logs... $(date +%T)"
  sleep 15
done
echo "Clone is ready!"
```

---

## 4. Key Troubleshooting Matrix

| Symptom | Root Cause | Immediate Resolution |
| :--- | :--- | :--- |
| **`No matches for ""` in DMS source connection profile dropdown** | Migration job created with default MySQL engine instead of PostgreSQL. | Cancel draft job, recreate with `Source database engine: PostgreSQL`. |
| **DMS Wizard spins infinitely on Step 4 (Connectivity)** | Cloud SQL VPC peering pending route allocation in Service Networking. | Pre-patch destination instance via `gcloud sql instances patch <INSTANCE> --network=default --no-assign-ip`. |
| **DMS job stuck at Full Dump or fails transitioning to CDC** | Missing `output_plugin_libraries = 'pglogical_output'` or table missing PK. | Add config to `postgresql.conf`, add PK to `inventory_items`, restart PostgreSQL. |
| **`HTTPError 409: The Cloud SQL instance already exists`** | Clone command aborted locally while GCP API accepted provisioning in background. | Monitor status via `gcloud sql instances describe postgres-orders-pitr --format="value(state)"` until `RUNNABLE`. |
| **`psql: connection timed out`** | Missing public IP or authorized network on destination instance. | Ensure `--assign-ip` is enabled and VM external IP CIDR is authorized. |
