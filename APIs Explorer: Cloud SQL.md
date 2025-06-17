#  APIs Explorer: Create and Update a Cluster

## Solution [Click here](https://youtu.be/CK5M0tEOouA)

```bash
export REGION=$(gcloud compute project-info describe \
--format="value(commonInstanceMetadata.items[google-compute-default-region])")

read -r -d '' REQUEST_BODY <<EOF
{
  "name": "my-instance",
  "region": "$REGION",
  "settings": {
    "tier": "db-n1-standard-1"
  }
}
EOF

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "$REQUEST_BODY" \
  "https://sqladmin.googleapis.com/sql/v1beta4/projects/$DEVSHELL_PROJECT_ID/instances"

check_sql_instance_status() {

  while true; do
    status=$(gcloud sql instances describe "my-instance" \
      --project "$DEVSHELL_PROJECT_ID" \
      --format="get(state)")

    echo "Current status: $status"

    if [[ "$status" == "RUNNABLE" ]]; then
      echo "Instance is RUNNABLE!"
      break
    elif [[ "$status" == "FAILED" || "$status" == "SUSPENDED" ]]; then
      echo "Instance creation failed or suspended. Status: $status"
      break
    fi

    sleep 60
  done
}

check_sql_instance_status

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "mysql-db"
      }' \
  "https://sqladmin.googleapis.com/sql/v1beta4/projects/$DEVSHELL_PROJECT_ID/instances/my-instance/databases"

BUCKET_NAME="bucket-$(date +%s)"  # unique name

IP_ADDRESS=$(gcloud sql instances describe "my-instance" \
  --format="value(ipAddresses[0].ipAddress)")

if [[ -z "$IP_ADDRESS" ]]; then
  echo "No public IP found. Enable Public IP on your SQL instance."
  exit 1
fi

MY_IP=$(curl -s ifconfig.me)

gcloud sql instances patch "my-instance" \
  --authorized-networks="$MY_IP/32" --quiet

mysql -h "$IP_ADDRESS" -u root "mysql-db" <<EOF
CREATE TABLE IF NOT EXISTS info (
  name VARCHAR(255),
  age INT,
  occupation VARCHAR(255)
);
EOF

cat > employee_info.csv <<EOF
"Sean", 23, "Content Creator"
"Emily", 34, "Cloud Engineer"
"Rocky", 40, "Event coordinator"
"Kate", 28, "Data Analyst"
"Juan", 51, "Program Manager"
"Jennifer", 32, "Web Developer"
EOF

gsutil mb -l "$REGION" -p "$DEVSHELL_PROJECT_ID" gs://"$BUCKET_NAME"

gsutil cp employee_info.csv gs://"$BUCKET_NAME"/

SERVICE_ACCOUNT=$(gcloud sql instances describe "my-instance" \
  --project="$DEVSHELL_PROJECT_ID" \
  --format="value(serviceAccountEmailAddress)")

if [[ -z "$SERVICE_ACCOUNT" ]]; then
  echo "Could not find the service account for the SQL instance."
  exit 1
fi

gsutil iam ch "serviceAccount:$SERVICE_ACCOUNT:roles/storage.admin" gs://"$BUCKET_NAME"
```
