[←]: ../README.md
[cimreg]: ./cimreg/README.md
[rancher]: ./rancher/README.md
[mxui]: ./mxui/README.md
[Developer Howto]: #-developer-mode
[End-User Mode]: #-end-user-mode
[Install Howto]: #how-to-install
# [←] Helm Charts

This **section** lists the `helm` **charts** created and maintained by the organization.

## Terminology
- A `helm` **chart** is a set of Kubernetes YAML files used to deploy a **containerized application**.  
- These files are called **manifests**.  
- A **manifest** describes a Kubernetes resource (e.g., Deployment, Service, ConfigMap) that runs or manages containers.

## Projects

| Name      | Purpose                  |
|-----------|--------------------------|
| [cimreg]  | Container Image Registry |
| [mxui]    | MX User Interface        |




- See each project’s README for specific installation instructions.

# How to Install
This section describes two installation modes depending on your role:
- [Developer Howto]
- [End-User Mode]
## [←][Install Howto] Developer Mode
Use this mode if you want to **update, build, and publish** Helm charts. Typical tasks include:  
- Cloning this git repository.  
- Modifying one or more chart manifests.  
- Testing it locally.  
- Building the chart locally.  
- Pushing the chart to a Helm registry.  


## [←][Install Howto] End-User Mode
Use this mode if you only want to **deploy the chart** without modifying it. Typical tasks include:
- Install a prebuild chart from the registry

```bash
helm install <release-name> oci://ghcr.io/abtransitionit/.../<chart-name> --version <chart-version>
