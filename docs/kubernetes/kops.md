# kOps

## Example file
```yaml
# kOps Cluster definition
apiVersion: kops.k8s.io/v1alpha2
kind: Cluster
metadata:
  ## Cluster name
  name: cluster.example
spec:
  api:
    loadBalancer:
      class: Network
      type: Public
  assets:
    containerProxy: registry.com:9090
  authorization:
    rbac: {}
  channel: stable
  cloudProvider: aws
  ## Bucket name and state file
  configBase: s3://bucket-name/cluster.example
  dnsZone: zone.example
  etcdClusters:
  - name: main
    cpuRequest: 200m
    etcdMembers:
    - instanceGroup: master-1
      name: etcd-1
    - instanceGroup: master-2
      name: etcd-2
    - instanceGroup: master-3
      name: etcd-3
    memoryRequest: 100Mi
  - name: events
    cpuRequest: 100m
    etcdMembers:
    - instanceGroup: master-1
      name: etcd-1
    - instanceGroup: master-2
      name: etcd-2
    - instanceGroup: master-3
      name: etcd-3
    memoryRequest: 100Mi
  iam:
    legacy: false
  ## Kubelet config
  kubelet:
    ### Enable to use metrics-server
    anonymousAuth: false
    ### Allow serviceaccount tokens to communicate with kubelet
    authorizationMode: Webhook
    authenticationTokenWebhook: true
  ## VPC CIDR access to Kubernetes API Server
  kubernetesApiAccess: []
  kubernetesVersion: 1.22.6
  masterPublicName: api.cluster.example
  ## VPC CIDR
  networkCIDR: 10.3.0.0/16
  ## VPC ID
  networkID: vpc-00000000000
  networking:
    ## CNI
    calico:
      majorVersion: v3
      ### maximum transmission unit
      mtu: 1500
  ## Internal kubernetes CIDR
  nonMasqueradeCIDR: 110.64.0.0/10
  sshAccess: null
  subnets:
  - cidr: 12.0.0.0/24
    id: subnet-00000000000
    name: utility-eu-west-1c
    type: Utility
    zone: eu-west-1c
  - cidr: 10.3.2.0/23
    id: subnet-1111111111
    name: eu-west-1c
    type: Private
    zone: eu-west-1c
  topology:
    dns:
      type: Private
    masters: private
    nodes: private
  ## Addons
  awsLoadBalancerController:
    enabled: false
  clusterAutoscaler:
    enabled: true
    awsUseStaticInstanceList: false
    balanceSimilarNodeGroups: false
    expander: random
    image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.3
    maxNodeProvisionTime: 15m0s
    newPodScaleUpDelay: 0s
    scaleDownDelayAfterAdd: 10m0s
    scaleDownUtilizationThreshold: "0.5"
    skipNodesWithLocalStorage: true
    skipNodesWithSystemPods: true
  cloudConfig:
    awsEBSCSIDriver:
      enabled: false
      managed: false
  nodeProblemDetector:
    enabled: true
    memoryRequest: 32Mi
    cpuRequest: 10m

---

# IG master-1
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: cluster.example
  name: master-1
spec:
  additionalSecurityGroups:
  ## Manage by Terraform
  - sg-00000000000
  autoscale: false
  iam:
    ## Manage by Terraform
    profile: arn:aws:iam::00000000000:instance-profile/MASTERS
  associatePublicIp: false
  rootVolumeSize: 30
  rootVolumeType: gp3
  ## AMI id for region
  image: ami-02b4e72b17337d6c1
  ## master set max = 1 min = 1
  minSize: 1
  maxSize: 1
  machineType: c5a.large
  ## Labels for Kubernetes
  nodeLabels:
    kops.k8s.io/instancegroup: master-1
    project: master
  ## Labels for AWS
  cloudLabels:
    environment: Dev
  role: Master
  subnets:
  - eu-west-1c

---

# IG master-2
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: cluster.example
  name: master-2
spec:
  additionalSecurityGroups:
  ## Manage by Terraform
  - sg-00000000000
  autoscale: false
  iam:
    ## Manage by Terraform
    profile: arn:aws:iam::00000000000:instance-profile/MASTERS
  associatePublicIp: false
  rootVolumeSize: 30
  rootVolumeType: gp3
  ## AMI id for region
  image: ami-02b4e72b17337d6c1
  ## master set max = 1 min = 1
  minSize: 1
  maxSize: 1
  machineType: c5a.large
  ## Labels for Kubernetes
  nodeLabels:
    kops.k8s.io/instancegroup: master-2
    project: master
  ## Labels for AWS
  cloudLabels:
    environment: Dev
  role: Master
  subnets:
  - eu-west-1c

---

# IG master-3
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: cluster.example
  name: master-3
spec:
  additionalSecurityGroups:
  ## Manage by Terraform
  - sg-00000000000
  autoscale: false
  iam:
    ## Manage by Terraform
    profile: arn:aws:iam::00000000000:instance-profile/MASTERS
  associatePublicIp: false
  rootVolumeSize: 30
  rootVolumeType: gp3
  ## AMI id for region
  image: ami-02b4e72b17337d6c1
  ## master set max = 1 min = 1
  minSize: 1
  maxSize: 1
  machineType: c5a.large
  ## Labels for Kubernetes
  nodeLabels:
    kops.k8s.io/instancegroup: master-3
    project: master
  ## Labels for AWS
  cloudLabels:
    environment: Dev
  role: Master
  subnets:
  - eu-west-1c

---

# IG nodes
apiVersion: kops.k8s.io/v1alpha2
kind: InstanceGroup
metadata:
  labels:
    kops.k8s.io/cluster: cluster.example
  name: worker
spec:
  additionalSecurityGroups:
  ## Manage by Terraform
  - sg-00000000000
  iam:
    ## Manage by Terraform
    profile: arn:aws:iam::00000000000:instance-profile/MASTERS
  associatePublicIp: false
  rootVolumeSize: 30
  rootVolumeType: gp3
  ## AMI id for region
  image: ami-02b4e72b17337d6c1
  minSize: 5
  maxSize: 50
  machineType: r5a.xlarge
  onDemandBase: 10
  onDemandAboveBase: 0
  ## Rolling upgrade
  rollingUpdate:
    drainAndTerminate: true
    maxSurge: 3
    maxUnavailable: 2
  ## Labels for Kubernetes
  nodeLabels:
    kops.k8s.io/instancegroup: nodes
    app: nodes
    ### Label to exclude from load balancers
    node.kubernetes.io/exclude-from-external-load-balancers: "true"
  ## Labels for AWS
  cloudLabels:
    k8s.io/cluster-autoscaler/enabled: ""
    k8s.io/cluster-autoscaler/cluster.example: ""
    k8s.io/cluster-autoscaler/node-template/label: ""
    environment: Dev
  role: Node
  subnets:
  - eu-west-1c
```
