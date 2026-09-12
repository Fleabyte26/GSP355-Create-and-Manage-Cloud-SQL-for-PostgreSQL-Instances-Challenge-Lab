# GSP355: Create and Manage Cloud SQL for PostgreSQL Instances - Challenge Lab

---

### Step 0: Setup Environment Variables

👉 CUT & PASTE INTO CLOUD SHELL:

export ZONE=$(gcloud compute project-info describe --format="value(commonInstanceMetadata.items[google-compute-default-zone])"); export REGION=$(gcloud compute project-info describe --format="value(commonInstanceMetadata.items[google-compute-default-region])"); [ -z "$REGION" ] && export REGION="${ZONE%-*}"; export PROJECT_ID=$(gcloud config get-value project); export DB_USER=$(gcloud config get-value account); export DB_NAME="orders"; export MIGRATION_JOB="orders-migration"; export CLOUD_SQL_INSTANCE="postgres-orders"; gcloud config set compute/region "$REGION"; gcloud config set compute/zone "$ZONE"; echo "----------------------------------------"; echo "PROJECT ID : $PROJECT_ID"; echo "ZONE       : $ZONE"; echo "REGION     : $REGION"; echo "USER       : $DB_USER"; echo "INSTANCE   : $CLOUD_SQL_INSTANCE"; echo "----------------------------------------"

### Task 1: Migrate Standalone PostgreSQL to Cloud SQL

👉 CUT & PASTE INTO CLOUD SHELL (SSH INTO VM):

gcloud compute ssh postgres-vm --zone=$ZONE --quiet

---

👉 CUT & PASTE INSIDE POSTGRES-VM:

sudo -u postgres psql -d orders -c "ALTER TABLE distribution_centers ADD PRIMARY KEY (id);"
sudo -u postgres psql -d orders -c "CREATE EXTENSION IF NOT EXISTS pglogical;"
exit

---

👉 CUT & PASTE INTO CLOUD SHELL:

VM_INTERNAL_IP=$(gcloud compute instances describe postgres-vm --zone=$ZONE --format='get(networkInterfaces[0].networkIP)')

gcloud database-migration connection-profiles create postgresql postgres-vm-profile \
    --region=$REGION \
    --host=$VM_INTERNAL_IP \
    --port=5432 \
    --username=postgres \
    --password="supersecret!" \
    --provider=POSTGRESQL

gcloud database-migration migration-jobs create $MIGRATION_JOB \
    --region=$REGION \
    --source=postgres-vm-profile \
    --destination-instance-id=$CLOUD_SQL_INSTANCE \
    --type=CONTINUOUS \
    --connectivity-vpc=default

gcloud database-migration migration-jobs start $MIGRATION_JOB --region=$REGION

---

🟢 WHEN TO CLICK CHECK MY PROGRESS (TASK 1):
* As soon as the command completes and your terminal returns to student_xx@cloudshell:~$
* Click Task 1. It turns green.

---

### Task 2: Promote Cloud SQL to Stand-Alone Instance

👉 CUT & PASTE INTO CLOUD SHELL (MONITOR SYNC):

watch -n 5 "gcloud database-migration migration-jobs describe $MIGRATION_JOB --region=\$(gcloud config get-value compute/region) --format='table(state,phase)'"

---

👉 ACTION REQUIRED IN TERMINAL:
* Watch the table on screen.
* Wait until STATE shows RUNNING and PHASE shows CDC.
* Press Ctrl + C on your keyboard to exit the watch screen.

---

👉 CUT & PASTE INTO CLOUD SHELL (PROMOTE):

gcloud database-migration migration-jobs promote $MIGRATION_JOB --region=$REGION --quiet

---

👉 CUT & PASTE INTO CLOUD SHELL (VERIFY READINESS):

gcloud sql instances describe $CLOUD_SQL_INSTANCE --format="value(state)"

---

🟢 WHEN TO CLICK CHECK MY PROGRESS (TASK 2):
* Wait until the verify command outputs RUNNABLE.
* Click Task 2. It turns green.

---

### Task 3: Secure the Database Using IAM DB Authentication

👉 CUT & PASTE INTO CLOUD SHELL:

gcloud sql instances patch $CLOUD_SQL_INSTANCE --database-flags=cloudsql.iam_authentication=on

VM_EXT_IP=$(gcloud compute instances describe postgres-vm --zone=$ZONE --format='get(networkInterfaces[0].accessConfigs[0].natIP)')

gcloud sql instances patch $CLOUD_SQL_INSTANCE --authorized-networks=$VM_EXT_IP

gcloud sql users create $DB_USER --instance=$CLOUD_SQL_INSTANCE --type=CLOUD_IAM_USER

gcloud sql connect $CLOUD_SQL_INSTANCE --user=postgres --quiet

---

👉 MANUAL ACTION:
* When prompted for password, enter: supersecret!

---

👉 CUT & PASTE INSIDE PSQL (POSTGRES PROMPT):

\c orders;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "$DB_USER";
\q

---

🟢 WHEN TO CLICK CHECK MY PROGRESS (TASK 3):
* As soon as \q brings you back to the student_xx@cloudshell:~$ prompt.
* Click Task 3. It turns green.

---

### Task 4: Configure Point-in-Time Recovery & Clone Instance

👉 CUT & PASTE INTO CLOUD SHELL:

gcloud sql instances patch $CLOUD_SQL_INSTANCE --enable-point-in-time-recovery --enable-bin-log

gcloud sql instances clone $CLOUD_SQL_INSTANCE ${CLOUD_SQL_INSTANCE}-pitr

---

🟢 WHEN TO CLICK CHECK MY PROGRESS (TASK 4):
* As soon as the clone command finishes and returns to your shell prompt.
* Click Task 4. You will reach 100/100 points!
