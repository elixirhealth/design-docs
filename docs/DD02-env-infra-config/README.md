# DD02 - Environment Configuration & Infrastructure

This design doc describes an approach for managing deployment environment infrastructure and configuration. 

#### Goals

Describe
- different deployment environments and their roles
- configuration & infrastructure management for application and core services
- testing and deployment process


#### Non-Goals

Describe
- testing environment b/c not totally sure of best approach for pre-develop feature config & infra testing yet
- if/how staging data will be hydrated from (a subset of) prod


#### Key Terms/Products

- [Docker](https://www.docker.com) containers are the smallest unit of our deployment, usually running a single service within them
- [Kubernetes](https://kubernetes.io) is the container orchestration system that wires together and configures all the containers
- [Terraform](https://www.terraform.io) is the declarative cloud infrastructure management solution that configures and provisions the cloud infrastructure
- Elxir service layers
   - the *application* layer contains the services and their infra involved in Elxir business logic (i.e., the majority of all services)
   - the *core* layer contains services and infra involved supporting the application layer, including
      - Kubernetes cluster
      - [Prometheus](https://prometheus.io/) monitoring and alerting
      - [Grafana](https://grafana.com/) dashboards
      - [Vault](https://www.vaultproject.io/) for secrets 
      - [Consul](https://www.consul.io/) for additional service discovery and config 
      - [Google Container Registry](https://cloud.google.com/container-registry/) for private Docker container images

#### Orchestration

- all services will be deployed through stand-alone Docker containers on a Kubernetes (k8s) cluster
- all infrastructure (including the Kubernetes cluster) will be defined by Terraform (TF)
- all infrastructure will reside in Google Cloud Platform


#### Deployment Environments

- the *production* environment contains the services and infrastructure used by external users
   - runs off master branch and/or versioned releases for each service and deploy repo
- the *staging* environment contains the services and infrastructure for internal testing
   - runs off develop branch and/or snapshot releases for each service and deploy repo
      - staging services only point to other staging services (not other production services)
	 - the staging environment is completely isolated from production via separate GCP project
      - all naming, monitoring, and alerting is exactly the same as that of production
	 - only exception might be service replica count, resource limits, and k8s cluster node machine types


#### Deployment Configuration & Infrastructure Management

Deployment config and infra management works through git repos. Each has a `master` branch, off which pull requests are created and into which they are merged. The repos are separated by environment and layer:
- [deploy-staging-app](https://github.com/elxirhealth/deploy-staging-app)
- [deploy-staging-core](https://github.com/elxirhealth/deploy-staging-core)
- deploy-prod-app
- deploy-prod-core

Each repo has a similar file structure, organized by service/infrastructure name. For example:
```
deploy-staging-app
- kubernetes
   - access
      - env.yml
      - service.yml
   - catalog
      - env.yml
      - service.yml
   - circulation
      - env.yml
      - service.yml
   - courier
      - env.yml
      - service.yml
   - directory
      - env.yml
      - service.yml
   - identity
      - env.yml
      - service.yml
   - libri
      - env.yml
      - service.yml
      - service.yml.template
      - gen.go
   - timeline
      - env.yml
      - service.yml
- terraform
   - directory
      - module.tf
   - libri
      - module.tf
   - main.tf
```

Each service/component has a k8s configuration, split into two files:
- `env.yml` is a k8s [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values) defining settings particular to the environment (i.e., staging)
   - literal values define things like the number of replicas and resource limits on the pods
- `service.yml` defines the bulk of the service and configuration and is not likely to change between environments
Complex service configurations (e.g., that of Libri) may also have a Golang `service.yml.template` along with a `gen.go` file that generates the resulting `service.yml` based on some input parameters. 

The service/component may also have a [Terraform Module](https://www.terraform.io/docs/modules/usage.html), which can define the infrastructure resources needed by the service (e.g., a database). Instances of these modules are defined in `terraform/main.tf`.


#### Testing 

Most k8s configs should be able to be tested locally in [minikube](https://github.com/kubernetes/minikube), but for now we will gloss over the details, since we are not exactly sure what method makes the most sense.

Changes to terraform configs can be inspected by running `terraform plan` from the developer's local machine. This plan output should be included in the PR.


#### Deployment

For now, deployments in both staging and prod will happen manually. We may want to automating the staging deploying, as something that happens at the end of the CircleCI build, but we won't do it right away. We probably should never automate production deploys, always enforcing that a human plan and then apply themselves from their local machine.












