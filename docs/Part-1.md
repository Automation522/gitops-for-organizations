# Provisioning Openshift clusters using GitOps with ACM

In the [introduction](../README.md), we described a solution where ACM is used to provision Openshift clusters using Gitops. The users fill in the clusters parameters in a form, which are written to a yaml/json object and pushed to git. ArgoCD synchronizes these objects into the ACM cluster. The cluster is provisioned with ACM, which automatically adds the clusters to Openshift GitOps for Day 2. And then, the cluster is configured automatically with ArgoCD using Helm + Kustomize, and ACM policies.

In this article, we'll explain in more detail the first part of the solution: *provisioning of Openshift clusters using GitOps with ACM*.

If you're intested in the second part of the solution, *Configuring Openshift cluster with ApplicationSets using Helm+Kustomize and ACM Policies*, don't miss the [next article](Part-2.md).

![Openshift Gitops Overview](../img/gitops-for-organization-overview.png)

## Frontend

We need a frontend (web application) to create the objects needed by ACM to provision the clusters. This frontend can be AAP (Tower), Jenkins, or any custom web application with a form. Although ACM can be used as frontend to provision the clusters, there are some drawbacks: ACM stores the objects locally, not in git; ACM has a fixed form for cluster creation, which cannot be customized.

![Openshift Gitops Overview](../img/gitops-for-organization-frontend.png)

This solution doesn't rely on any specific application/orchestrator. This solution just need that this application writes 2 configuration files: `conf.yaml` and `provision.yaml`.  

### Cluster Parameters

Instead of creating all the objects needed by ACM in the frontend, we just need to create a 2 objects:
- `conf.yaml`: contains all the configuration parameters of the cluster.
- `provision.yaml`: contains the parameters needed to provision the cluster, like `intall-config.yaml`.


In our example repository, we can see these files for the cluster example [small1.skyr.dca.scc](../clusters/dev/small1.skyr.dca.scc):

```
├── clusters
│   └── dev
│       └── small1.skyr.dca.scc
│           ├── conf.yaml
│           ├── provision.yaml
│           └── overlay
│               └── kustomization.yaml
└── conf
    └── dev
        ├── conf.yaml
        └── provision.yaml
```

There are no specif format for conf.yaml and provision.yaml, but we follow these guidelines:
* In the folder `conf/<environment>`, we sould add the common configuration for each environment.
* In the folder `clusters/<environment>/<cluster>`, we should add the specific configuration for the cluster. 
* In `conf.yaml`, we should add the configuration needed for Day 2, explained in the next part: [Configuring Openshift cluster with ApplicationSets using Helm+Kustomize and ACM Policies](Part-2.md).
* In `provision.yaml`, we should add the configuration needed for provisioning, for our Cluster-provisioning Helm chart. You can see an example of provision.yaml for the [cluster here](../clusters/dev/small1.skyr.dca.scc/provision.yaml), and for the [environment here](../conf/dev/provision.yaml).


## Cluster-provisioning ApplicationSet

We’re using an ApplicationSet to search for provision.yaml files, and it’ll create an Argo Application for each file. Each provision application will create all the ACM objects needed to deploy an Openshift cluster.

![Openshift Gitops Overview](../img/gitops-for-organization-provision-applicationset.png)

This application uses a Helm chart to deploy all the ACM objects needed:

- ClusterDeployment
- KlusterletAddonConfig
- MachinePool
- ManagedCluster
- Namespace
- Secrets: pull-secret, install-config, ssh-private-key and creds

The Helm [ACM provision chart](../base/provision/openshift-provisioning/templates) has these templates: 

```
base
└── provision
    ├── Chart.yml
    └── templates
        ├── ClusterDeployment.yml
        ├── KlusterletAddonConfig.yml
        ├── MachinePool.yml
        ├── ManagedCluster.yml
        ├── Namespace.yml
        └── Secrets.yml
```

The ApplicationSet creates an Argo Application for each `provision.yaml` under clusters folder using the path: `base/provision/openshift-provisioning` and the config files:

* `/conf/<cluster environment>/provision.yaml`
* `/clusters/<cluster environment>/<cluster fqdn>/provision.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
 name: cluster-provisioning
 namespace: openshift-gitops
spec:
 generators:
   - git:
       repoURL: https://github.com/Automation522/gitops-for-organizations.git
       revision: main
       files:
         - path: "clusters/**/provision.yaml"
 template:
   metadata:
     name: "provision-{{cluster.environment}}-{{cluster.name}}"
     labels:
       environment: "{{cluster.environment}}"
       cluster: "{{cluster.fqdn}}"
       region: "{{cluster.region}}"
       cloud: "{{cluster.cloud}}"
   spec:
     project: default
     source:
       repoURL: https://github.com/Automation522/gitops-for-organizations.git
       targetRevision: main
       path: base/provision/openshift-provisioning
       helm:
         valueFiles:
           - /conf/{{cluster.environment}}/provision.yaml
           - /clusters/{{cluster.environment}}/{{cluster.fqdn}}/provision.yaml
     destination:
       server: 'https://kubernetes.default.svc'
       namespace:
     syncPolicy:
       syncOptions:
         - preserveResourcesOnDeletion=true
```

### Use of preserveResourcesOnDeletion

Using an ApplicationSet to dynamically generate the provision applications has a risk involved: if the ApplicationSet is deleted. the following occurs (in rough order):

The ApplicationSet resource itself is deleted
Any Application resources that were created from this ApplicationSet (as identified by owner reference)
Any deployed resources (Deployments, Services, ConfigMaps, etc) on the managed cluster, that were created from that Application resource (by Argo CD), will be deleted.
  
To prevent the deletion of the resources of the Application, such as ManagedCluster, ClusterDeployment, etc, set .syncPolicy.preserveResourcesOnDeletion to true in the ApplicationSet. This syncPolicy parameter prevents the finalizer from being added to the Application.

## Cluster Provision

Once the ACM objects are synchronized to the cluster, ACM will start to provision the new cluster. 


## Day 2 Configuration

To add each new cluster to GitOps, we need to create this 3 objects in ACM:
- ManagedClusterSetBinding
- Placement
- GitOpsCluster

These objects will be also stored in git, as explained in the next part, [Configuring Openshift cluster with ApplicationSets using Helm+Kustomize and ACM Policies](Part-2.md):

```
└── clusters
    └── acm-hub
        ├── applications
        ....
        └── gitops-cluster
            ├── GitOpsCluster.yaml
            ├── ManagedClusterSetBinding.yaml
            ├── Placement.yaml
            └── kustomization.yaml
```

A ManagedClusterSetBinding resource allows us to bind a ManagedClusterSet resource to a namespace. In our example, we want to bind the clusterSet vmware to the openshift-gitops namespace:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
 name: vmware
 namespace: openshift-gitops
spec:
 clusterSet: vmware
```


The Placement allows us to select clusters based on labels, in the case, platform = vmware:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
 name: vmware
 namespace: openshift-gitops
spec:
 predicates:
 - requiredClusterSelector:
     labelSelector:
       matchExpressions:
       - key: platform
         operator: "In"
         values:
         - vmware
```



The GitOpsCluster custom resource allows us to register the clusters selected with a Placement to an ArgoCD server:

```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
 name: argo-acm-clusters
 namespace: openshift-gitops
spec:
 argoServer:
   cluster: local-cluster
   argoNamespace: openshift-gitops
 placementRef:
   kind: Placement
   apiVersion: cluster.open-cluster-management.io/v1beta1
   name: vmware
   namespace: openshift-gitops
```

<br />
<br />

>Continue to the second part: [Configuring Openshift cluster with ApplicationSets using Helm+Kustomize and ACM Policies](Part-2.md)

>Return to the Introduction: [GitOps for organizations: provisioning and configuring Openshift clusters automatically](../README.md)
