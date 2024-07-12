# Monitoring Openshift upgrades using CronJobs

# Pre-Requisites 
1. Cronjob service account should have permission to monitor Machine Config Pools.
2. A separate namespace.
3. An Open Network connection in case the SMTP server reside in a different Subnet.

## Procedure
1. Create a shell script to check mcp status *monitor_mcp.sh*.
```
#!/bin/bash

# Set your email address
EMAIL="your_email@example.com"

# Set SMTP details
SMTP_SERVER="smtp.example.com"
SMTP_PORT="587"
SMTP_USER="smtp_user@example.com"
SMTP_PASS="smtp_password"

# MCP names to monitor
MCP_NAMES=("infra" "master" "worker")

# Function to send email
send_email() {
  local CLUSTER_VERSION=$1
  local BODY=$2

  SUBJECT="Openshift Cluster Upgraded to version $CLUSTER_VERSION"
  echo "$BODY" | curl --url "smtp://$SMTP_SERVER:$SMTP_PORT" \
                      --ssl-reqd \
                      --mail-from "$SMTP_USER" \
                      --mail-rcpt "$EMAIL" \
                      --user "$SMTP_USER:$SMTP_PASS" \
                      -T - \
                      -k --insecure
}

# Get the OpenShift cluster version
CLUSTER_VERSION=$(oc get clusterversion version -o jsonpath='{.status.desired.version}')

# Check counts for all MCPs
all_equal=true
BODY=""

for MCP_NAME in "${MCP_NAMES[@]}"; do
  MACHINE_COUNT=$(oc get mcp $MCP_NAME -o jsonpath='{.status.machineCount}')
  UPDATED_MACHINE_COUNT=$(oc get mcp $MCP_NAME -o jsonpath='{.status.updatedMachineCount}')

  if [ "$MACHINE_COUNT" -ne "$UPDATED_MACHINE_COUNT" ]; then
    all_equal=false
    break
  fi

  BODY+="Machine count equals updated machine count for MCP $MCP_NAME. "
done

if [ "$all_equal" = true ]; then
  send_email $CLUSTER_VERSION "$BODY"
fi
```

2. Create a config map to store the script
`oc create configmap monitor-mcp-script  --from-file=path/to/monitor_mcp.sh`

3. Create service account with appropriate permission.
```oc create serviceaccount <your-service-account>
oc adm policy add-cluster-role-to-user <role> -z <service-account>
```
***(Note:- Can use Cluster Reader role, However it doesn't comply with least privilege rule. Try to customise a role with required permission.)***

4. Update the CronJob to run the updated script every 30 minutes:
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: monitor-mcp
  namespace: your-namespace
spec:
  schedule: "*/30 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: monitor-mcp
            image: quay.io/openshift/origin-cli:latest
            command: ["/bin/bash", "-c", "/scripts/monitor_mcp.sh"]
            volumeMounts:
            - name: script-volume
              mountPath: /scripts
              readOnly: true
            env:
            - name: KUBECONFIG
              value: /var/run/secrets/kubernetes.io/serviceaccount/token
          restartPolicy: OnFailure
          volumes:
          - name: script-volume
            configMap:
              name: monitor-mcp-script
              defaultMode: 0755
          serviceAccountName: your-service-account
```
Once the update is complete cron job can be suspended with the following command :
`oc patch cronjob monitor-mcp -p '{"spec": {"suspend": true}}' -n your-namespace`
