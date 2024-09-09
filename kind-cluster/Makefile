# Variables
reg_name=my-registry
reg_port=5001

# Define the shell to use
SHELL := /bin/bash

# Prerequisites
## Ensure you have the following tools installed:
## - Go: https://golang.org/doc/install
## - Kind: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
## - Kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/
## - Docker: https://docs.docker.com/get-docker/
## - Homebrew: https://brew.sh/
## - Helm: https://helm.sh/docs/intro/install/

## Main Targets
.PHONY: all
all: install-tools setup-env

.PHONY: install-tools
install-tools: check-dependencies install-kind install-cloud-provider-kind install-postgres-server install-dbeaver

.PHONY: setup-env
setup-env: create-kind-cluster create-local-registries add-registry-config-to-nodes connect-registry-to-cluster install-ingress-nginx

# Check for required dependencies
.PHONY: check-dependencies
check-dependencies:
	@echo "Checking dependencies..."
	@command -v go >/dev/null 2>&1 || { echo >&2 "Go is required but not installed. Aborting."; exit 1; }
	@command -v kind >/dev/null 2>&1 || { echo >&2 "Kind is required but not installed. Aborting."; exit 1; }
	@command -v kubectl >/dev/null 2>&1 || { echo >&2 "Kubectl is required but not installed. Aborting."; exit 1; }
	@echo "All dependencies are installed."

# Install Kind if not installed
.PHONY: install-kind
install-kind:
	@if ! kind version >/dev/null 2>&1; then \
		echo "Installing Kind..."; \
		GO111MODULE=on go install sigs.k8s.io/kind@latest; \
		echo "Kind installed successfully."; \
	else \
		echo "Kind is already installed."; \
	fi

# Create a Kind cluster
.PHONY: create-kind-cluster
create-kind-cluster:
	@echo "Creating Kind cluster..."
	@if ! kind get clusters | grep -q "^tooling$$"; then \
		if [ -f cluster.yaml ]; then \
			kind create cluster --config cluster.yaml; \
		else \
			kind create cluster; \
		fi; \
		echo "Kind cluster created successfully."; \
	else \
		echo "Kind cluster already exists."; \
	fi


# Create a local Docker registry
.PHONY: create-local-registries
create-local-registries:
	@echo "Creating local Docker registry..."
	@if [ "$$(docker inspect --format '{{.State.Running}}' $(reg_name) 2>/dev/null || echo 'false')" != 'true' ]; then \
		docker run -d --restart=always -p "127.0.0.1:$(reg_port):5000" --network bridge --name "$(reg_name)" registry:2; \
		if [ $$? -ne 0 ]; then \
			echo "Failed to create Docker registry."; \
			exit 1; \
		fi; \
		echo "Local Docker registry created successfully."; \
	else \
		echo "✓ Local Docker registry is already running."; \
	fi


# Add registry configuration to Kind nodes
.PHONY: add-registry-config-to-nodes
add-registry-config-to-nodes:
	@echo "Adding registry config to nodes..."
	REGISTRY_DIR="/etc/containerd/certs.d/localhost:$(reg_port)"
	for node in $$(kind get nodes); do \
		docker exec "$${node}" mkdir -p "$${REGISTRY_DIR}"; \
		docker exec -i "$${node}" sh -c "cat > $${REGISTRY_DIR}/hosts.toml <<EOF \
[host.\"http://$(reg_name):5000\"] \
EOF"; \
	done
	@echo "✓ Registry config added to nodes successfully."

# Connect the local registry to the Kind cluster
.PHONY: connect-registry-to-cluster
connect-registry-to-cluster:
	@echo "Connecting registry to cluster..."
	@if [ "$(shell docker inspect -f='{{json .NetworkSettings.Networks.kind}}' "${reg_name}")" = 'null' ]; then \
		docker network connect "kind" "${reg_name}"; \
	fi
	@echo "✓ Registry connected to cluster successfully."

# Document the local registry in Kubernetes
.PHONY: document-local-registry
document-local-registry:
	@echo "Documenting local registry in Kubernetes..."
	kubectl apply -f - << EOF
	apiVersion: v1
	kind: ConfigMap
	metadata:
	  name: local-registry-hosting
	  namespace: kube-public
	data:
	  localRegistryHosting.v1: |
		host: "localhost:$(reg_port)"
		help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
	EOF
	@echo "✓ Local registry documented successfully."

# Install cloud provider for Kind
.PHONY: install-cloud-provider-kind
install-cloud-provider-kind:
	@echo "Installing cloud-provider-kind..."
	GO111MODULE=on go install sigs.k8s.io/cloud-provider-kind@latest
	@echo "Cloud-provider-kind installed successfully."

# Install Ingress Nginx in the Kind cluster
.PHONY: install-ingress-nginx
install-ingress-nginx:
	@echo "Installing Ingress Nginx..."
	kubectl delete job --namespace ingress-nginx ingress-nginx-admission-create ingress-nginx-admission-patch || true
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
	@echo "Waiting for Ingress Nginx pods to be created..."
	sleep 10 # Give some time for the pods to start being created
	POD_NAME=""
	while [ -z "$${POD_NAME}" ]; do \
		POD_NAME=$$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}' 2>/dev/null); \
		if [ -z "$${POD_NAME}" ]; then \
			echo "Waiting for Ingress Nginx pods to be created..."; \
			sleep 5; \
		fi; \
	done
	@echo "Ingress Nginx pod $${POD_NAME} created. Waiting for it to be ready..."
	kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=180s
	@echo "✓ Ingress Nginx installed and ready."


# Clean up the Kind cluster
.PHONY: clean
clean:
	@echo "Deleting Kind cluster..."
	@kind delete clusters tooling
	@echo "✓ Kind cluster deleted successfully."