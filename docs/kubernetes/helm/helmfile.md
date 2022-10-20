---
title: Helm
date: 20220220
author: Adrián Martín García
---

# Helmfile
Helmfile is a Helm wrapper that allows to manage multiple Helm Charts in a simpler way.

## Requirements
* Helmfile
* Helm
* Helm-diff
* Helm-secrets
* SOPS  By default `export HELM_SECRETS_BACKEND=sops`.
* VALS `export HELM_SECRETS_BACKEND=vals`.

## Hierarchy
```
.
├── bases
│   ├── environments.yaml ------------------------------------------>   (Environments are defined. And versions are declared for the official Helm Charts)
│   └── helmDefaults.yaml ------------------------------------------>   (Default values of the Helm deployment. For example: wait or timeout)
├── helmfile.yaml -------------------------------------------------->   (Main helmfile.yaml file)
├── releases ------------------------------------------------------->   (Directory to locate all releases)
│   └── my-release
│       ├── defaults.yaml ------------------------------------------>   (Set default values for the release)
│       ├── envs --------------------------------------------------->   (Environment directory)
│       │   ├── dev
│       │   │   ├── aws-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from AWS)
│       │   │   ├── az-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Azure)
│       │   │   ├── gcp-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from GCP)
│       │   │   ├── hc-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Hashicorp Vault)
│       │   │   ├── az-local-secret-values.yaml -------------------->   (Optional. File with values encrypted locally with SOPS)
│       │   │   ├── .sops.yaml -> ../../../../templates/.sops.yaml ->   (Symbolic link to .sops.yaml file)
│       │   │   ├── values.yaml.gotmpl ----------------------------->   (Environment value using gotmpl)
│       │   │   └── values.yaml ------------------------------------>   (environment value file)
│       │   ├── pre
│       │   │   ├── aws-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from AWS)
│       │   │   ├── az-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Azure)
│       │   │   ├── gcp-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from GCP)
│       │   │   ├── hc-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Hashicorp Vault)
│       │   │   ├── az-local-secret-values.yaml -------------------->   (Optional. File with values encrypted locally with SOPS)
│       │   │   ├── .sops.yaml -> ../../../../templates/.sops.yaml ->   (Symbolic link to .sops.yaml file)
│       │   │   ├── values.yaml.gotmpl ----------------------------->   (Environment value using gotmpl)
│       │   │   └── values.yaml ------------------------------------>   (environment value file)
│       │   └── pro
│       │       ├── aws-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from AWS)
│       │       ├── az-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Azure)
│       │       ├── gcp-remote-secret-values.yaml ------------------>   (Optional. File with reference to remote encrypted values from GCP)
│       │       ├── hc-remote-secret-values.yaml ------------------->   (Optional. File with reference to remote encrypted values from Hashicorp Vault)
│       │       ├── az-local-secret-values.yaml -------------------->   (Optional. File with values encrypted locally with SOPS)
│       │       ├── .sops.yaml -> ../../../../templates/.sops.yaml ->   (Symbolic link to .sops.yaml file)
│       │       ├── values.yaml.gotmpl ----------------------------->   (Environment value using gotmpl)
│       │       └── values.yaml ------------------------------------>   (Environment value file)
│       └── helmfile.yaml ------------------------------------------>   (Helmfile.yaml release file)
└── templates ------------------------------------------------------>   (Template directory)
    ├── default_values.yaml ---------------------------------------->   (File with declared YAML Anchors that allow parameterizing the access to the value files)
    └── .sops.yaml ------------------------------------------------->   (File with SOPS encryption rules)
```


## Example files
### bases/environments.yaml
```yaml
---
templates:
  default_releases: &default_releases
    values:
    - metrics-server: 3.8.2
    - reloader: 0.0.118

environments:
  dev:
    values:
    <<: *default_releases
  pre:
    values:
    <<: *default_releases
  pro:
    values:
    <<: *default_releases
```

### bases/helmDefaults.yaml
```yaml
---
# If set to "Error", return an error when a subhelmfile points to a
# non-existent path. The default behavior is to print a warning and continue.
missingFileHandler: Error

# these labels will be applied to all releases in a Helmfile. Useful in templating if you have a helmfile per environment or customer and don't want to copy the same label to each release
commonLabels:
  provisioning: helmfile

# Default values to set for args along with dedicated keys that can be set by contributors, cli args take precedence over these.
# In other words, unset values results in no flags passed to helm.
# See the helm usage (helm SUBCOMMAND -h) for more info on default values when those flags aren't provided.
helmDefaults:
  # wait for k8s resources via --wait. (default false)
  wait: false
  # if set and --wait enabled, will wait until all Jobs have been completed before marking the release as successful. It will wait for as long as --timeout (default false, Implemented in Helm3.5)
  waitForJobs: false
  # time in seconds to wait for any individual Kubernetes operation (like Jobs for hooks, and waits on pod/pvc/svc/deployment readiness) (default 300)
  timeout: 600
  # verify the chart before upgrading (only works with packaged charts not directories) (default false)
  verify: false
  # forces resource update through delete/recreate if needed (default false)
  force: false
  ## enable TLS for request to Tiller (default false)
  #tls: true
  ## path to TLS CA certificate file (default "$HELM_HOME/ca.pem")
  #tlsCACert: "path/to/ca.pem"
  ## path to TLS certificate file (default "$HELM_HOME/cert.pem")
  #tlsCert: "path/to/cert.pem"
  ## path to TLS key file (default "$HELM_HOME/key.pem")
  #tlsKey: "path/to/key.pem"
  # limit the maximum number of revisions saved per release. Use 0 for no limit. (default 10)
  historyMax: 10
  # when using helm 3.2+, automatically create release namespaces if they do not exist (default true)
  createNamespace: true
  # if used with charts museum allows to pull unstable charts for deployment, for example: if 1.2.3 and 1.2.4-dev versions exist and set to true, 1.2.4-dev will be pulled (default false)
  devel: false
  # When set to `true`, skips running `helm dep up` and `helm dep build` on this release's chart.
  # Useful when the chart is broken, like seen in https://github.com/roboll/helmfile/issues/1547
  #skipDeps: false
```

### templates/.sops.yaml
```yaml
creation_rules:
    # PGP
    - path_regex: pgp-local.*.yaml$
      pgp: B911C4BA2C10BF8BA1D9D005A680D32C9AE9B9CB
    # HC Vault
    - path_regex: hc-local-.*.yaml$
      vault_path: "sops/"
      vault_kv_mount_name: "secret/"
      vault_kv_version: 2
    # GCP KMS
    - path_regex: gcp-local-.*.yaml$
      gcp_kms: projects/mygcproject/locations/global/keyRings/mykeyring/cryptoKeys/thekey 
    # AWS KMS
    - path_regex: aws-local-.*.yaml$
      kms: 'arn:aws:kms:us-west-2:0000000000000:key/fe86dd69-4132-404c-ab86-4269956b4500'
    # AZ Key Vault
    - path_regex: az-local-.*.yaml$
      azure_keyvault: https://test.vault.azure.net/keys/sops/7b7c6b92999b42e79d744a0d4dc23e4adf

```

### templates/default_values.yaml
```yaml
---
templates:
  # Labels
  any_enc_labels: &any_enc_labels
    labels:
      enc: any

  sops_enc_labels: &sops_enc_labels
    labels:
      enc: sops

  vals_enc_labels: &vals_enc_labels
    labels:
      enc: vals

  # Values files
  default_values: &default_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
  
  gotmpl_values: &gotmpl_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml.gotmpl
  
  remote_values: &remote_values
    values:
    - defaults.yaml
    - git::https://user:{{ env "CI_JOB_TOKEN" }}@git.company.com/demo/helmfiles/{{ .Environment.Name }}/values.yaml?ref=master
    - http://$HOSTNAME/artifactory/example-repo-local/test.tgz@values.yaml
  
  # Secrets on values files
  ## Local `SOPS`
  pgp_local_secret_values: &pgp_local_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/pgp-local-secret-values.yaml
  
  hc_local_secret_values: &hc_local_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/hc-local-secret-values.yaml
  
  gcp_local_secret_values: &gcp_local_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/gcp-local-secret-values.yaml
  
  aws_local_secret_values: &aws_local_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/aws-local-secret-values.yaml

  az_local_secret_values: &az_local_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/az-local-secret-values.yaml
  
  ## Remote `VALS`
  hc_remote_secret_values: &hc_remote_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/hc-remote-secret-values.yaml
  
  gcp_remote_secret_values: &gcp_remote_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/gcp-remote-secret-values.yaml
  
  aws_remote_secret_values: &aws_remote_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/aws-remote-secret-values.yaml

  az_remote_secret_values: &az_remote_secret_values
    values:
    - defaults.yaml
    - envs/{{ .Environment.Name }}/values.yaml
    secrets:
    - envs/{{ .Environment.Name }}/az-remote-secret-values.yaml
```

### helmfile.yaml
```yaml
---
bases:
- "bases/helmDefaults.yaml"
- "bases/environments.yaml"

repositories:
- name: metrics-server
  url: https://kubernetes-sigs.github.io/metrics-server/
- name: reloader
  url: https://stakater.github.io/stakater-charts

helmfiles:
- "releases/*/helmfile.yaml"
```

### releases/metrics-server/helmfile.yaml
```yaml
----
bases:
  - "../../bases/helmDefaults.yaml"
  - "../../bases/environments.yaml"

---
{{ readFile "../../templates/default_values.yaml" }}

releases:
# https://github.com/helm/charts/blob/master/stable/metrics-server/values.yaml
- name: metrics-server
  chart: metrics-server/metrics-server
  namespace: metrics-server
  version: {{ index (.Values | get "metrics-server" "3.7.0") }}
  <<: *any_enc_labels                                                          # You can use: any_enc_labels, sops_enc_labels or vals_enc_labels
  <<: *gotmpl_values                                                           # Depending on the type of values you need, 
                                                                               # you can use the anchors declared in the defaults_values.yaml file.
                                                                               # Example: default_values, remote_values, gcp_remote_secret_values, gcp_local_secret_values ...
```

### releases/metrics-server/defaults.yaml
```yaml
# Custom values for metrics-server.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
---
args:
# enable this if you have self-signed certificates, see: https://github.com/kubernetes-incubator/metrics-server
  - --kubelet-insecure-tls
```

### releases/metrics-server/envs/dev/values.yaml
```yaml
# Custom values to dev environment.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.
---
podLabels:
  env: ""
```

### releases/metrics-server/envs/dev/values.yaml.gtmpl
```go
{{ readFile "values.yaml" | fromYaml | setValueAtPath "podLabels.env" .Environment.Name | toYaml }}
```

## Hemfile Commands

### Rendering dev environment
```sh
$ helmfile template --environment dev
```
### Rendering dev environment for metrics-server
```sh
$ helmfile template --environment dev --selector name=metrics-server
```
### Diff. Shows differences between the code and the cluster
```sh
$ helmfile diff --environment dev --selector name=metrics-server
```
#### Linters
```sh
$ helmfile lint --environment dev
```
### Sync (Install) metrics-server release
```sh
$ helmfile sync --environment dev --selector name=metrics-server
```
### Delete metrics-server release
```sh
$ helmfile destroy --environment dev --selector name=metrics-server
```
