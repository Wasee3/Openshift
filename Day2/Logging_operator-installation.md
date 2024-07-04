# Installing the Red Hat OpenShift Logging Operator

1. Create a Namespace object as a YAML file:
```
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat 
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring: "true" 
```
2. Apply the Namespace object by running the following command:
`oc apply -f <filename>.yaml`
3. Create a Namespace object for the Red Hat OpenShift Logging Operator:
```
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-logging
  annotations:
    openshift.io/node-selector: ""
  labels:
    openshift.io/cluster-monitoring: "true"
```

4. Apply the Namespace object by running the following command:
`oc apply -f <filename>.yaml`

5. Create an OperatorGroup object as a YAML file:
```
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cluster-logging
  namespace: openshift-logging 
spec:
  targetNamespaces:
  - openshift-logging 
```

6. Apply the OperatorGroup object by running the following command:
`oc apply -f <filename>.yaml`

7. Create a Subscription object to subscribe the namespace to the Red Hat OpenShift Logging Operator:
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging 
spec:
  channel: stable 
  name: cluster-logging
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
```

8. Apply the subscription by running the following command:
`oc apply -f <filename>.yaml`

Once the Logging Operator is installed. A Custom Resource(CR) called ClusterLogging is installed.
Create ClusterLogging CR to instantiate the EFK/Lokistack pods.

# Creating a ClusterLogging object

1. Create a ClusterLogging object as a YAML file:

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance                               #The name must be instance.
  namespace: openshift-logging                 #The OpenShift Logging management state. In some cases, if you change the OpenShift Logging defaults, you must set this to Unmanaged. However, an unmanaged deployment does not receive updates until OpenShift Logging is placed back into a managed state.
spec:
  managementState: Managed 
  logStore:
    type: elasticsearch                         #Settings for configuring Elasticsearch. Using the CR, you can configure shard replication policy and persistent storage.
    retentionPolicy:                            #Specify the length of time that Elasticsearch should retain each log source. Enter an integer and a time designation: weeks(w), hours(h/H), minutes(m) and seconds(s). For example, 7d for seven days. Logs older than the maxAge are deleted. You must specify a retention policy for each log source or the Elasticsearch indices will not be created for that source.

      application:
        maxAge: 1d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
    elasticsearch:
      nodeCount: 3                             #Specify the number of Elasticsearch nodes. See the note that follows this list.
      storage:
        storageClassName: <storage_class_name> #Enter the name of an existing storage class for Elasticsearch storage. For best performance, specify a storage class that allocates block storage. If you do not specify a storage class, OpenShift Logging uses ephemeral storage.
        size: 200G
      resources:                               #Specify the CPU and memory requests for Elasticsearch as needed. If you leave these values blank, the OpenShift Elasticsearch Operator sets default values that should be sufficient for most deployments. The default values are 16Gi for the memory request and 1 for the CPU request.
          limits:
            memory: 16Gi
          requests:
            memory: 16Gi
      proxy:                                   #Specify the CPU and memory requests for the Elasticsearch proxy as needed. If you leave these values blank, the OpenShift Elasticsearch Operator sets default values that should be sufficient for most deployments. The default values are 256Mi for the memory request and 100m for the CPU request.
        resources:
          limits:
            memory: 256Mi
          requests:
            memory: 256Mi
      redundancyPolicy: SingleRedundancy
  visualization:
    type: kibana                               #Settings for configuring Kibana. Using the CR, you can scale Kibana for redundancy and configure the CPU and memory for your Kibana nodes. For more information, see Configuring the log visualizer.
    kibana:
      replicas: 1
  collection:
    type: fluentd                             #Settings for configuring Fluentd. Using the CR, you can configure Fluentd CPU and memory limits. For more information, see "Configuring Fluentd".
    fluentd: {}
```
2. Create ClusterLogging CR
`oc apply -f <filename>.yaml`

***(Note:- These commands work for Recent versions of Openshift, For earlier version check Red Hat Openshift Official documentation)***
