## Development & Troubleshooting Methodology

To rapidly prototype and deploy this architecture, the initial Kubernetes and Strimzi YAML manifests were generated using [Gemini](https://gemini.google.com) to speed up boilerplate scaffolding. 

However, to ensure production readiness, security, and architectural integrity, all generated configurations were strictly validated against the official Kubernetes and [Strimzi documentation](https://strimzi.io/documentation/).

Gemini was also utilized as a pair-programming assistant to debug and resolve several complex infrastructure hurdles during the integration of the Kafka Connect plugins. 

### Key Troubleshooting Scenarios Addressed

1. **Strimzi Kaniko Builder & Registry Authentication:** Resolved `access denied` errors when Strimzi's internal build pod attempted to compile the custom Kafka Connect image. Fixed by properly configuring Kubernetes Secrets for Docker Hub authentication and mapping the `pushSecret` to the Strimzi build configuration.
2. **Kafka Engine & Confluent S3 Plugin Class Collisions:** Diagnosed a fatal `500 Internal Server Error` and `java.lang.NoSuchMethodError (Utils.join)` crash on the Sink Connector. Identified the root cause as a dependency conflict between the newer Kafka 4.1.0 engine dropping legacy methods and the Confluent S3 plugin. Resolved by forcing a clean cache rebuild with an upgraded, patched version (`12.1.3`) of the Confluent S3 connector.
3. **Strict API Schema & Secret Management:** Migrated deprecated configuration patterns (like Camel's `${file:...}` syntax and `externalConfiguration` blocks) into the strict `kafka.strimzi.io/v1` API. Successfully routed AWS credentials from secure Kubernetes Secrets directly into the `spec.template.connectContainer.env` block to ensure keys remained hidden from version control and connector configs.

### 📚 Official Reference Documentation

The following official resources were heavily utilized to validate the architecture, manage resources, and deploy the operator:

* **Strimzi Configuration Guide:** [Configuring Strimzi](https://strimzi.io/docs/operators/in-development/configuring.html)
* **Kafka Node Pools Architecture:** [Introduction to Kafka Node Pools](https://strimzi.io/blog/2023/08/14/kafka-node-pools-introduction/)
* **Kubernetes Resource Management:** [Assign Memory Resources to Containers and Pods](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
* **Strimzi Helm Charts:** [Strimzi Helm3 Operator](https://github.com/strimzi/strimzi-kafka-operator/tree/main/helm-charts/helm3/strimzi-kafka-operator)
* **Strimzi Releases & Versions:** [Strimzi GitHub Tags](https://github.com/strimzi/strimzi-kafka-operator/tags)
