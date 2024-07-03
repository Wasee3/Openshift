# Infrastructure Nodes in OpenShift 4

Infrastructure nodes allow customers to isolate infrastructure workloads for two primary purposes:
  - to prevent incurring billing costs against subscription counts and
  - to separate maintenance and management.

All that is needed is a node label added to a particular node, set of nodes, or machines and machineset. 

Red Hat subscription vCPU counts omit any vCPU reported by a node labeled node-role.kubernetes.io/infra: "" and you will not be charged for these resources from Red Hat.

Workloads that can be scheduled on Infra nodes are EFK Log stack, Registry, Prometheus and Grafana,etc.

### Isolating Infrastructure Nodes
Applying a specific node selector to all infrastructure components will guarantee that they will be scheduled on nodes with that label.

Our node label and matching selector for infrastructure components will be `node-role.kubernetes.io/infra: ""`.

To prevent other workloads from also being scheduled on those infrastructure nodes, we need one of two solutions:

- Apply a taint to the infrastructure nodes and tolerations to the desired infrastructure workloads.
OR
- Apply a completely separate label to your other nodes and matching node selector to your other workloads such that they are mutually exclusive from infrastructure nodes.

### About the "worker" role and the MachineConfigPool
By default all nodes except for masters will be labeled with `node-role.kubernetes.io/worker: ""`. We will be adding `node-role.kubernetes.io/infra: ""` to infrastructure nodes.

The MachineConfigOperator reconciles any available MachineConfigs defined to match a specific selector in a MachineConfigPool. All custom MCP objects will descend from the parent worker pool as documented in custom pools. You do not need a custom pool unless you actually need to change a specific node to have a different set of MachineConfigs applied to it.

Machine configs are applied to Machine config Pool via label selector, Hence any custom settings exclusively on Infra Machines can be applied via Machine Config and labeling them `node-role.kubernetes.io/infra: ""`.

This infra MCP definition below will find all MachineConfigs labeled both "worker" and "infra" and it will apply them to any Machines or Nodes that have the "infra" role label. In this manner, you will ensure that your infra nodes can upgrade without the "worker" role label.
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: infra
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,infra]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/infra: ""
```

With MachineSets
If your cluster was installed using MachineSets to manage your Machines and Nodes, then you can use MachineSets to define your infrastructure nodes as well.

An example MachineSet with the required nodeSelector and taints applied might look like this:
```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    machine.openshift.io/cluster-api-cluster: <infrastructureID> 
  name: <infrastructureID>-infra-<zone> 
  namespace: openshift-machine-api
spec:
  replicas: 1
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: <infrastructureID> 
      machine.openshift.io/cluster-api-machineset: <infrastructureID>-infra-<zone> 
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: <infrastructureID> 
        machine.openshift.io/cluster-api-machine-role: infra 
        machine.openshift.io/cluster-api-machine-type: infra 
        machine.openshift.io/cluster-api-machineset: <infrastructureID>-infra-<zone> 
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/infra: ""
          node-role.kubernetes.io: infra
      taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
        value: reserved
      - effect: NoExecute
        key: node-role.kubernetes.io/infra
        value: reserved
```

### Without MachineSets
If you are not using the MachineSet API to manage your nodes, labels and taints are applied manually to each node:

Label it:
```
oc label node <node-name> node-role.kubernetes.io/infra=
oc label node <node-name> node-role.kubernetes.io=infra
```
Taint it
```
oc adm taint nodes -l node-role.kubernetes.io/infra node-role.kubernetes.io/infra=reserved:NoSchedule node-role.kubernetes.io/infra=reserved:NoExecute
```

### Moving Components to the Infrastructure Nodes
To move components to the infrastructure nodes, they must now have the infra Node Selector and a Toleration for the Taint assigned to the infrastructure nodes.
The following is an example, taken from the IngressController default, of what should be included in each respective resource spec to apply the node selector and toleration:
```
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
      value: reserved
    - effect: NoExecute
      key: node-role.kubernetes.io/infra
      value: reserved
```

### Registry
To move the registry, apply the following patch to the config/cluster object.
```
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"nodeSelector": {"node-role.kubernetes.io/infra": ""},"tolerations": [{"effect":"NoSchedule","key": "node-role.kubernetes.io/infra","value": "reserved"},{"effect":"NoExecute","key": "node-role.kubernetes.io/infra","value": "reserved"}]}}'
```

### Monitoring
Prometheus, Grafana and AlertManager comprise the default monitoring stack. To move these components create a config map with the required Node Selectors and Tolerations.

Define the ConfigMap as the cluster-monitoring-configmap.yaml file with the following:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
    thanosQuerier:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoSchedule
      - key: node-role.kubernetes.io/infra
        value: reserved
        effect: NoExecute
```
Then apply it to the cluster:
`oc create -f cluster-monitoring-configmap.yaml`

