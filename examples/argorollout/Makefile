export FLAGGER_NS = linkerd

.id:
	git config user.email | awk -F@ '{print $$1}' > .id

cluster:
	minikube stop; minikube delete;
	minikube start --cpus=2
	minikube addons enable ingress
	minikube addons enable metrics-server
	@echo 'Built minikube cluster'

metric-server: cluster
	kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
	@echo 'Done with deploy Metrics Server'

linkerd:
	linkerd check --pre                     # validate that Linkerd can be installed
	linkerd install | kubectl apply -f -    # install the control plane into the 'linkerd' namespace
	linkerd check                           # validate everything worked!
	@echo 'Done with deploy Linkerd'

dashboard:
	linkerd dashboard                      # launch the Linkerd dashboard

rollout: cluster linkerd
	kubectl create namespace argo-rollouts
	kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
	@echo 'Done with deploy Argo Rollout'

test:
	kubectl apply -f examples/smi/namespace.yaml
	kubectl apply -f examples/smi/rollout.yaml
	kubectl apply -f examples/smi/services.yaml
	kubectl apply -f examples/smi/ingress.yaml

status:
	kubectl -n test get rollout
	kubectl -n test get service
	kubectl -n test get ingress
	kubectl -n test get trafficsplit

watch:
	kubectl argo rollouts -n test get rollout rollouts-demo --watch

update-yellow:
	kubectl argo rollouts -n test set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow

update-blue:
	kubectl argo rollouts -n test set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:blue

clean:
	minikube stop; minikube delete