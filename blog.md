## ArgoCD and Red Hat Advanced Cluster Management-Policies: Better together

Sometimes we are getting asked about the relationship between Gitops-Operator/ArgoCD and Red Hat Advanced Cluster Management (RHACM)-Policies and if they can be used together.
In the following we will list the advantages of the integration showing some examples.

## Advantages of using RHACM with ArgoCD

* RHACM can be used to install/configure Gitops-Operator/ArgoCD consistently either on the Hub or on Managed-Clusters. See an example [here](https://github.com/stolostron/policy-collection/blob/main/community/CM-Configuration-Management/policy-openshift-gitops.yaml).
  Using the `App-of-Apps` pattern you can e.g. have a root Gitops-Operator/ArgoCD-Application which deploys other Applications from them one or more can have the purpose to deploy Policies. 
  Please review this [blog]((https://gexperts.com/wp/bootstrapping-openshift-gitops-with-rhacm/)) for a comprehensive example how to bootstrap an Environment using Policies.

* It offers you the option to enforce and monitor the settings of Gitops-Operator/ArgoCD regardless if you have a `centralized` or `decentralized` approach. This means you can consistently rollout 
  the configuration to your fleet of clusters avoiding any issues which come from `inconsistencies1 e.g. regarding RBAC and which are later difficult to troubleshoot.

* You get advanced templating features optimized for `Multi-Cluster-Management` which includes `Secrets-Management` where you can securely copy a secret from the Hub to a ManagedCluster
  like in below example.

```
object-templates:
  - complianceType: musthave
    objectDefinition:
      kind: Secret
      type: Opaque
      metadata:
        name: config-demo-secret
        namespace: config-demo
      apiVersion: v1
        data:
          secret: '{{ `{{hub fromSecret "config-demo" "config-demo-secret" "secret" hub}}` }}'
```

* There is the option to generate resources (e.g `Roles`, `Rolebindings`) in one or several namespaces based on namespace `names`, `labels` or `expressions`.

  In ACM 2.6 - as you see below - we enhanced our `namespaceSelector` to chose namespaces also by `label` and `expression`:

```
namespaceSelector:
  matchLabels:
    name: test2
  matchExpressions:
    key: name
    operator: In
    values: ["test1", "test2"]
```

* You have the capability to patch resources. This means if a Kubernetes-Object must contain certain values you specify `musthave` in case you can tolerate other fields.
  Else - if the object must match exactly - you must specify `mustonlyhave`.

* We provide the option to just monitor resources instead of creating/patching them (`inform`, versus `enforce`). It is possible to monitor the status of any Kubernetes-Object.
  In this case we check for namespaces in `terminating` status leading to a violation.

```
spec:
  remediationAction: inform
  severity: low
  object-templates:
  - complianceType: mustnothave
    objectDefinition:
      apiVersion: v1
      kind: Namespace
      status:
      phase: Terminating
```

* Similar to above we provide the option to delete certain objects, you would just set the `remediationAction` to `enforce`.

  Please check here for more [examples](https://github.com/stolostron/governance-policy-framework/blob/main/doc/configuration-policy/README.md#basic-usage) regarding the previous points.

  One of the most interesting usecases here is to delete the kubeadmin-secret from the managed-clusters.
  
  The capability to delete objects is enhanced by specifying a prune-behaviour so you can decide what should happen
  with the objects once you delete them. Please review here [Prune Object Behavior](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/governance/governance#cleaning-up-resources-from-policies) 

* RHACM's Governance framework provides the option to group objects to certain sets (PolicySets), a feature which has both UI and Gitops-Support
  - See how PolicySets can be configured using [PolicyGenerator:](https://github.com/stolostron/policy-collection/blob/main/policygenerator/policy-sets/community/openshift-plus/policyGenerator.yaml#L154)
  - See an example of a PolicySet being stored in git by checking:

```
apiVersion: policy.open-cluster-management.io/v1beta1
kind: PolicySet
  metadata:
    name: certificates-policyset
    namespace: cert-manager
  spec:
    description: "Grouping policies related to certificate handling"
  policies:
  - azure-clusterissuer-policy
  - cert-manager-csv-policy
  - certification-expiration-policy
```

  - See how they look in the UI:

* You have the possibility to configure how often checks should be evaluated considering the current status of an evaluated Object

```
spec:
  evaluationInterval:
  compliant: 10m
  noncompliant: 10s
```
  This example 

```
  spec.evaluationInterval.compliant: never
```
  stops evaluating the policy once it is in compliant state. So it enforces it only once.
  The above feature has mainly the advantage to tune environments with many policies to consume less resources.

* You can use PolicyGenerator (at Runtime) which also can be used for integration of Kyverno and Gatekeeper 

  In the following example we use a Policy for creating an `ArgoCD-Notification` config map using PolicyGenerator.
  As the config map already contains templating-expression we disable RHACM-Templating for this Policy in the PolicyGenerator  
  file. PolicyGenerator can be used in ArgoCD to transform yaml-resources to Policies at Runtime. The integration works via CustomTooling as you see [here]
  (https://argo-cd.readthedocs.io/en/stable/operator-manual/custom_tools/).

* Governance focused UI-support (Governance-Dashboard) which enables you to drill down into errors from every Single Policy

* Option to have less/or more fine grained checks by using Configuration-Policies

  This means you can create one ConfigurationPolicy for every single Kubernetes-Object or bundle many of them. Each 
  Configuration Policy will be one unit when it comes to check the status in the UI. 

* Monitoring- and Ansible-integration (gives you the option to implement Automated Governance)

  Those topics have already been explained in several blogs which can be found [here](https://github.com/stolostron/policy-collection/tree/main/blogs).

* Policies can be used to check for expired Certificates in different namespaces

```
apiVersion: policy.open-cluster-management.io/v1
kind: CertificatePolicy
metadata:
  name: certificate-policy-1
  namespace: kube-system
  label:
    category: "System-Integrity"
spec:
  # include are the namespaces you want to watch certificatepolicies in, while exclude are the namespaces you explicitly do not want to watch
  namespaceSelector:
    include: ["default", "kube-*"]
    exclude: ["kube-system"]
  # Can be enforce or inform, however enforce doesn't do anything with regards to this controller
  remediationAction: inform
  # minimum duration is the least amount of time the certificate is still valid before it is considered non-compliant
  minimumDuration: 100h
```

### Running the example

All you need to do is executing this [example](https://raw.githubusercontent.com/ch-stark/argocdpoliciesblog/main/setuppolicies/setuppolicies.yaml), execute it 2 times with some interval of ca 30 sec as GitopsOperator might need to be installed first.

ArgoCD in `policies namespace` is configured to setup PolicyGenerator

```
repo:
  resources:
    limits:
      cpu: '1'
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  env:
  - name: KUSTOMIZE_PLUGIN_HOME
    value: /etc/kustomize/plugin
  initContainers:
  - args:
    - cp /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator
      /policy-generator/PolicyGenerator
    command:
    - sh
    - -c
    image: registry.redhat.io/rhacm2/multicluster-operators-subscription-rhel8:v2.6.2
    name: policy-generator-install
    volumeMounts:
    - mountPath: /policy-generator
      name: policy-generator
  volumeMounts:
  - mountPath: /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator
    name: policy-generator
  volumes:
  - emptyDir: {}
    name: policy-generator
kustomizeBuildOptions: --enable-alpha-plugins
```

We deploy three Applications with only slighty different purpososes:

- `Application 1` deploys a `stable` (means supported)-PolicySet in order to harden RHACM. PolicyGenerator is used.
- `Application 2` deploys a custom PolicySet e.g. for Configuration-Purposes. It contains one policy and is designed to be   
  extended. PolicyGenerator is used also here.
- With `Application 3` we are deploying supported Policies from our `PolicyCollection` repository. As we need to apply them
  from `subdirectories` the following property `recurse: true` has to be present (unlike in the first two Applications).

```
source:
  path: stable
  repoURL: https://github.com/stolostron/policy-collection
  targetRevision: HEAD
  directory:
    recurse: true # <--- Here
```

This short overview had the purpose to explain why it is a good idea to use policies together with GitOpsOperator/ArgoCD. Both approaches can benefit from each other. You get all the benefits highlighted above out of the box.
