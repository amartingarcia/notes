---
title: Kyverno
date: 20230607
author: Adrián Martín García
---

# Kyverno
Kyverno is an open source policy engine for Kubernetes, a popular container orchestration platform. It provides a way to apply custom policies and rules to manage and secure Kubernetes resources.

With Kyverno, you can define policies as code using a declarative language. These policies are applied to Kubernetes resources at runtime, allowing you to enforce various configurations, security rules, and best practices.

## Políticas y Reglas
A policy is a collection of rules. Each rule consists of a `match` or `exclude` statement. And a `validate`, `mutate` or `generate` statement. It is also possible to specify `Verify Images` rules.

### Settings
Policies can contain one or more rules and you can specify some common settings:
* `applyRules`: indicates how many rules to apply One|All.
* `background`: controls the scanning of existing resources.
* `failurePolicy`: API Server behaviour if the webhook does not respond.
* `generateExisting`: controls whether to evaluate the policy at the time it is created.
* mutateExistingOnPolicyUpdate`: evaluates rule when it is updated.
* `schemaValidation`: policy validation check.
* `validationFailureAction`: Enforce (blocking) or Audit (non-blocking).
* `validationFailureActionOverrides`: override validationFailureAction
* webhookTimeoutSeconds`: maximum time to apply the policy.

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

### Validate rules
For example, allowing to generate an alert for all Pods in the cluster that do not contain the `team` tag (with any value), and blocking the creation of Pods that do not have the `team` tag in the `req-labels` namespace.
```yaml
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
```

The patterns you can use are:
* '*'
* '?'
* '?*'
* 'null'

### Mutate Rules
For example, it allows to mutate all Pods in the `pull-policy` namespace that define an image `:latest` by setting an `imagePullPolicy` to `IfNotPresent`.
```yaml
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
```
Some mutations cannot be performed by patchStrategicMerge so you must use patchesJson6902.

```yaml
apiVersion: kyverno.io/v1
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
```

### Generate Rules
It is possible to create Kubernetes objects as a ConfigMap with some data in all namespaces except the excluded list.
```yaml
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
```

### Verify Images
It is possible to verify images with Notary and Sigstore.

```yaml
#Notary
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

# Signature Cosign
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: Enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: check-image
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - imageReferences:
        - "ghcr.io/kyverno/test-verify-image*"
        attestors:
        - count: 1
          entries:
          - keys:
              publicKeys: |-
                -----BEGIN PUBLIC KEY-----
                MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE8nXRh950IZbRj8Ra/N9sbqOPZrfM
                5/KAQN0/KjHcorm/J5yctVd7iEcnessRQjU917hmKO6JWVGHpDguIyakZA==
                -----END PUBLIC KEY-----     
```

### Cleanup
It is also possible to delete objects
```yaml
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
```

### Testing policies
It is possible to test policies in pipelines, before applying the resources to be created.

### Reporting
Kyverno provides reports on all policies.

```sh
$ kubectl get polr -A
NAMESPACE     NAME                                  PASS   FAIL   WARN   ERROR   SKIP   AGE
kube-system   cpol-disallow-privileged-containers   14     0      0      0       0      6s
kyverno       cpol-disallow-privileged-containers   2      0      0      0       0      5s
default       cpol-disallow-privileged-containers   0      1      0      0       0      5s
```
