# Infrastructure Nodes in OpenShift 4

Infrastructure nodes allow customers to isolate infrastructure workloads for two primary purposes:
  - to prevent incurring billing costs against subscription counts and
  - to separate maintenance and management.

All that is needed is a node label added to a particular node, set of nodes, or machines and machineset. 

Red Hat subscription vCPU counts omit any vCPU reported by a node labeled node-role.kubernetes.io/infra: "" and you will not be charged for these resources from Red Hat.

Workloads that can be scheduled on Infra nodes are EFK Log stack, Registry, Prometheus and Grafana,etc.
