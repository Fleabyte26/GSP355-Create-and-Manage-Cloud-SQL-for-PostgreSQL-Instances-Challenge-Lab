# GSP355: Create and Manage Cloud SQL for PostgreSQL Instances - Challenge Lab Guide

* **Lab Name:** Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab
* **Lab Code:** GSP355
* **Estimated Completion Time:** ~12–15 minutes
* **Target Score:** 100 / 100

---

### Step 0: Initialize Dynamic Environment Variables (Cloud Shell)

👉 **PASTE INTO CLOUD SHELL:**

```bash
export PROJECT_ID=$(gcloud config get-value project)
export IAM_USER=$(gcloud config get-value account)
export DEST_INSTANCE=$(gcloud sql instances list --format="value(name)" | head -n 1)
export REGION=$(gcloud sql instances describe$DEST_INSTANCE --format="value(region)")
export ZONE=$(gcloud compute instances list --filter="name~postgres" --format="value(zone.basename())" | head -n 1)
export VM_INTERNAL_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].networkIP)")
export VM_EXT_IP=$(gcloud compute instances list --filter="name~postgres" --format="value(networkInterfaces[0].accessConfigs[0].natIP)")

gcloud config set compute/region "$REGION"
gcloud config set compute/zone "$ZONE"

echo "----------------------------------------"
echo "PROJECT ID:    $PROJECT_ID"
echo "DEST INSTANCE: $DEST_INSTANCE"
echo "REGION:        $REGION"
echo "ZONE:          $ZONE"
echo "STUDENT USER:  $IAM_USER"
echo "INTERNAL IP:   $VM_INTERNAL_IP"
echo "----------------------------------------"
