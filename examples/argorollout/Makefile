export FLAGGER_NS = linkerd

.id:
	git config user.email | awk -F@ '{print $$1}' > .id

cluster:
	minikube stop; minikube delete;
#    💡  Due to docker networking limitations on darwin, ingress addon is not supported for this driver.
#    Alternatively to use this addon you can use a vm-based driver:
#
#            'minikube start --vm=true'
# Trying v1.17.5 instead of v1.18.3
	minikube start --cpus=3 --kubernetes-version v1.17.5
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

flagger: cluster linkerd
	kubectl apply -k github.com/weaveworks/flagger//kustomize/linkerd
	@echo 'Done with deploy Flagger'

rollout: cluster linkerd
	kubectl create namespace argo-rollouts
	# Use latest; stable does not have support for SMI yet
	# kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/latest/manifests/install.yaml
	kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/manifests/install.yaml
	@echo 'Done with deploy Argo Rollout'

test:
	kubectl apply -f examples/smi/namespace.yaml
	kubectl apply -f examples/smi/rollout.yaml
	kubectl apply -f examples/smi/services.yaml
	kubectl apply -f examples/smi/ingress.yaml

test-deployment:
	kubectl apply -f examples/deployment/namespace.yaml
	kubectl apply -f examples/deployment/deployment.yaml
	kubectl apply -f examples/deployment/services.yaml
	kubectl apply -f examples/deployment/ingress.yaml

test-flagger:
	kubectl apply -k workloads
	@echo 'Done with deploy test application'

status:
	kubectl -n test get rollout
	kubectl -n test get service
	kubectl -n test get ingress
	kubectl -n test describe trafficsplit rollouts-demo
	kubectl argo rollouts -n test get rollout rollouts-demo

watch:
	kubectl argo rollouts -n test get rollout rollouts-demo --watch

promote:
	kubectl argo rollouts -n test promote rollouts-demo

update-yellow:
	kubectl argo rollouts -n test set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:yellow

update-blue:
	kubectl argo rollouts -n test set image rollouts-demo rollouts-demo=argoproj/rollouts-demo:blue

clean:
	minikube stop; minikube delete
