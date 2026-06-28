## Architectural Design Decisions & Thought Process

When approaching this assessment to deploy a production-ready Kafka Cluster on Kubernetes, my primary goal was to ensure **maintainability, scalability, high availability, and adherence to cloud-native best practices.** Below is the rationale behind the key architectural choices made during this setup.

---

### 1. High Availability & KRaft Quorum Stability

**Decision:** I chose to deploy a minimum configuration of **3 Kafka brokers** operating as a combined broker/controller pool.

**Why this is Best Practice:**
* **Quorum Resilience:** In modern KRaft-based Kafka clusters, maintaining a strict majority (quorum) is vital for election stability. By deploying 3 brokers, the cluster can gracefully tolerate the complete failure of the active controller. The remaining 2 controllers still represent a majority ($2 > 3/2$), allowing them to immediately carry out a split-second quorum election to appoint a new leader without causing cluster-wide downtime.

---

### 2. Strict Data Durability Guarantees (`min.insync.replicas`)

**Decision:** I configured a cluster-wide default of `min.insync.replicas=2` across all Kafka topics.

**Why this matters:**
* **Preventing Data Loss:** Working hand-in-hand with our 3-broker topology, setting the minimum in-sync replicas to 2 ensures that when producers publish data using `acks=all`, Kafka guarantees the message is safely written to the partition leader *and* at least one follower before acknowledging success.
* **Fault Tolerance:** If a broker hosting a partition leader suddenly goes down, the data is already safely replicated on at least one other active broker. This prevents data loss and allows the cluster to safely promote the follower to leader while maintaining system integrity.

---

### 3. Resource Management & Guaranteed QoS Class

**Observation:** During early local testing, Kafka brokers experienced frequent JVM Out-Of-Memory (OOM) crashes due to default resource starvation.

**Mitigation & Strategy:**
* **Guaranteed QoS Tiering:** I configured explicit Kubernetes resource `requests` and `limits` for both CPU and memory, ensuring they are mapped to **identical values**. 
* **Eviction Protection:** In Kubernetes, matching requests and limits automatically places the broker pods into the strict **Guaranteed Quality of Service (QoS)** class. In the event of cluster-wide resource starvation, the kubelet prioritizes evicting pods in the `BestEffort` or `Burstable` classes first. This safeguards our stateful Kafka infrastructure, ensuring the brokers are the absolute last pods to be terminated when the node is under pressure.

---

### 4. Externalizing the Strimzi Operator (Upstream Helm Reference)

**Decision:** I chose to point directly to the upstream Strimzi Helm repository rather than vendoring/copying all the operator template files into this repository alongside our custom `Chart.yaml` and `values.yaml` overrides.

**Why this is Best Practice:**
* **Upstream Maintainability:** Forcing deep-copied template files into a local repository introduces significant technical debt. It makes upgrading the Strimzi Operator a manual, error-prone process of diffing hundreds of lines of CRDs and cluster roles.
* **Separation of Concerns:** The Helm chart should define *our configuration layer*, not house the entire codebase of the third-party operator. 
* **GitOps & Clean Infrastructure as Code (IaC):** By leveraging standard Helm dependencies, we keep our codebase incredibly lightweight, auditable, and easily upgradeable via a simple `helm dependency update`.

---

### 5. Decoupling Configuration and Infrastructure Secrets

**Decision:** Hardcoded database credentials inside KafkaConnector YAML files were extracted and migrated to native Kubernetes Secrets.

**Why this matters:**
* **Security & Git Ingestion:** Credentials are completely scrubbed from the source code, rendering manifests perfectly safe to commit to version control.
* **Modern Strimzi v1 API Adherence:** Instead of utilizing deprecated configuration properties, secrets are dynamically injected as environment variables directly inside the Connect pod templates via the `template.connectContainer.env` block, utilizing Kafka's built-in `EnvVarConfigProvider`.

---

## Future Improvements & Pragmatic Trade-offs

To keep the local environment lightweight, repeatable, and straightforward, I deliberately chose pragmatic engineering defaults over infrastructure complexity. In a true enterprise-scale production environment, I would advocate for the following advancements:

### Secret Management Optimization
* **Current State:** Credentials are created imperatively as a local Kubernetes Secret object. This avoids checking standard declarative YAML into Git, which is a risk since anyone can decode native base64-encoded strings.
* **Production Improvement:** Deploy the **External Secrets Operator (ESO)** alongside a secure external store (like AWS Secrets Manager, HashiCorp Vault, or Google Secret Manager). This would allow secrets to be automatically and declaratively managed and synchronized across Kubernetes namespaces without exposing values in plaintext or base64 inside version control.

### Topic Lifecycle Automation
* **Current State:** Topics are left to be automatically initialized by the CDC framework (Debezium/Kafka Connect) as soon as new data streams or database schemas are discovered. 
* **Production Improvement:** While Strimzi supports creating and tracking topics declaratively using the `KafkaTopic` custom resource object, doing so here would duplicate effort given that the CDC layer inherently generates its own mapping. For high-throughput production lines, I would transition to configuring the auto-creation properties specifically within the Kafka Connect runtime settings to pre-define deterministic partition counts and replication factors before traffic flows.
