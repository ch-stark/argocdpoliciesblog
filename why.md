ArgoCD

* Cluster and application configuration versioned in Git
* Automatically syncs configuration from Git to clusters
* Drift detection, visualization and correction
* Granular control over sync order for complex rollouts
* Rollback and rollforward to any Git commit
* Manifest templating support (Helm, Kustomize, etc)
* Visual insight into sync status and history




* by deploying Policies (together with its Placementinfo) and using a simple ArgoCD-App (deployed on the Hub) you can bootstrap and consistently configure a whole fleet of Clusters
advanced Templating features optimized for Multi-Cluster-Management which includes Secrets-Management)
* option to generate resources (e.g Roles, Rolebindings) in one or several namespaces based on namespace names, labels or complex expressions
* optionally patching resources  (merge/replace implemented via musthave versus mustonlyhave controls)
* option to just monitor resources instead of creating/patching them (inform, versus enforce), it is possible to monitor the status of any Kubernetes-Object
* option to delete certain objects (delete implemented via mustnothave)
* option to group objects to certain sets (PolicySets), this feature has both UI/Gitops-Support
* possibility to configure how often checks should be evaluated considering the current status of an evaluated Object
integration with Kyverno/Gatekeeper (implemented using PolicyGenerator but also UI-support)
* Governance focused UI-support (Governance-Dashboard) which enables you to drill down into errors from every Single Policy
* Option to have less/or more fine grained checks by using Configuration-Policies
* Monitoring- and Ansible-integration (gives you option to implement Automated Governance)
* Policies can be used to check for expired Certificates in different namespaces
