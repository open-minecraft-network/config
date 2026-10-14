.PHONY: main
main:
	@echo "For local development run: make init-local."

NAMESPACE_MANIFESTS := $(wildcard config/local/servers/*namespace*.yaml)

.PHONY: create-namespaces
create-namespaces:
	kubectl apply $(addprefix -f ,$(NAMESPACE_MANIFESTS))

.PHONY: create-forwarding-secret
create-forwarding-secret:
	@kubectl -n servers create secret generic proxy-forwarding-secret \
		--from-literal=forwarding.secret="$$(openssl rand -hex 32)" \
		--dry-run=client -o yaml | kubectl apply -f -

.PHONY: setup-helm-repos
setup-helm-repos:
	helm repo add kyverno https://kyverno.github.io/kyverno/
	helm repo add agones https://agones.dev/chart/stable
	helm repo update

.PHONY: install-kyverno
install-kyverno: setup-helm-repos
	@echo Installing Kyverno
	helm upgrade --install kyverno kyverno/kyverno \
		--namespace kyverno-system \
		--create-namespace
	@echo Applying Kubernetes Pod Security Standarts
	helm upgrade --install kyverno-policies kyverno/kyverno-policies -n kyverno-system

.PHONY: uninstall-kyverno
uninstall-kyverno:
	@echo Uninstalling Kyverno Policies
	helm uninstall kyverno-policies -n kyverno-system || true
	@echo Uninstalling Kyverno
	helm uninstall kyverno -n kyverno-system || true
	@echo Cleaning up lingering Kyverno webhooks
	kubectl delete validatingwebhookconfigurations -l webhook.kyverno.io/managed-by=kyverno --ignore-not-found=true
	kubectl delete mutatingwebhookconfigurations -l webhook.kyverno.io/managed-by=kyverno --ignore-not-found=true
	@echo Deleting Kyverno namespaces
	kubectl delete namespace kyverno kyverno-system || true
	
.PHONY: install-agones
install-agones: setup-helm-repos create-namespaces
	helm upgrade --install agones agones/agones \
		--namespace agones-system \
		--create-namespace \
		-f config/local/agones/values.yaml

.PHONY: unistall-agones
uninstall-agones:
	@echo Uninstalling Agones
	helm uninstall agones -n agones-system || true
	@echo Deleting Agones namespace
	kubectl delete namespace agones-system || true

.PHONY: init-local
init-local: setup-helm-repos install-kyverno install-agones
	@echo "COMPLETED!"