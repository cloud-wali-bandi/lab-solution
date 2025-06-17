#  APIs Explorer: Create and Update a Cluster

## Solution [Click here](https://youtu.be/AmHHB7a-Jsg)

```bash
export ZONE=$(gcloud compute project-info describe \
--format="value(commonInstanceMetadata.items[google-compute-default-zone])")

export REGION=$(gcloud compute project-info describe \
--format="value(commonInstanceMetadata.items[google-compute-default-region])")

read -r -d '' REQUEST_BODY <<EOF
{
  "clusterName": "mydataproccluster",
  "config": {
    "gceClusterConfig": {
      "zoneUri": "https://www.googleapis.com/compute/v1/projects/$DEVSHELL_PROJECT_ID/zones/$ZONE"
    },
    "softwareConfig": {
      "imageVersion": "2.0-debian10",
      "optionalComponents": ["JUPYTER"]
    }
  }
}
EOF

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "$REQUEST_BODY" \
  "https://dataproc.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/regions/$REGION/clusters"

check_cluster_status() {

  while true; do
    status=$(gcloud dataproc clusters describe "mydataproccluster" \
      --region "$REGION" \
      --project "$DEVSHELL_PROJECT_ID" \
      --format="get(status.state)")

    echo "${BOLD}${BLUE}Current Cluster Status: $status${RESET}"

    if [[ "$status" == "RUNNING" ]]; then
      echo "${BOLD}${GREEN}Cluster is now RUNNING.${RESET}"
      break
    elif [[ "$status" == "ERROR" || "$status" == "DELETING" || "$status" == "UNKNOWN" ]]; then
      echo "${BOLD}${RED}Cluster entered terminal state: $status${RESET}"
      break
    fi

    sleep 30
  done
}

check_cluster_status

read -r -d '' REQUEST_BODY <<EOF
{
  "job": {
    "placement": {
      "clusterName": "mydataproccluster"
    },
    "sparkJob": {
      "mainClass": "org.apache.spark.examples.SparkPi",
      "args": ["1000"],
      "jarFileUris": ["file:///usr/lib/spark/examples/jars/spark-examples.jar"]
    }
  }
}
EOF

curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "$REQUEST_BODY" \
  "https://dataproc.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/regions/$REGION/jobs:submit"

read -r -d '' PATCH_BODY <<EOF
{
  "config": {
    "workerConfig": {
      "numInstances": 3
    }
  }
}
EOF

curl -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "$PATCH_BODY" \
  "https://dataproc.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/regions/$REGION/clusters/mydataproccluster?updateMask=config.worker_config.num_instances"
```
