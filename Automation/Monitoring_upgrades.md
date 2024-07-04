# Monitoring Openshift Cluster Upgrades through Tekton Pipeline   

### Openshift Cluster Upgrade Process
OpenShift upgrades the entire cluster by draining all nodes and rebooting using the latest rendered image. Even though all cluster operators may be in a Ready state, the upgrade is not considered complete until all nodes in the cluster are rebooted with the latest rendered image. Moreover, for major version upgrades, the cluster has to go through intermediate upgrade versions before reaching the desired version. Monitoring cluster upgrades can be tedious and time-consuming for production-grade clusters. This automation can help monitor the cluster upgrades. Additionally, this Tekton Pipeline can be customized to trigger the next version upgrade once the previous upgrade is complete. However, that can be an upgrade to this current pipeline.

### Tekton Pipeline
1. Task to Check MCP Status: This task monitors multiple MCPs (master, infra, worker) and uses the oc CLI for authentication. Save it with name check-mcp-status.yaml
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: check-mcp-status
spec:
  params:
    - name: CLUSTER_URL
      type: string
      description: The OpenShift cluster URL
    - name: SA_TOKEN
      type: string
      description: The service account token for authentication
  results:
    - name: master_status
      description: Result of the master MCP status check
    - name: infra_status
      description: Result of the infra MCP status check
    - name: worker_status
      description: Result of the worker MCP status check
  steps:
    - name: check-master-status
      image: quay.io/openshift/origin-cli:latest
      script: |
        #!/bin/bash
        set -e

        CLUSTER_URL=$(params.CLUSTER_URL)
        SA_TOKEN=$(params.SA_TOKEN)

        # Log in to the OpenShift cluster
        oc login --token=$SA_TOKEN --server=$CLUSTER_URL

        while true; do
          # Get the machine count and updated machine count for master MCP
          MASTER_MACHINE_COUNT=$(oc get machineconfigpool master -o jsonpath='{.status.machineCount}')
          MASTER_UPDATED_MACHINE_COUNT=$(oc get machineconfigpool master -o jsonpath='{.status.updatedMachineCount}')

          echo "Master MCP Machine Count: $MASTER_MACHINE_COUNT"
          echo "Master MCP Updated Machine Count: $MASTER_UPDATED_MACHINE_COUNT"

          # Check if all machines are updated for master MCP
          if [ "$MASTER_MACHINE_COUNT" -eq "$MASTER_UPDATED_MACHINE_COUNT" ]; then
            echo "All machines are updated in the Master MCP"
            echo -n "Updated" > $(results.master_status.path)
            break
          else
            echo "Not all machines are updated in the Master MCP"
            sleep 30
          fi
        done

    - name: check-infra-status
      image: quay.io/openshift/origin-cli:latest
      script: |
        #!/bin/bash
        set -e

        CLUSTER_URL=$(params.CLUSTER_URL)
        SA_TOKEN=$(params.SA_TOKEN)

        # Log in to the OpenShift cluster
        oc login --token=$SA_TOKEN --server=$CLUSTER_URL

        while true; do
          # Get the machine count and updated machine count for infra MCP
          INFRA_MACHINE_COUNT=$(oc get machineconfigpool infra -o jsonpath='{.status.machineCount}')
          INFRA_UPDATED_MACHINE_COUNT=$(oc get machineconfigpool infra -o jsonpath='{.status.updatedMachineCount}')

          echo "Infra MCP Machine Count: $INFRA_MACHINE_COUNT"
          echo "Infra MCP Updated Machine Count: $INFRA_UPDATED_MACHINE_COUNT"

          # Check if all machines are updated for infra MCP
          if [ "$INFRA_MACHINE_COUNT" -eq "$INFRA_UPDATED_MACHINE_COUNT" ]; then
            echo "All machines are updated in the Infra MCP"
            echo -n "Updated" > $(results.infra_status.path)
            break
          else
            echo "Not all machines are updated in the Infra MCP"
            sleep 30
          fi
        done

    - name: check-worker-status
      image: quay.io/openshift/origin-cli:latest
      script: |
        #!/bin/bash
        set -e

        CLUSTER_URL=$(params.CLUSTER_URL)
        SA_TOKEN=$(params.SA_TOKEN)

        # Log in to the OpenShift cluster
        oc login --token=$SA_TOKEN --server=$CLUSTER_URL

        while true; do
          # Get the machine count and updated machine count for worker MCP
          WORKER_MACHINE_COUNT=$(oc get machineconfigpool worker -o jsonpath='{.status.machineCount}')
          WORKER_UPDATED_MACHINE_COUNT=$(oc get machineconfigpool worker -o jsonpath='{.status.updatedMachineCount}')

          echo "Worker MCP Machine Count: $WORKER_MACHINE_COUNT"
          echo "Worker MCP Updated Machine Count: $WORKER_UPDATED_MACHINE_COUNT"

          # Check if all machines are updated for worker MCP
          if [ "$WORKER_MACHINE_COUNT" -eq "$WORKER_UPDATED_MACHINE_COUNT" ]; then
            echo "All machines are updated in the Worker MCP"
            echo -n "Updated" > $(results.worker_status.path)
            break
          else
            echo "Not all machines are updated in the Worker MCP"
            sleep 30
          fi
        done
```
***(Note:- Sleep interval is 30 seconds. It might be too frequent. Please adjust as per your needs)***
2. *Task to Send Email*: This task sends an email notification once all machines are updated. Save it with name send-email.yaml
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: send-email
spec:
  params:
    - name: TO_EMAIL
      type: string
      description: The recipient email address
    - name: MCP_NAME
      type: string
      description: The name of the Machine Config Pool
    - name: SMTP_SERVER
      type: string
      description: The SMTP server address
    - name: SMTP_PORT
      type: string
      description: The SMTP server port
    - name: SMTP_USER
      type: string
      description: The SMTP user
    - name: SMTP_PASSWORD
      type: string
      description: The SMTP password
  steps:
    - name: send-email
      image: appropriate/curl:latest
      script: |
        #!/bin/sh
        TO_EMAIL=$(params.TO_EMAIL)
        MCP_NAME=$(params.MCP_NAME)
        SMTP_SERVER=$(params.SMTP_SERVER)
        SMTP_PORT=$(params.SMTP_PORT)
        SMTP_USER=$(params.SMTP_USER)
        SMTP_PASSWORD=$(params.SMTP_PASSWORD)

        EMAIL_SUBJECT="MCP Update Notification"
        EMAIL_BODY="All machines in the Machine Config Pool $MCP_NAME have been successfully updated."

        echo "Sending email to $TO_EMAIL..."

        curl --url "smtp://$SMTP_SERVER:$SMTP_PORT" --ssl-reqd \
          --mail-from "$SMTP_USER" \
          --mail-rcpt "$TO_EMAIL" \
          --upload-file <(echo -e "Subject: $EMAIL_SUBJECT\n\n$EMAIL_BODY") \
          --user "$SMTP_USER:$SMTP_PASSWORD"
```

3. Create a pipeline. Save it with name monitor-mcp-pipeline.yaml
```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: monitor-mcp-pipeline
spec:
  params:
    - name: MCP_NAME
      type: string
      description: The name of the Machine Config Pool
    - name: CLUSTER_URL
      type: string
      description: The OpenShift cluster URL
    - name: SA_TOKEN
      type: string
      description: The service account token for authentication
    - name: TO_EMAIL
      type: string
      description: The recipient email address
    - name: SMTP_SERVER
      type: string
      description: The SMTP server address
    - name: SMTP_PORT
      type: string
      description: The SMTP server port
    - name: SMTP_USER
      type: string
      description: The SMTP user
    - name: SMTP_PASSWORD
      type: string
      description: The SMTP password
  tasks:
    - name: check-mcp-status
      taskRef:
        name: check-mcp-status
      params:
        - name: MCP_NAME
          value: $(params.MCP_NAME)
        - name: CLUSTER_URL
          value: $(params.CLUSTER_URL)
        - name: SA_TOKEN
          value: $(params.SA_TOKEN)
      results:
        - name: status
    - name: send-email
      taskRef:
        name: send-email
      runAfter:
        - check-mcp-status
      params:
        - name: TO_EMAIL
          value: $(params.TO_EMAIL)
        - name: MCP_NAME
          value: $(params.MCP_NAME)
        - name: SMTP_SERVER
          value: $(params.SMTP_SERVER)
        - name: SMTP_PORT
          value: $(params.SMTP_PORT)
        - name: SMTP_USER
          value: $(params.SMTP_USER)
        - name: SMTP_PASSWORD
          value: $(params.SMTP_PASSWORD)
      when:
        - input: "$(tasks.check-mcp-status.results.status)"
          operator: in
          values:
            - "Updated"
```
4. Create a PipelineRun Definition. Save it as monitor-mcp-pipelinerun.yaml
```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: monitor-mcp-pipelinerun
spec:
  pipelineRef:
    name: monitor-mcp-pipeline
  params:
    - name: MCP_NAME
      value: <YOUR_MCP_NAME>
    - name: CLUSTER_URL
      value: <YOUR_CLUSTER_URL>
    - name: SA_TOKEN
      value: <YOUR_SA_TOKEN>
    - name: TO_EMAIL
      value: <RECIPIENT_EMAIL>
    - name: SMTP_SERVER
      value: <SMTP_SERVER_ADDRESS>
    - name: SMTP_PORT
      value: <SMTP_SERVER_PORT>
    - name: SMTP_USER
      value: <SMTP_USER>
    - name: SMTP_PASSWORD
      value: <SMTP_PASSWORD>
```
5. Apply all the yamls
```
oc apply -f check-mcp-status.yaml
oc apply -f send-email.yaml
oc apply -f monitor-mcp-pipeline.yaml
oc apply -f monitor-mcp-pipelinerun.yaml
```
