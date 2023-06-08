---
title: Helm
date: 20220220
author: Adrián Martín García
---

# Helm
Helm helps you manage Kubernetes applications. 

A Kubernetes manifest describes the resources (e.g., Deployments, Services, Pods, etc.) you want to create, and how you want those resources to run inside a cluster.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: my-namespace
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## Helm Charts
Helm uses a packaging format called Charts. A Helm Chart is a collection of files that describe a set of Kubernetes resources.

### Directory hierarchy 
```
.
└── my-release
    ├── charts
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── NOTES.txt
    │   ├── serviceaccount.yaml
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

## Helm-docs
The helm-docs tool auto-generates documentation from helm charts into markdown files. The resulting files contain metadata about their respective chart and a table with each of the chart's values, their defaults, and an optional description parsed from comments.

*  https://github.com/norwoodj/helm-docs
