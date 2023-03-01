---
# GitOpsifying Cloud Infra with Crossplane
---
##### About Me
![bg contain](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/main/Chapter03/Diagram/Samples/me.png)

---

# Agenda
- Infrastructure Automation History
- what is GitOps?
- Problem
- Solution 
- What Why How Crossplane? 
- Crossplane components overview.
- Demo 
- Q & A
---
# Infrastructure Automation History

Name | Tools | 
-----|------|
Manual | 
Scripting | (Powershell , Bash ) 
Configuration Management | (Puppet ,Chef ,Ansible)
IAC | 
Declarative | (CloudFormation ,Terraform) 
Componentized |(CDK , Pulumi , CDKTF)
Central control plane | (Crossplane) 

---

![bg fit](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/main/Chapter03/Diagram/Samples/GitOps.png)

---
# Problem
 The problem is to achieve **true GitOps** for both **infrastructure** and **applications** while minimizing the use of **multiple tools** and **languages** and reducing **complexity**.

 To have True GitOps you will face challenges:
 * Multiple Tools
 * Multiple Languages
 * Complexity

---

# solution
- **Kubernetes** and **Argo CD**, along with **Crossplane**, are **solutions** that help to solve the problem statement by **providing** a more streamlined and **automated** approach to **managing infrastructure and applications** using **GitOps** principles.


----
# Kubernetes

Kubernetes provides a platform for deploying, scaling, and managing containerized applications, making it easier to manage infrastructure as code.


---
# Argo CD

 Argo CD is a GitOps-based continuous delivery tool that automates the deployment of applications to Kubernetes clusters, helping to ensure that the desired state of the infrastructure is always in sync with the state described in Git.



---
# Crossplane
 Crossplane provides a unified control plane that makes it easier to manage multiple cloud resources using GitOps principles, by allowing administrators to manage infrastructure as code.


---
By combining these tools, it becomes possible to achieve true GitOps for both infrastructure and applications, reducing complexity and minimizing the need to use multiple tools and languages.


---
# Solution Diagram  

![bg contain](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/main/Chapter03/Diagram/Samples/Solution.png)

---
# What is Crossplane?
Crossplane is an open-source multi-cloud control plane that enables users to manage infrastructure as code and automate provisioning, scaling, and management of cloud resources. It provides a common way to provision and manage resources across different cloud providers and can be integrated with Kubernetes.

![bg left:10% 80%](https://cncf-branding.netlify.app/img/projects/crossplane/icon/color/crossplane-icon-color.png)

---

# Why use Crossplane?

![bg fit](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/main/Chapter03/Diagram/Samples/Why_Crossplane.png)


---
#
![bg fit](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/46e8007ac954926a642c37c70e0af13e06f24de7/Chapter03/Presentation/Samples/images/crossplane1.png)

---
# Crossplane components overview 
Component | Abbreviation | Scope 
-----|------|:-----:
Provider | | cluster | 
ProviderConfig | PC | cluster |
Managed Resource | MR | cluster | 
Composition | | cluster |
Composite Resources| XR | cluster | 
Composite Resource Definitions | XRD | cluster |
Claims | XC | namespace | 

---
# Providers
Creates new Kubernetes Custom Resource Definitions for an external service.

### ProviderConfig 
Applies settings for a Provider.

![bg right:60% 80% fit](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/Providers.png?raw=true)

---
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: "xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0"
```
```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```
---
# Managed Resources
A provider resource created and managed by Crossplane inside the Kubernetes cluster.

![bg right fit](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/Managed_Resources.png?raw=tru)

---
```yaml
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: rdspostgresql
spec:
  forProvider:
    region: us-east-1
    dbInstanceClass: db.t2.small
    masterUsername: masteruser
    allocatedStorage: 20
    engine: postgres
    engineVersion: "12"
    skipFinalSnapshotBeforeDeletion: true
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: aws-rdspostgresql-conn
```

---

# Composition
A template for creating multiple managed resources at once.

---
```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example
  labels:
    crossplane.io/xrd: xpostgresqlinstances.database.example.org
    provider: gcp
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XPostgreSQLInstance
  resources:
  - name: cloudsqlinstance
    base:
      apiVersion: database.gcp.crossplane.io/v1beta1
      kind: CloudSQLInstance
      spec:
        forProvider:
          databaseVersion: POSTGRES_12
          region: us-central1
          settings:
            tier: db-custom-1-3840
            dataDiskType: PD_SSD
            ipConfiguration:
              ipv4Enabled: true
              authorizedNetworks:
                - value: "0.0.0.0/0"
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.storageGB
      toFieldPath: spec.forProvider.settings.dataDiskSizeGb
```
---
# Composite Resources
Uses a Composition template to create multiple managed resources as a single Kubernetes object.

![bg right fit](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/Composite_Resources.png?raw=tru)

---
```YAML
apiVersion: database.example.org/v1alpha1
kind: XPostgreSQLInstance
metadata:
  name: my-db
spec:
  parameters:
    storageGB: 20
  compositionRef:
    name: production
  writeConnectionSecretToRef:
    namespace: crossplane-system
    name: my-db-connection-details
```
---
# Composite Resource Definitions
Defines the API schema for Composite Resources and Claims

---
```YAML
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.database.example.org
spec:
  group: database.example.org
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  storageGB:
                    type: integer
                required:
                - storageGB
            required:
            - parameters

```
---
# Claims
Like a Composite Resource Uses a Composition template to create multiple managed resources as a single Kubernetes object., but namespace scoped.

---
```yaml
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  namespace: default
  name: my-db
spec:
  parameters:
    storageGB: 20
  compositionRef:
    name: production
  writeConnectionSecretToRef:
    name: my-db-connection-details
```
---
 
![bg fit](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/How_Composite_Resources_Works.png?raw=true)

---
# Demo
![bg fit ](https://raw.githubusercontent.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/main/Chapter03/Diagram/Samples/Crossplane%20demo.png)

Demo code: https://github.com/itumor/demodoc

---

![bg fit](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/Q&A.png?raw=true)

---
You Can Find Me https://www.linkedin.com/in/Ebrahim-Ramadan

![bg fit ](https://github.com/itumor/End-to-End-Automation-with-Kubernetes-and-Crossplane/blob/main/Chapter03/Diagram/Samples/thank_you.png?raw=true)

![bg fit 80% ](https://chart.googleapis.com/chart?cht=qr&chl=https%3A%2F%2Fwww.linkedin.com%2Fin%2FEbrahim-Ramadan%2F&chs=180x180&choe=UTF-8&chld=L|2)

