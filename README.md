Markdown
# GSP355: Create and Manage Cloud SQL for PostgreSQL Instances - Challenge Lab Guide

* **Lab Name:** Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab
* **Lab Code:** GSP355
* **Estimated Completion Time:** ~12–15 minutes
* **Target Score:** 100 / 100

---
GSP355: Create and Manage Cloud SQL for PostgreSQL Instances - Challenge Lab Guide
Lab Name: Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab

Lab Code: GSP355

Estimated Completion Time: ~12–15 minutes

Target Score: 100 / 100

Step 0: Initialize Dynamic Environment Variables (Cloud Shell)
Discovers lab resources dynamically and sets your default region and zone.

Bash
export PROJECT_ID=$(gcloud config get-value project)
export IAM_USER=$(gcloud config get-value account)

export VM_NAME=$(gcloud compute instances list --filter="name~postgres" --format="value(name)" | head -n 1)
export ZONE=$(gcloud compute instances list --filter="name~postgres" --format="value(zone.basename())" | head -n 1)
export VM_INTERNAL_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].networkIP)" | head -n 1)
export VM_EXT_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | head -n 1)

export REGION=${ZONE%-*}
export DEST_INSTANCE=$(gcloud sql instances list --format="value(name)" | head -n 1)

gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"

echo "----------------------------------------"
echo "PROJECT ID:    $PROJECT_ID"
echo "STUDENT USER:  $IAM_USER"
echo "SOURCE VM:     $VM_NAME"
echo "DEST INSTANCE: $DEST_INSTANCE"
echo "REGION:        $REGION"
echo "ZONE:          $ZONE"
echo "INTERNAL IP:   $VM_INTERNAL_IP"
echo "EXTERNAL IP:   $VM_EXT_IP"
echo "----------------------------------------"
Step 1: Pre-Patch Destination Instance (Cloud Shell)
Enables APIs and binds the Cloud SQL destination instance to the default VPC network.

Bash
gcloud services enable datamigration.googleapis.com servicenetworking.googleapis.com --quiet
gcloud sql instances patch "$DEST_INSTANCE" --network=default --no-assign-ip --quiet
Step 2: Configure Source VM & Database (Cloud Shell)
Installs pglogical, appends required configurations, creates replication users, and adds missing keys.

Bash
cat << 'EOF' > vm_setup.sh
#!/bin/bash
sudo apt update && sudo apt install -y postgresql-14-pglogical

sudo su - postgres -c "gsutil cp gs://cloud-training/gsp355/pg_hba_append.conf . 2>/dev/null || gsutil cp gs://cloud-training/gsp918/pg_hba_append.conf ."
sudo su - postgres -c "gsutil cp gs://cloud-training/gsp355/postgresql_append.conf . 2>/dev/null || gsutil cp gs://cloud-training/gsp918/postgresql_append.conf ."
sudo su - postgres -c "cat pg_hba_append.conf >> /etc/postgresql/14/main/pg_hba.conf"
sudo su - postgres -c "cat postgresql_append.conf >> /etc/postgresql/14/main/postgresql.conf"

echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf
echo "output_plugin_libraries = 'pglogical_output'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf

sudo systemctl restart postgresql@14-main

sudo -u postgres psql << 'SQL'
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
SQL

sudo systemctl restart postgresql@14-main
EOF

gcloud compute scp vm_setup.sh "$VM_NAME":~/vm_setup.sh --zone="$ZONE" --quiet
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --quiet --command="chmod +x ~/vm_setup.sh && ~/vm_setup.sh"
Step 3: Create Source Connection Profile (Cloud Shell)
Creates the Database Migration Service connection profile pointing to the source database.

Bash
gcloud database-migration connection-profiles create postgresql vm-source \
    --region="$REGION" \
    --display-name="vmsource" \
    --host="$VM_INTERNAL_IP" \
    --port=5432 \
    --username="replication_user" \
    --password="DMS_1s_cool!" \
    --no-async
Step 4: Create, Monitor & Promote Continuous Migration Job
A. Create in Cloud Console UI
Navigate to Database Migration > Migration jobs and click + Create migration job.

Settings:

Job name: orders-migration

Source database engine: PostgreSQL

Destination database engine: Cloud SQL for PostgreSQL

Migration job type: Continuous

Define source: Select vm-source.

Define destination: Select destination instance and enter root password: supersecret!

Connectivity: Select VPC peering, network: default.

Click Test Job, then Create & Start Job.

🟢 CHECK PROGRESS: Check Task 1 once job status shows Starting or Running.

B. Monitor CDC and Promote (Cloud Shell)
Bash
echo "Waiting for migration job 'orders-migration' to reach CDC phase..."
while [ "$(gcloud database-migration migration-jobs describe orders-migration --region="$REGION" --format='value(phase)' 2>/dev/null)" != "CDC" ]; do
  echo "Current Phase: $(gcloud database-migration migration-jobs describe orders-migration --region="$REGION" --format='value(phase)' 2>/dev/null) - waiting 15s..."
  sleep 15
done

echo "CDC reached. Promoting migration job..."
gcloud database-migration migration-jobs promote orders-migration --region="$REGION" --quiet

while [ "$(gcloud sql instances describe "$DEST_INSTANCE" --format='value(state)' 2>/dev/null)" != "RUNNABLE" ]; do
  echo "Waiting for destination instance to become RUNNABLE..."
  sleep 10
done
echo "Destination is RUNNABLE."
🟢 CHECK PROGRESS: Check Task 2 (Promote continuous migration job).

Step 5: Implement Cloud IAM Database Authentication (Cloud Shell)
Configures IAM database authentication, creates the IAM database user, and grants SELECT permissions.

Bash
gcloud sql instances patch "$DEST_INSTANCE" \
    --authorized-networks="${VM_EXT_IP}/32" \
    --assign-ip \
    --database-flags=cloudsql.iam_authentication=on \
    --quiet

gcloud sql users create "$IAM_USER" \
    --instance="$DEST_INSTANCE" \
    --type=CLOUD_IAM_USER

SQL_IP=$(gcloud sql instances describe "$DEST_INSTANCE" --format="value(ipAddresses[0].ipAddress)")

gcloud compute ssh "$VM_NAME" --zone="$ZONE" --quiet --command="PGPASSWORD='supersecret!' psql -h '$SQL_IP' -U postgres -d orders -c 'GRANT SELECT ON inventory_items TO \"$IAM_USER\";'"
🟢 CHECK PROGRESS: Check Task 3 (Implement Cloud IAM Database Authentication).

Step 6: Point-in-Time Recovery & Clone Testing (Cloud Shell)
Configures point-in-time recovery on Cloud SQL, inserts test data, and provisions a point-in-time clone.

Bash
gcloud sql instances patch "$DEST_INSTANCE" \
    --backup-start-time 00:00 \
    --enable-point-in-time-recovery \
    --retained-transaction-log-days 3 \
    --quiet

TIME_STAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
echo "Recovery Point Timestamp: $TIME_STAMP"

SQL_IP=$(gcloud sql instances describe "$DEST_INSTANCE" --format="value(ipAddresses[0].ipAddress)")

gcloud compute ssh "$VM_NAME" --zone="$ZONE" --quiet --command="PGPASSWORD='supersecret!' psql -h '$SQL_IP' -U postgres -d orders -c \"INSERT INTO distribution_centers (name, latitude, longitude) VALUES ('Orlando Center', 28.5383, -81.3792);\""

gcloud sql instances clone "$DEST_INSTANCE" postgres-orders-pitr --point-in-time "$TIME_STAMP"

while [ "$(gcloud sql instances describe postgres-orders-pitr --format='value(state)' 2>/dev/null)" != "RUNNABLE" ]; do
  echo "Cloning instance from PITR logs... $(date +%T)"
  sleep 15
done

echo "Clone is ready! Challenge lab complete."
🟢 CHECK PROGRESS: Check Task 4 (100 / 100 Points).
