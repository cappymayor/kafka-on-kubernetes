## Future Improvements

To keep the local environment lightweight, repeatable, and straightforward, I deliberately chose pragmatic engineering defaults over infrastructure complexity. In a true enterprise-scale production environment, I would advocate for the following advancements:

### Secret Management Optimization
* **Current State:** Credentials are created imperatively as a local Kubernetes Secret object. This avoids checking standard declarative YAML into Git, which is a risk since anyone can decode native base64-encoded strings.
* **Production Improvement:** Deploy the **External Secrets Operator (ESO)** alongside a secure external store (like AWS Secrets Manager, HashiCorp Vault). This would allow secrets to be automatically and declaratively managed and synchronized across Kubernetes namespaces without exposing values in plaintext or base64 inside version control.

### Topic Lifecycle Automation
* **Current State:** Topics are left to be automatically initialized by the CDC framework (Debezium/Kafka Connect) as soon as new data streams or database schemas are discovered. 
* **Production Improvement:** While Strimzi supports creating and tracking topics declaratively using the `KafkaTopic` custom resource object, doing so here would duplicate effort given that the CDC layer inherently generates its own mapping. For high-throughput production lines, I would transition to configuring the auto-creation properties specifically within the Kafka Connect runtime settings to pre-define deterministic partition counts and replication factors before traffic flows.

### GitOps & Release Automation Strategy (Why ArgoCD was omitted)

#### Context

- For this local setup, manifests were applied directly using kubectl apply. In a production environment, manual operations via CLI create configuration drift and lack auditing.

#### The Decision
- I explicitly chose not to implement ArgoCD for the local prototyping phase.
  Running the ArgoCD controller, Redis cache, application server and other argo components inside Minikube will drastically increases 
  CPU and Memory utilization, pulling resources away from the Strimzi operator and Kafka Connect.
- In production, the entire objects will be rolled out and managed by ArgoCD. This enable Argo to continuously reconcile the cluster state against Git, ensuring that any ad-hoc changes made by engineers via the CLI are instantly overwritten by the declarative source of truth.

#### Enterprise Security & Container Registry 

While the local development prototype utilizes static access keys and public container registries to minimize bootstrap complexity, a production rollout would transition to cloud-native security and container management frameworks.

### 1. IAM Roles for Service Accounts (IRSA) via OIDC vs. Static Secrets

#### Current Local Implementation
* **Mechanism:** Static `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` stored inside a Kubernetes Secret and injected as pod environment variables.
* **Risk Profile:** High. Static credentials introduce risks of token leakage, require manual rotational lifecycle management, and violate the principle of least privilege if keys are over-provisioned.

#### Production Target: IRSA (IAM Roles for Service Accounts)
In a production AWS EKS deployment, static AWS credentials will be completely eliminated. Instead, we will leverage **IRSA** combined with an **OpenID Connect (OIDC)** identity provider.

#### Amazon Elastic Container Registry (ECR) vs. Docker Hub
- For this setup, Custom Kafka Connect images containing the Debezium and Confluent JAR files are pushed to a public Docker Hub repository using local build secrets.

- Risk Profile: Medium. Public registries introduce risk of image poisoning, docker pull-rate throttling limits, and data exposure if internal custom connector configurations are accidentally baked into the image layers.

### Production Target: AWS Elastic Container Registry (ECR)
- For production workloads, we will migrate the image compilation destination to an internal, private Amazon ECR registry.

- Seamless Authentication: The Strimzi operator running on AWS worker nodes will authenticate with ECR natively using IAM instance profiles, eliminating the need to maintain, rotate, or store Docker Hub pushSecret login credentials in Kubernetes manifests.
