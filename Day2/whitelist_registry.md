# Adding specific registries
You can add a list of registries, and optionally an individual repository within a registry, that are permitted for image pull and push actions by editing the image.config.openshift.io/cluster custom resource (CR). OpenShift Container Platform applies the changes to this CR to all nodes in the cluster.

When pulling or pushing images, the container runtime searches the registries listed under the registrySources parameter in the image.config.openshift.io/cluster CR. If you created a list of registries under the allowedRegistries parameter, the container runtime searches only those registries. Registries not in the list are blocked.

1. Edit the image.config.openshift.io/cluster CR:
`oc edit image.config.openshift.io/cluster`
The following is an example image.config.openshift.io/cluster CR with an allowed list:
```
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  annotations:
    release.openshift.io/create-only: "true"
  creationTimestamp: "2019-05-17T13:44:26Z"
  generation: 1
  name: cluster
  resourceVersion: "8302"
  selfLink: /apis/config.openshift.io/v1/images/cluster
  uid: e34555da-78a9-11e9-b92b-06d6c7da38dc
spec:
  registrySources: 
    allowedRegistries: 
    - example.com
    - quay.io
    - registry.redhat.io
    - reg1.io/myrepo/myapp:latest
    - image-registry.openshift-image-registry.svc:5000
status:
  internalRegistryHostname: image-registry.openshift-image-registry.svc:5000
```
2. To check that the registries have been added to the policy file, use the following command on a node:
`cat /host/etc/containers/policy.json`
