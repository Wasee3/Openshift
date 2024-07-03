# Configuring chrony time service

You can set the time server and related settings used by the chrony time service (chronyd) by modifying the contents of the chrony.conf file and passing those contents to your nodes as a machine config.

1. Create a Butane config including the contents of the chrony.conf file. For example, to configure chrony on worker nodes, create a 99-worker-chrony.bu file.
```
variant: openshift
version: 4.14.0
metadata:
  name: 99-worker-chrony 
  labels:
    machineconfiguration.openshift.io/role: worker 
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        pool 0.rhel.pool.ntp.org iburst 
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony
```
2. Use Butane to generate a MachineConfig object file, 99-worker-chrony.yaml, containing the configuration to be delivered to the nodes:
`butane 99-worker-chrony.bu -o 99-worker-chrony.yaml`
3. Apply the configurations
`oc apply -f ./99-worker-chrony.yaml`
