## ArgoCD and Red Hat Advanced Cluster Management-Policies: Better together

In this blog, we are going to explore the relationship between OpenShift GitOps (Argo CD) and Red Hat Advanced Cluster Management's (RHACM's) policy framework and how they fit together.
While RHACM is RedHat’s solution for Kubernetes `MultiClusterManagement` with a strong focus on governance, OpenShift GitOps (Argo CD) is a very popular Gitops-Engine which is used by many customers and which just reached CNCF's [graduated](https://www.cncf.io/announcements/2022/12/06/the-cloud-native-computing-foundation-announces-argo-has-graduated/ ) status. 
In the following we will list the advantages of deploying RHACM-Policies using Argo CD showing concrete examples.


## Advantages of using Policies with ArgoCD

1. RHACM-Policies can be used to install and configure Argo CD consistently either on the Managing or on Managed-Clusters. See    an example [here](https://github.com/stolostron/policy-collection/blob/main/community/CM-Configuration-Management/policy-openshift-gitops.yaml).
   The Policy is marked as part of the `Baseline-Configuration` of the `NIST SP 800-53` standard and could certainly be part of    any other standard you like to implement.

   ```
   kind: Policy
   metadata:
     name: openshift-gitops-installed
     annotations:
       policy.open-cluster-management.io/standards: NIST SP 800-53
       policy.open-cluster-management.io/categories: CM Configuration Management
       policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
   ```

   as an example you could consistently overwrite the default Gitops-Instance in all your namespaces and environments.

   ```
   spec:
     channel: stable
     config:
       env:
       - name: DISABLE_DEFAULT_ARGOCD_INSTANCE
         value: "true"
       - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
         value: <namespace to deploy to>
   ```

   another example would be to configure [SSO](https://docs.openshift.com/container-platform/4.11/cicd/gitops/configuring-sso-for-argo-cd-using-keycloak.html) for your `fleet` of Gitops-Instances.

   ```
   spec:
     sso:
       provider: keycloak
   ```
   It offers you the option to `enforce` and `monitor` the settings of `Gitops-Operator/ArgoCD` regardless if you have a  
  `centralized` or `decentralized` approach. This means you can consistently rollout 
   the configuration to your fleet of clusters avoiding any issues which might come from `inconsistencies` e.g. regarding RBAC    and which are later difficult to troubleshoot.

2. Using the `App-of-Apps` pattern you can e.g. have a root `Gitops-Operator/ArgoCD-Application` which deploys other
   Applications. One of those child-apps could have the purpose to deploy Policies to easily integrate the compliance aspect. 
   Please review this [blog](https://gexperts.com/wp/bootstrapping-openshift-gitops-with-rhacm/) for a comprehensive example
   how to bootstrap an environment using Policies using this pattern.

3. You get `advanced templating features` optimized for `Multi-Cluster-Management` which includes `Secrets-Management` where 
   you can securely copy a secret from the Hub to a ManagedCluster like in the example below:

   ```
   objectDefinition:
     apiVersion: v1
     data:
       city: '{{hub fromSecret "" "hub-secret" "city" hub}}'
       state: '{{hub fromSecret "" "hub-secret" "state" hub}}'
     kind: Secret
     metadata:
       name: copied-secret
       namespace: target
   ```

   `Benefits` of this approach are among others that there is no `duplication` of policies (and thus easier maintenance) as you     customize specific elements of a policy over various clusters within the fleet.

    See here a `real world` example how to create an ApplicationSet using a `TemplatizedPolicies` approach:

   ```
    apiVersion: policy.open-cluster-management.io/v1
    kind: Policy
    metadata:
      name: policy-as-bootstrap-gitops
      annotations:
        policy.open-cluster-management.io/standards: NIST SP 800-53
        policy.open-cluster-management.io/categories: CM Configuration Management
        policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    spec:
      remediationAction: enforce
      disabled: false
      policy-templates:
        - objectDefinition:
            apiVersion: policy.open-cluster-management.io/v1
            kind: ConfigurationPolicy
            metadata:
              name: gitops-argocd-applicationset
            spec:
              remediationAction: inform  # will be overridden by remediationAction in parent policy
              severity: high
              object-templates:
                - complianceType: mustonlyhave
                  objectDefinition:
                    apiVersion: argoproj.io/v1alpha1
                    kind: ApplicationSet
                    metadata:
                      name: as-bootstrap-acm
                      namespace: openshift-gitops
                    spec:
                      generators:
                      - git:
                          repoURL: https://gitlab.example.com/openshift/argocd-cluster-config.git
                          revision: main
                          files:
                          - path: 'cluster-definitions/{{ fromClusterClaim "name" }}/cluster.json'
                      template:
                        metadata:
                          name: 'ocp-{{ fromClusterClaim "name" }}-bootstrap-acm'
                        spec:
                          project: default
                          source:
                            repoURL: https://gitlab.example.com/openshift/argocd-cluster-config.git
                            targetRevision: main
                            path: "cluster-config/overlays/{{`{{cluster.name}}`}}"
                          destination:
                            server: https://kubernetes.default.svc
                          syncPolicy:
                            automated:
                              prune: true
                              selfHeal: true
    ```

4.  RHACM Policy framework provides the option to generate resources (e.g `Roles`, `Rolebindings`) in one or several namespaces
    based on namespace `names`, `labels` or `expressions`.

    In RHACM version 2.6 - as you see below - we enhanced our `namespaceSelector` to chose namespaces also by `label` and          `expression` which gives you more flexibility on which namespaces you like to operate on:

    ```
       namespaceSelector:
         matchLabels:
           name: test2
         matchExpressions:
           key: name
           operator: In
           values: ["test1", "test2"]
    ```
    Benefit of this feature is that it reduces significantly the Policies you need to configure, e.g. when you want to copy a
   `ConfigMap` into several namespace.


5.  You have the capability to patch resources. This means if a Kubernetes-Object `must` contain certain values you specify        `musthave` in case you can tolerate other fields.
    Else - if the object must match exactly - you must specify `mustonlyhave`.

    A often requested example is to disable the `self-provisioner role` from an existing or newly created OpenShift-Cluster:
   
    ```
        metadata:
          name: policy-remove-self-provisioner
        spec:
          remediationAction: enforce
          severity: high
          object-templates:
            - complianceType: mustonlyhave
              objectDefinition:
                kind: ClusterRoleBinding
                apiVersion: rbac.authorization.k8s.io/v1
                metadata:
                  name: self-provisioners
                  annotations:
                    rbac.authorization.kubernetes.io/autoupdate: 'false'
                subjects: []
                roleRef:
                  apiGroup: rbac.authorization.k8s.io
                  kind: ClusterRole
                  name: self-provisioner
    ```

6.  We provide the option to just monitor resources instead of creating/patching them (can be configured via `inform`, versus
    `enforce`). It is possible to monitor the status of `any` Kubernetes-Object.
    In the following case we check for namespaces in `terminating` status.

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

   In this example none of the 4 evaluated Clusters has such a violation.

   ![Check Terminating](images/policy-terminating.png)

7. Similar to above we provide the option to `delete` certain objects, you would just set the `remediationAction` to `enforce`.

   Please check here for more [examples](https://github.com/stolostron/governance-policy-framework/blob/main/doc/configuration-policy/README.md#basic-usage) regarding the previous points.

   Please note that one of the most interesting usecases here is to delete the `kubeadmin-secret` from the managed-clusters.
  
   The capability to delete objects is enhanced by specifying a `prune-behaviour` so you can decide what should happen
   with the objects once you delete a Policy. Please review here [Prune Object Behavior](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html/governance/governance#cleaning-up-resources-from-policies). 

8. RHACM's Governance framework provides the option to group objects to certain sets (PolicySets), a feature which has both UI 
   and Gitops-Support.
   - See how PolicySets can be configured using [PolicyGenerator](https://github.com/stolostron/policy-collection/blob/main/policygenerator/policy-sets/community/openshift-plus/policyGenerator.yaml#L154)
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

   - See how they look in the UI (examples taken from our [Kyverno-Policysets](https://github.com/stolostron/policy-collection/tree/main/policygenerator/policy-sets/community/kyverno)):

![Kyverno-Policysets](images/policysets.png)

   - See how the [OpenShift-Hardening-Policyset](https://github.com/stolostron/policy-collection/tree/main/policygenerator/policy-sets/community/openshift-hardening) looks like. You see some policies are   
     compliant, some others are not and need investigation.

![OpenShift-Hardening](images/openshifthardening.png)


9. You have the possibility to configure how often checks should be evaluated considering the current status of an evaluated  
   Object.

   ```
   spec:
     evaluationInterval:
     compliant: 10m
     noncompliant: 10s
   ```
   or for one time jobs
   ```
   spec.evaluationInterval.compliant: never
   ```

   The above example stops evaluating the policy once it is in compliant state. So it enforces it only once.
   The feature has mainly the advantage to tune environments with many policies to consume less resources.

10. You can use `PolicyGenerator` which also can be used for integration of `Kyverno` and `Gatekeeper`. 

  `PolicyGenerator` can be used in ArgoCD to transform `yaml-resources` to Policies at Runtime. The integration works via  
   Custom Tooling as you see [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/custom_tools/).

   Let's take the following example. You want to deploy the following resources together:
  
   - `Deployment`: define which image to run.
   - `Service`: component can be reached over the network.
   - `Ingress`: the outside world can access our Service.
   - `ConfigMap`: configure the component (often makes sense to make this templatized).
   - `Secret`: supply credentials to the component (often makes sense to make this templatized).
   - `NetworkPolicy`: restrict the component's attack surface.

   So you could place all `yaml-files` into a folder, together with the `checks` like `deployment must be running` and you can
   create a single Policy by just configure:

   ```
      policies:
      # Deployment-Policy - start
      - name: policy-deployment
        categories:
          - System-Configuration
        controls: 
          - ApplicatonDeployment
        manifests:
          - path: input/
   ```

   In the above all yaml-files would be used to generate a single Policy. 
   Please note that you can also generate Policies from folders which contain a `Kustomization`.
   Let's check the following example-extract:

   ```
      policies:
        - name: policy-gatekeeperlibrary
          manifests:
            - path: gatekeeperlibrary
   ```
   points to

   ```
      apiVersion: kustomize.config.k8s.io/v1beta1
      kind: Kustomization
      resources:
        - https://github.com/open-policy-agent/gatekeeper-library/library     
     
   ```

   ### Migrate from PlacementRules to Placement using PolicyGenerator
      
   Another nice feature of `PolicyGenerator` is that it helps you to upgrade from `PlacementRules` to the new `Placement-API`.
   You see in above file that there is both the option to set a `PlacementRule` or a `Placement`. You can either specify a name
   (when the object already exists in the Cluster) or a path in the Gitrepo to apply the objects. See:  
   placementPath,placementName or placementRulePath and placementRuleName in the reference file [here](https://github.com/stolostron/policy-generator-plugin/blob/main/docs/policygenerator-reference.yaml).


11. Governance focused UI-support (Governance-Dashboard) which enables you to drill down into errors from every single Policy.

    See here for an overall Governance overview in RHACM-UI
   ![Governance](images/governance.png)
    

    See here example of our `build-in policy` which checks if backup is setup correctly:
   ![Backup Policy](images/backuprestore.png)


12. Option to have less/or more fine grained checks by using Configuration-Policies

    This means you can create one `Configuration Policy` for every single Kubernetes-Object or bundle many of them. Each 
   `Configuration Policy` will be one unit when it comes to check the status in the UI as you see in the screenshot above.
    Benefit is that it gives you more flexibility. 

13. Monitoring- and Ansible-integration (gives you the option to implement `Automated Governance`)

    Those topics have already been explained in several blogs which can be found [here](https://github.com/stolostron/policy-collection/tree/main/blogs) just to summarize:
  
    - you can give Policies a `severity` sending alerts via `RHACM's Obvervability` framework only from high-prio policies
    - you can invoke Ansible-Jobs from Policies so you could only invoke Ansible from critical policies
    - you could use the previously discussed `evaluationInterval` to trigger an Ansible Job at a regular basis.
   
14. Policies can be used to check for `expired` Certificates in different namespaces.
    Please review the following example where a violation will be created if a certificate is valid for less than 100 hours:  

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
   
   For running the example all you need to do is executing this [example](https://raw.githubusercontent.com/ch-stark/argocdpoliciesblog/main/setuppolicies/setuppolicies.yaml), execute it 2 times with some interval of ca 30 sec as  
   GitopsOperator might need to be installed first.

   The ArgoCD instance will be deployed in `policies namespace` and is configured to setup PolicyGenerator.

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

  We deploy several Applications with only slighty different purposes:
  e.g. 
   - `Application 1` deploys a `stable` (means supported)-PolicySet in order to harden RHACM. PolicyGenerator is used.
   - `Application 2` deploys a custom PolicySet e.g. for Configuration-Purposes. It contains one policy and is designed to be   
      extended. PolicyGenerator is used also here.
   -  With `Application 3` we are deploying supported Policies from our `PolicyCollection` repository. As we need to apply them
      from `subdirectories` the following property `recurse: true` has to be present (unlike in the first two Applications).

   ```
      source:
        path: stable
        repoURL: https://github.com/stolostron/policy-collection
        targetRevision: HEAD
        directory:
          recurse: true # <--- Here
   ```

   See some of the Policies being synced onto the Hub-Cluster using ArgoCD-Applications in the `Governance-View`.
   ![Governance-View](images/policies_from_argo.png)

## Fixing the issues that ArgoCD gets out of sync and monitoring the solution

   A nice example how ArgoCD can be configured to optimize the interaction with RHACM-policies has been the following. It   
   turned out that as the RHACM-Policy-Controller is copying Policies into a namespace (representing a managed cluster)
   therefore `ArgoCD-Applications` became `out-of-sync`. This can be fixed by setting the `resource tracking` method to
   [annotation](https://argocd-operator.readthedocs.io/en/latest/reference/argocd/#resource-tracking-method) which is already
   included in the examples.

   Again you can benefit from RHACM's `Gatekeeper Integration` by using this `Gatekeeper-Constraint` together with 
   PolicyGenerator.

   ```
   ---
   # Require applications deploy with ArgoProj, ex demo-accept-app:/Service:secure/summer-k8s-service-canary
   # To use this, set Argo CD tracking method to annotation or annotation+label. https://argo-cd.readthedocs.io/en/stable/user-      guide/resource_tracking/
   apiVersion: constraints.gatekeeper.sh/v1beta1
   kind: K8sRequiredAnnotations
   metadata:
     name: must-be-deployed-by-argocd
   spec:
     enforcementAction: deny
     match:
       namespaces:
         - "*"
       kinds:
         - apiGroups: [""]
           kinds: ["ReplicaSet", "Deployment"]
     parameters:
       message: "All resources must have a `argocd.argoproj.io/tracking-id` annotation."
       annotations:
         - key: argocd.argoproj.io/tracking-id
           # Matches annotation format
           allowedRegex: ^[a-zA-Z0-9-]+:.+$
   ```

### Summary

   This overview had the purpose to explain why it is a good idea to use Policies together with `GitOpsOperator/ArgoCD`. Both      approaches can heavily benefit from each other. Certainly it needs to be highlighted that the focus of RHACM-Policies is to 
   support customers becoming `compliant` from a technical point of view. It can be even seen as a `ArgoCD extension to
   Governance`.  You get all the great benefits highlighted above out of the box.
   Both `sites` will provide further features in the future like enhanced `Operator` or `Dependency` Management and we think of
   further integratin possibilities. We would love to get your feedback and thoughts on this topic.


