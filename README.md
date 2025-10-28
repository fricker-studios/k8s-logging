# Internal ElasticSearch/Kibana Log Shipping for K8s
This repo serves as a starting point for implementing a log shipping service inside of an existing K8s cluster, which sends container logs to an Elastic/Kibana cluster.

The manifests in this repo leverage the existing [Logging Operator](https://ot-logging-operator.netlify.app/) from Opstree to set up the elastic cluster in K8s.

# Openshift
If deploying in Openshift, put the resources into a dedicated namespace and either:
- Grant the default user in this namespace the `anyuid` and `privileged` security context constraints
- Modify the manifests to use a dedicated service account, which is granted the `anyuid` and `privileged` security context constraints

In addition, grant the 'fluentd' service account the same `anyuid` and `privileged` security context constraints

For example:

```YAML
# ClusterRole granting `anyuid` and `privileged` security context constraints
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ocp-privileged-role
rules:
  - verbs:
      - use
    apiGroups:
      - security.openshift.io
    resources:
      - securitycontextconstraints
    resourceNames:
      - anyuid
      - privileged
---
# ClusterRoleBinding granting access to the service accounts
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ocp-privileged-bind
subjects:
  - kind: ServiceAccount
    name: default
    namespace: logging
  - kind: ServiceAccount
    name: fluentd
    namespace: logging
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ocp-privileged-role
```

# Kibana Token
The Kibana access token is not generated automatically - you will need to remote into one of the `elasticsearch-master` containers and generate a token. For example:

```bash
curl -u elastic:'<password>' -k -X POST https://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token
```

Store the output of this command into a Secret named `elasticsearch-token`, under the `token` key:

```YAML
kind: Secret
apiVersion: v1
metadata:
  name: elasticsearch-token
  namespace: logging
data:
  token: ---------
type: Opaque
```