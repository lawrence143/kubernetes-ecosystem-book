
      [adriano] In this section we're going to cover the design patterns principles and
      # the related available tools currently available from the ecosystem.
      # Feel free to review. Any comment will be appreciated.
      # Thanks.

# Kuberentes Design Patterns
 * [Baseline Patterns](#baseline-patterns)
      * [Declarative patterns](#declarative-patterns)
      * [Behavorial patterns](#behavorial-patterns)
      * [Observability patterns](#observability-patterns)
      * [Life Cycle Conformance patterns](#life-cycle-conformance-patterns)
      * [Structure Patterns](#structure-patterns)
      * [Configuration Patterns](#configuration-patterns)
 * [Advanced patterns](#advanced-patterns)
      * [Kubernetes extension points](#kubernetes-extension-points)
      * [COperator pattern](#operator-patterns)


## Baseline Patterns

### Declarative patterns

- Update strategy
- Blue/Green strategy
- Canary strategy

### Behavorial patterns

- Batch Jobs
- Scheduled Jobs
- Services

### Observability patterns

- Container Healt Check
- Liveness Probe
- Readiness Probe

### Life Cycle Conformance patterns

- Pod temination
- Life cycle hooks: post-start and pre-stop

### Structure Patterns

- Sidecar
- Initializer
- Ambassador
- Adapter

### Configuration Patterns

- Environment variables
- Config Maps
- Secrets

## Advanced patterns

### Kubernetes extension points

- Services Catalog
- APIs aggregation server
- Custom Resources Definition
- Custom Controllers

### Operator pattern

- Platform as Code
- Operator framework
- Operator Lifecycle Manager
- Kubebuilder
- Metacontroller
