---
title: Kyverno
date: 20230607
author: Adrián Martín García
---

# Kyverno
## Start minikube
```sh
$ minikube start -p kyverno --container-runtime=containerd --docker-opt containerd=/var/run/containerd/containerd.sock --driver=virtualbox
```

## Add Helm Repo and install it
```sh
$ helm repo add kyverno https://kyverno.github.io/kyverno/
$ helm repo update
$ cat <<EOF >> values.yaml
---
cleanupController:
  rbac:
    clusterRole:
      extraResources:
      - apiGroups:
          - 'apps'
        resources:
          - 'deployments'
EOF

$ helm upgrade --install kyverno -n kyverno -f values.yaml --create-namespace kyverno/kyverno --version 3.0.0
```

## Policies and Rules
A Kyverno policy is a collection of rules. Each rule consists of a `match` declaration, an optional `exclude` declaration, and one of a `validate`, `mutate`, `generate`, or `verifyImages` declaration. Each rule can contain only a single validate, mutate, generate, or verifyImages child declaration.

Las politicas se pueden definir como recurso de todo el clúster (`ClusterPolicy`) o recursos de espacio de nombres (`Policy`).

## Policy enforcement
Policies can be enforced in clusters or in pipelines. On installation a kyverno cluster runs as an admission controller, however rules can be evaluated via the Kyverno CLI.

### Configurations
* `applyRules`: indicates how many rules to apply One|All. 
* `background`: controls the scanning of existing resources. 
* `failurePolicy`: API Server behaviour if the webhook does not respond. 
* `generateExisting`: controls whether to evaluate the policy at the time it is created. 
* `mutateExistingOnPolicyUpdate`: evaluates rule when it is updated.
* `schemaValidation`: policy validation check.
* `validationFailureAction`: Enforce (blocking) or Audit (non-blocking).
* `validationFailureActionOverrides`: override validationFailureAction
* `webhookTimeoutSeconds`: maximum time to apply the policy.

*Use `kubectl explain policy.spec` for command-line help on the policy schema.*

### Resource selector
The match and exclude filters control the resources to which the policy is applied. 
* any: as **OR**
* all: as **AND**

Resource filters can be applied to: 
* resources: resources by name, namespaces, types, operations, tags, annotations, or selectors. 
* subjets: user, group, or service accounts. 
* roles 
* clusterRoles

```yaml
spec:
  rules:
  - name: no-LoadBalancer
    match:
      any:
      - resources:
          names: 
          - "prod-*"
          - "staging"
          kinds:
          - Service
          operations:
          - CREATE
      - resources:
          kinds:
          - Service
          operations:
          - CREATE
        subjects:
        - kind: User
          name: dave
```

### Validate Rules
These are the most common rules. The behaviour of how Kyverno responds to a validation is determined by the `validationFailureAction` field. Which can be blocked (`Enforce`) or allowed and registered (`Audit`).

You can use:

#### `Patterns`
* '*'
* '?'
* '?*'
* 'null'

#### `Operators`
* '>'	greater than
* '<'	less than
* '>='	greater than or equals to
* '<='	less than or equals to
* '!'	not equals
* '|'	logical or
* '&'	logical and
* '-'	within a range
* '!-'	outside a range

#### `Anchors` *("if-then-else")*
  * `Conditional`	**()**	If tag with the given value (including child elements) is specified, then peer elements will be processed.
e.g. If image has tag latest then imagePullPolicy cannot be IfNotPresent.
  * `Equality` **=()**	If tag is specified, then processing continues. For tags with scalar values, the value must match. For tags with child elements, the child element is further evaluated as a validation pattern.
  * `Existence`	**^()**	Works on the list/array type only. If at least one element in the list satisfies the pattern. In contrast, a conditional anchor would validate that all elements in the list match the pattern.
  * `Negation`	**X()**	The tag cannot be specified. The value of the tag is not evaluated (use exclamation point to negate a value). The value should ideally be set to "null" (quotes surrounding null).
  * `Global`	**<()**	The content of this condition, if false, will cause the entire rule to be skipped. Valid for both validate and strategic merge patches.

#### Required labels
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: req-labels
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Audit
  validationFailureActionOverrides:
    - action: Enforce
      namespaces:
        - req-labels
  rules:
  - name: check-team
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
EOF

# Create pod without labels
$ kubectl create deployment labels --image=nginx -n req-labels
error: failed to create deployment: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Deployment/req-labels/nginx was blocked due to the following policies 

require-labels:
  autogen-check-team: 'validation error: label ''team'' is required. rule autogen-check-team
    failed at path /spec/template/metadata/labels/team/'

# Create pod with labels
$ kubectl run labels --image nginx --labels team=backend -n req-labels 
pod/labels created
```

#### Check Resources
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: check-resources
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-container-resources
spec:
  validationFailureAction: Audit
  validationFailureActionOverrides:
    - action: Enforce
      namespaces:
        - check-resources
  rules:
  - name: check-container-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "All containers must have CPU and memory resource requests and limits defined."
      pattern:
        spec:
          containers:
          - name: "*"
            resources:
              limits:
                memory: "?*"
                cpu: "<=0.6"
EOF

# Create Pod without resources
$ kubectl run frontend --image resources -n check-resources
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Pod/default/nginx2 was blocked due to the following policies 

check-container-resources:
  check-container-resources: 'validation error: All containers must have CPU and memory
    resource requests and limits defined. rule check-container-resources failed at
    path /spec/containers/0/resources/limits/'

# Create Pod with resources
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: resources
  namespace: check-resources
  labels:
    team: resources
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: 0.1
      limits:
        memory: "128Mi"
        cpu: 0.5
EOF
```

#### Foreach
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion : kyverno.io/v2beta1
kind: ClusterPolicy
metadata:
  name: check-ingress
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: check-tls-secret-host
    match:
      any:
      - resources:
          kinds:
          - Ingress
    validate:
      message: "All TLS hosts must use a domain of old.com."  
      foreach:
      - list: request.object.spec.tls[]
        foreach:
        - list: "element.hosts"
          deny:
            conditions:
              all:
              - key: "{{element}}"
                operator: Equals
                value: "*.new.com"
EOF

# Create Ingress resource
$ cat <<EOF | kubectl create -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuard
  labels:
    app: kuard
spec:
  rules:
  - host: kuard.old.com
    http:
      paths:
      - backend:
          service: 
            name: kuard
            port: 
              number: 8080
        path: /
        pathType: ImplementationSpecific
  - host: hr.old.com
    http:
      paths:
      - backend:
          service: 
            name: kuard
            port: 
              number: 8090
        path: /myhr
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - kuard.old.com
    - kuard-foo.new.com
    secretName: foosecret.old.com
  - hosts:
    - hr.old.com
    secretName: hr.old.com
EOF

Error from server: error when creating "k8s_objects/validate/ingress.yaml": admission webhook "validate.kyverno.svc-fail" denied the request: 

resource Ingress/default/kuard was blocked due to the following policies 

check-ingress:
  check-tls-secret-host: 'validation failure: validation failure: All TLS hosts must
    use a domain of old.com.'
```

### Mutate
The mutate rule can be used to modify matching resources and is written as RFC 6902 JSON Patch or a strategic merge patch.

#### ImagePullPolicy
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: pull-policy
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: set-image-pull-policy
spec:
  rules:
    - name: set-image-pull-policy
      match:
        any:
        - resources:
            kinds:
            - Pod
            namespaces:
            - pull-policy
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              # match images which end with :latest
              - (image): "*:latest"
                # set the imagePullPolicy to "IfNotPresent"
                imagePullPolicy: "IfNotPresent"
EOF

# Create Pod
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: kyverno-pull-policy
  namespace: pull-policy
  labels:
    team: kyverno-pull-policy
spec:
  containers:
  - name: app
    image: nginx:latest
    imagePullPolicy: Always
    resources:
      requests:
        memory: "64Mi"
        cpu: 0.1
      limits:
        memory: "128Mi"
        cpu: 0.5

EOF

$ kubectl -n pull-policy get po kyverno-pull-policy -oyaml | grep -i imagePullPolicy:
    imagePullPolicy: IfNotPresent
```

#### NodeSelector
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: medium
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-nodeselector
spec:
  rules:
  - name: add-medium
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - medium
    mutate:
      patchStrategicMerge:
        spec:
          nodeSelector:
            type: medium
EOF

# Create Pod
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: kyverno-node-selector
  namespace: medium
  labels:
    team: kyverno-node-selector
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: 0.1
      limits:
        memory: "128Mi"
        cpu: 0.5
EOF

$ kubectl -n medium get po -oyaml | grep -i nodeselector -A1
    nodeSelector:
      type: medium
```

#### Tolerations
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: tolerations
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-tolerations
spec:
  rules:
  - name: service-toleration
    match:
      any:
      - resources:
          kinds:
          - Pod
          namespaces:
          - tolerations
    preconditions:
      any:
      - key: "app"
        operator: AnyNotIn
        value: "{{ request.object.spec.tolerations[].key || `[]` }}"
    mutate:
      patchesJson6902: |-
        - op: add
          path: "/spec/tolerations/-"
          value:
            key: app
            operator: Equal
            value: common
            effect: NoSchedule
EOF

# Create Pod
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerations
  namespace: tolerations
  labels:
    team: tolerations
spec:
  containers:
  - name: app
    image: nginx
EOF

$ kubectl -n medium get po -oyaml
...
    - effect: NoSchedule
      key: app
      operator: Equal
      value: common
```

### Generate
Generation rules can be used to create new Kubernetes resources.
#### Configmap
```sh
$ cat <<EOF | kubectl create -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: zk-kafka-address
spec:
  generateExisting: true
  rules:
  - name: k-kafka-address
    match:
      any:
      - resources:
          kinds:
          - Namespace
    exclude:
      any:
      - resources:
          namespaces:
          - kube-system
          - default
          - kube-public
          - kyverno
    generate:
      synchronize: true
      apiVersion: v1
      kind: ConfigMap
      name: zk-kafka-address
      # generate the resource in the new namespace
      namespace: "{{request.object.metadata.name}}"
      data:
        kind: ConfigMap
        metadata:
          labels:
            somekey: somevalue
        data:
          ZK_ADDRESS: "192.168.10.10:2181,192.168.10.11:2181,192.168.10.12:2181"
          KAFKA_ADDRESS: "192.168.10.13:9092,192.168.10.14:9092,192.168.10.15:9092"
EOF

$ kubectl get cm -A | grep zk-kafka-address
check-resources   zk-kafka-address                     2      14s
medium            zk-kafka-address                     2      14s
pull-policy       zk-kafka-address                     2      14s
req-labels        zk-kafka-address                     2      14s
tolerations       zk-kafka-address                     2      14s
```

#### Clone
```sh
# Create Secret
$ cat <<EOF | kubectl create -f -
apiVersion: v1
data:
  admin: cGFzc3M=
kind: Secret
metadata:
  creationTimestamp: null
  name: regcred
EOF

$ cat <<EOF | kubectl create -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-secrets
spec:
  rules:
  - name: sync-image-pull-secret
    match:
      any:
      - resources:
          kinds:
          - Namespace
    generate:
      apiVersion: v1
      kind: Secret
      name: regcred
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: regcred
EOF

# Create a new namespace
$ kubectl create ns sync-secret

# Get secrets
$ k -n sync-secret get secret
NAME      TYPE     DATA   AGE
regcred   Opaque   1      5s
```

### Verify Images
```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: kyverno.io/v2beta1
kind: ClusterPolicy
metadata:
  name: check-image-notary
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  failurePolicy: Fail  
  rules:
    - name: verify-signature-notary
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - type: Notary
        imageReferences:
        - "ghcr.io/kyverno/test-verify-image*"
        attestors:
        - count: 1
          entries:
          - certificates:
              cert: |-
                -----BEGIN CERTIFICATE-----
                MIIDTTCCAjWgAwIBAgIJAPI+zAzn4s0xMA0GCSqGSIb3DQEBCwUAMEwxCzAJBgNV
                BAYTAlVTMQswCQYDVQQIDAJXQTEQMA4GA1UEBwwHU2VhdHRsZTEPMA0GA1UECgwG
                Tm90YXJ5MQ0wCwYDVQQDDAR0ZXN0MB4XDTIzMDUyMjIxMTUxOFoXDTMzMDUxOTIx
                MTUxOFowTDELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAldBMRAwDgYDVQQHDAdTZWF0
                dGxlMQ8wDQYDVQQKDAZOb3RhcnkxDTALBgNVBAMMBHRlc3QwggEiMA0GCSqGSIb3
                DQEBAQUAA4IBDwAwggEKAoIBAQDNhTwv+QMk7jEHufFfIFlBjn2NiJaYPgL4eBS+
                b+o37ve5Zn9nzRppV6kGsa161r9s2KkLXmJrojNy6vo9a6g6RtZ3F6xKiWLUmbAL
                hVTCfYw/2n7xNlVMjyyUpE+7e193PF8HfQrfDFxe2JnX5LHtGe+X9vdvo2l41R6m
                Iia04DvpMdG4+da2tKPzXIuLUz/FDb6IODO3+qsqQLwEKmmUee+KX+3yw8I6G1y0
                Vp0mnHfsfutlHeG8gazCDlzEsuD4QJ9BKeRf2Vrb0ywqNLkGCbcCWF2H5Q80Iq/f
                ETVO9z88R7WheVdEjUB8UrY7ZMLdADM14IPhY2Y+tLaSzEVZAgMBAAGjMjAwMAkG
                A1UdEwQCMAAwDgYDVR0PAQH/BAQDAgeAMBMGA1UdJQQMMAoGCCsGAQUFBwMDMA0G
                CSqGSIb3DQEBCwUAA4IBAQBX7x4Ucre8AIUmXZ5PUK/zUBVOrZZzR1YE8w86J4X9
                kYeTtlijf9i2LTZMfGuG0dEVFN4ae3CCpBst+ilhIndnoxTyzP+sNy4RCRQ2Y/k8
                Zq235KIh7uucq96PL0qsF9s2RpTKXxyOGdtp9+HO0Ty5txJE2txtLDUIVPK5WNDF
                ByCEQNhtHgN6V20b8KU2oLBZ9vyB8V010dQz0NRTDLhkcvJig00535/LUylECYAJ
                5/jn6XKt6UYCQJbVNzBg/YPGc1RF4xdsGVDBben/JXpeGEmkdmXPILTKd9tZ5TC0
                uOKpF5rWAruB5PCIrquamOejpXV9aQA/K2JQDuc0mcKz
                -----END CERTIFICATE-----
EOF

# Verify image
kubectl run test --image=ghcr.io/kyverno/test-verify-image:signed --dry-run=server
pod/test created (server dry run)

$ kubectl run test --image=ghcr.io/kyverno/test-verify-image:signed --dry-run=server -o yaml | grep "image: "
  - image: ghcr.io/kyverno/test-verify-image:signed@sha256:b31bfb4d0213f254d361e0079deaaebefa4f82ba7aa76ef82e90b4935ad5b105

# Block pod with an unsigned image will be blocked
$ kubectl run test --image=ghcr.io/kyverno/test-verify-image:unsigned --dry-run=server
Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request: 

resource Pod/default/test was blocked due to the following policies 

check-image-notary:
  verify-signature-notary: 'failed to verify image ghcr.io/kyverno/test-verify-image:unsigned:
    .attestors[0].entries[0]: failed to verify ghcr.io/kyverno/test-verify-image@sha256:74a98f0e4d750c9052f092a7f7a72de7b20f94f176a490088f7a744c76c53ea5:
    no signature is associated with "ghcr.io/kyverno/test-verify-image@sha256:74a98f0e4d750c9052f092a7f7a72de7b20f94f176a490088f7a744c76c53ea5",
    make sure the image was signed successfully'
```

### Cleanup
Kyverno has the ability to clean up existing resources with a policy called `CleaunPolicy`. By default the CleanupController does not have permissions to delete deployments it will need:
```yaml
---
cleanupController:
  rbac:
    clusterRole:
      extraResources:
      - apiGroups:
          - 'apps'
        resources:
          - 'deployments'
```

```sh
# Create Kyverno ClusterCleanupPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: kyverno.io/v2alpha1
kind: ClusterCleanupPolicy
metadata:
  name: cleandeploy
spec:
  match:
    any:
    - resources:
        kinds:
          - Deployment
        selector:
          matchLabels:
            canremove: "true"
  conditions:
    any:
    - key: "{{ target.spec.replicas }}"
      operator: LessThan
      value: 2
  schedule: "*/5 * * * *"
EOF

# Create Deployment
$ cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: cleanup
    canremove: "true"
  name: cleanup
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cleanup
  template:
    metadata:
      labels:
        app: cleanup
    spec:
      containers:
      - image: nginx
        name: nginx
EOF

# Show logs
$ kubectl -n kyverno logs kyverno-cleanup-controller-56d7bcd685-wlwgq
...
I0608 08:51:46.932186       1 controller.go:71] cleanup-controller "msg"="added" "kind"="ClusterCleanupPolicy" "name"="cleandeploy" "operation"="added"
I0608 08:51:46.932583       1 controller.go:43] setup/cluster-cleanup-policy "msg"="resource added" "name"="cleandeploy" "type"="ClusterCleanupPolicy"
I0608 08:55:13.403812       1 handlers.go:89] cleanup "msg"="cleaning up..." "policy"="cleandeploy"
I0608 08:55:13.417117       1 handlers.go:216] cleanup "msg"="resource matched, it will be deleted..." "name"="cleanup" "namespace"="default" "policy"="cleandeploy"
I0608 08:55:13.424618       1 handlers.go:99] cleanup "msg"="done" "policy"="cleandeploy"
I0608 08:55:13.425265       1 event.go:307] "Event occurred" object="cleandeploy" fieldPath="" kind="ClusterCleanupPolicy" apiVersion="kyverno.io/v2alpha1" type="Normal" reason="PolicyApplied" message="successfully cleaned up the target resource Deployment/default/cleanup"
```

### Variables
Kyverno supports the definition of variables

```sh
# Create Kyverno ClusterPolicy
$ cat <<EOF | kubectl create -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: who-created-this
spec:
  background: false
  rules:
  - name: who-created-this
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          annotations:
            created-by: "{{request.userInfo.username}}"
EOF

# Create pod
$ kubectl run variable --image nginx

# Show annotations
$ kubectl get pod variable -o 
jsonpath='{.metadata.annotations}'
{"created-by":"minikube-user","policies.kyverno.io/last-applied-patch-this.kyverno.io: added /metadata/annotations\n"}
```

### External Data Sources
Get data from configmaps, the Kubernetes API, other cluster services and image logs.

#### Configmaps
```sh
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: external-data-source
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: some-config-map
  namespace: external-data-source
data:
  env: production
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: cm-variable-example
  annotations:
    pod-policies.kyverno.io/autogen-controllers: DaemonSet,Deployment,StatefulSet
spec:
    rules:
    - name: example-configmap-lookup
      match:
        any:
        - resources:
            kinds:
            - Pod
            namespaces:
            - external-data-source
      context:
      - name: dictionary
        configMap:
          name: some-config-map
          namespace: external-data-source
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              my-environment-name: "{{dictionary.data.env}}"
EOF

# Run Pod
$ kubectl run external-data-source --image nginx -n external-data-source

# Show labels
$ kubectl -n external-data-source get po --show-labels 
NAME                   READY   STATUS    RESTARTS   AGE   LABELS
external-data-source   1/1     Running   0          32s   my-environment-name=production,run=external-data-source
```

## Reporting
### Get and Describe result
```sh
$ kubectl get policyreport
NAME                  PASS   FAIL   WARN   ERROR   SKIP   AGE
cpol-require-labels   1      0      0      0       0      10m
```

```sh
$ kubectl describe policyreport
Name:         cpol-require-labels
Namespace:    default
Labels:       app.kubernetes.io/managed-by=kyverno
              cpol.kyverno.io/require-labels=862
Annotations:  <none>
API Version:  wgpolicyk8s.io/v1alpha2
Kind:         PolicyReport
Metadata:
  Creation Timestamp:  2023-06-07T08:44:08Z
  Generation:          1
  Managed Fields:
    API Version:  wgpolicyk8s.io/v1alpha2
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:app.kubernetes.io/managed-by:
          f:cpol.kyverno.io/require-labels:
      f:results:
      f:summary:
        .:
        f:error:
        f:fail:
        f:pass:
        f:skip:
        f:warn:
    Manager:         reports-controller
    Operation:       Update
    Time:            2023-06-07T08:44:08Z
  Resource Version:  1278
  UID:               ee1dd3fa-c526-4e98-88d1-1e6cbb3455d4
Results:
  Message:  validation rule 'check-team' passed.
  Policy:   require-labels
  Resources:
    API Version:  v1
    Kind:         Pod
    Name:         nginx
    Namespace:    default
    UID:          5ef94d4f-c84e-4b46-b12f-8956a134e53f
  Result:         pass
  Rule:           check-team
  Scored:         true
  Source:         kyverno
  Timestamp:
    Nanos:    0
    Seconds:  1686127448
Summary:
  Error:  0
  Fail:   0
  Pass:   1
  Skip:   0
  Warn:   0
Events:   <none>
```

## KyvernoCLI
```sh
# Install Kyverno CLI using kubectl krew plugin manager
kubectl krew install kyverno

# test the Kyverno CLI
kubectl kyverno version  
```

### Apply
#### Example 1
```yaml
# cli/example1/add_network_policy.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy
spec:
  background: false
  rules:
  - name: default-deny-ingress
    match:
      any:
      - resources:
          kinds:
          - Namespace
        clusterRoles:
        - cluster-admin
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny-ingress
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      data:
        spec:
          # select all pods in the namespace
          podSelector: {}
          policyTypes:
          - Ingress
---
# cli/example1/required_default_network_policy.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: devtest
---
# cli/example1/value.yaml 
policies:
  - name: add-networkpolicy
    resources:
      - name: devtest
        values:
          request.namespace: devtest
```

```sh
$ kubectl kyverno apply cli/example1/add_network_policy.yaml --resource cli/example1/required_default_network_policy.yaml -f cli/example1/value.yaml 

Applying 1 policy rule to 1 resource...

pass: 1, fail: 0, warn: 0, error: 0, skip: 0
```

#### Example 2 with Policy Report
```yaml
# cli/example2/enforce-pod-name.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-pod-name
spec:
  validationFailureAction: Audit
  background: true
  rules:
    - name: validate-name
      match:
        any:
        - resources:
            kinds:
              - Pod
          namespaceSelector:
            matchExpressions:
            - key: foo.com/managed-state
              operator: In
              values:
              - managed
      validate:
        message: "The Pod must end with -nginx"
        pattern:
          metadata:
            name: "*-nginx"
---
# cli/example2/nginx.yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-nginx
  namespace: test1
spec:
  containers:
  - name: nginx
    image: nginx:latest
---
# cli/example2/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test1
  labels:
    foo.com/managed-state: managed
---
# cli/example2/value.yaml
namespaceSelector:
  - name: test1
    labels:
      foo.com/managed-state: managed

```
```sh
$ kubectl kyverno apply cli/example2/enforce-pod-name.yaml --resource cli/example2/nginx.yaml -f cli/example2/value.yaml  --policy-report

Applying 1 policy rule to 1 resource...
----------------------------------------------------------------------
POLICY REPORT:
----------------------------------------------------------------------
apiVersion: wgpolicyk8s.io/v1alpha2
kind: ClusterPolicyReport
metadata:
  name: clusterpolicyreport
results:
- message: validation rule 'validate-name' passed.
  policy: enforce-pod-name
  resources:
  - apiVersion: v1
    kind: Pod
    name: test-nginx
    namespace: test1
  result: pass
  rule: validate-name
  scored: true
  source: kyverno
  timestamp:
    nanos: 0
    seconds: 1686217475
- message: The Pod must end with -nginx
  policy: enforce-pod-name
  resources:
  - apiVersion: v1
    kind: Pod
    name: test-nginx
    namespace: test1
  result: skip
  rule: autogen-validate-name
  scored: true
  source: kyverno
  timestamp:
    nanos: 0
    seconds: 1686217475
- message: The Pod must end with -nginx
  policy: enforce-pod-name
  resources:
  - apiVersion: v1
    kind: Pod
    name: test-nginx
    namespace: test1
  result: skip
  rule: autogen-cronjob-validate-name
  scored: true
  source: kyverno
  timestamp:
    nanos: 0
    seconds: 1686217475
summary:
  error: 0
  fail: 0
  pass: 1
  skip: 2
  warn: 0
```
