# DNS Forwarding

You can use DNS forwarding to override the default forwarding configuration in the /etc/resolv.conf file in the following ways:

- Specify name servers for every zone. If the forwarded zone is the Ingress domain managed by OpenShift Container Platform, then the upstream name server must be authorized for the domain.
- Provide a list of upstream DNS servers.
- Change the default forwarding policy.

### Procedure
1. Modify the DNS Operator object named default:
    `oc edit dns.operator/default`
   After you issue the previous command, the Operator creates and updates the config map named dns-default with additional server configuration blocks based on Server. If none of the servers have a zone that matches the query, then name resolution falls back to the upstream DNS servers.
```
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  name: default
spec:
  servers:
  - name: example-server 
    zones: 
    - example.com
    forwardPlugin:
      policy: Random 
      upstreams: 
      - 1.1.1.1
      - 2.2.2.2:5353
  upstreamResolvers: 
    policy: Random 
    upstreams: 
    - type: SystemResolvConf 
    - type: Network
      address: 1.2.3.4 
      port: 53   
```
- line 20: Must comply with the rfc6335 service name syntax.
- line 21: Must conform to the definition of a subdomain in the rfc1123 service name syntax. The cluster domain, cluster.local, is an invalid subdomain for the zones field.
- line 24: Defines the policy to select upstream resolvers. Default value is Random. You can also use the values RoundRobin, and Sequential.
- line 25: A maximum of 15 upstreams is allowed per forwardPlugin.
- line 28: Optional. You can use it to override the default policy and forward DNS resolution to the specified DNS resolvers (upstream resolvers) for the default domain. If you do not provide any upstream resolvers, the DNS name queries go to the servers in /etc/resolv.conf.
- line 29: Determines the order in which upstream servers are selected for querying. You can specify one of these values: Random, RoundRobin, or Sequential. The default value is Sequential.
- line 30: Optional. You can use it to provide upstream resolvers.
- line 31: You can specify two types of upstreams - SystemResolvConf and Network. SystemResolvConf configures the upstream to use /etc/resolv.conf and Network defines a Networkresolver. You can specify one or both.
- line 33: If the specified type is Network, you must provide an IP address. The address field must be a valid IPv4 or IPv6 address.
- line 34: If the specified type is Network, you can optionally provide a port. The port field must have a value between 1 and 65535. If you do not specify a port for the upstream, by default port 853 is tried.
