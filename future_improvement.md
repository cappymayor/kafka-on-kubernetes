## Future Improvements

To keep the local environment lightweight, repeatable, and straightforward, I deliberately chose pragmatic engineering defaults over infrastructure complexity. In a true enterprise-scale production environment, I would advocate for the following advancements:

### Secret Management Optimization
* **Current State:** Credentials are created imperatively as a local Kubernetes Secret object. This avoids checking standard declarative YAML into Git, which is a risk since anyone can decode native base64-encoded strings.
* **Production Improvement:** Deploy the **External Secrets Operator (ESO)** alongside a secure external store (like AWS Secrets Manager, HashiCorp Vault, or Google Secret Manager). This would allow secrets to be automatically and declaratively managed and synchronized across Kubernetes namespaces without exposing values in plaintext or base64 inside version control.

### Topic Lifecycle Automation
* **Current State:** Topics are left to be automatically initialized by the CDC framework (Debezium/Kafka Connect) as soon as new data streams or database schemas are discovered. 
* **Production Improvement:** While Strimzi supports creating and tracking topics declaratively using the `KafkaTopic` custom resource object, doing so here would duplicate effort given that the CDC layer inherently generates its own mapping. For high-throughput production lines, I would transition to configuring the auto-creation properties specifically within the Kafka Connect runtime settings to pre-define deterministic partition counts and replication factors before traffic flows.

### GitOps & Release Automation Strategy (Why ArgoCD was omitted)

#### Context

For this local setup, manifests were applied directly using kubectl apply. 
In a production environment, manual operations via CLI create configuration drift and lack auditing.

#### The Decision
I explicitly chose not to implement ArgoCD for the local prototyping phase.
Running the ArgoCD controller, Redis cache, application server and other argo components inside Minikube will drastically increases 
CPU and Memory utilization, pulling resources away from the Strimzi operator and Kafka Connect.
In production, the entire objects will be rolled out and managed by ArgoCD. This enable Argo to continuously reconcile the cluster state against Git, ensuring that any ad-hoc changes made by engineers via the CLI are instantly overwritten by the declarative source of truth.
