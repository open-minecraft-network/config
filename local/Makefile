APP_NAME ?= omcn-local
MANIFEST_DIR ?= ./live

.PHONY: main
main:
	@echo "For local development run: make init-local."

.PHONY: ensure-namespaces
ensure-namespaces:
	kubectl apply -f live/namespaces.yaml

.PHONY: create-forwarding-secret
create-forwarding-secret:
	@kubectl -n servers create secret generic proxy-forwarding-secret \
		--from-literal=forwarding.secret="$$(openssl rand -hex 32)" \
		--dry-run=client -o yaml | kubectl apply -f -

.PHONY: install-local-argocd
install-local-argocd: ensure-namespaces
	kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
	@echo "Waiting for ArgoCD server to be ready..."
	kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=90s

.PHONY: uninstall-local-argocd
uninstall-local-argocd:
	kubectl delete -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

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
		--create-namespace \
		--timeout 10m
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
install-agones: setup-helm-repos ensure-namespaces
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
init-local: ensure-namespaces create-forwarding-secret install-local-argocd setup-helm-repos install-kyverno install-agones
	@echo "\n========================================"
	@echo "INITIALIZATION COMPLETED!"
	@echo "ArgoCD initial password:"
	@kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
	@echo "\n\nNext steps:"
	@echo "1. Run 'make port-forward' in a separate terminal."
	@echo "2. Run 'argocd login localhost:8080' in this terminal."
	@echo "3. Run 'make watch-local' to start developing."
	@echo "========================================\n"
	
.PHONY: sync-local
sync-local:
	@echo "Syncing local directory $(MANIFEST_DIR) to ArgoCD app $(APP_NAME)..."
	argocd app sync $(APP_NAME) --local $(MANIFEST_DIR)

.PHONY: watch-local
watch-local:
	@echo "Watching $(MANIFEST_DIR) for changes... (Requires fswatch)"
	fswatch -o $(MANIFEST_DIR) | xargs -n1 -I{} argocd app sync $(APP_NAME) --local $(MANIFEST_DIR)