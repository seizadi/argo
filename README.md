# Argo

Argo project is made up of a number of solutions, I will review them here:
   * Argo Workflow (ArgoWF)
   * ArgoCD
   * Argo Rollout

## Argo Workflow (ArgoWF) is a workflow solution
[It is compared to other workflow solutions](https://www.youtube.com/watch?v=oXPgX7G_eow&utm_campaign=Singapore&utm_content=97139107&utm_medium=social&utm_source=twitter&hss_channel=tw-381629790)
I was intersted in it for the CI/CD use case but realized Lots of people look at for
data pipeline and ML use cases, which was an intersting use case I had not thought of for
Argo DAG. Here are some more options across those use cases, with native kubernetes solutions in the top:

   * [Brigade](https://brigade.sh/)
   * [Fision](https://fission.io) and [Fision Workflows](https://docs.fission.io/docs/workflows/)
   * [Airship](https://www.airshipit.org/) [Shipyard DAG](https://airshipit.readthedocs.io/projects/shipyard/en/latest/)
   * [Apache Airflow](https://airflow.apache.org/)
   * [Oozie](https://oozie.apache.org/docs/3.1.3-incubating/DG_Overview.html)
   * [Luigie](https://luigi.readthedocs.io/en/stable/workflows.html)
   * [AWS Data Pipeline](https://aws.amazon.com/datapipeline/)
   * [AWS Step Functions](https://aws.amazon.com/step-functions/)
   * [Spinnaker](https://www.spinnaker.io/)
   * [AWS Code Pipeline](https://aws.amazon.com/codepipeline/)
   * [Harness](https://harness.io/)
   * NiFi
   * Azkaban

This [blog is for CI/CD workflow](https://medium.com/axons/ci-cd-with-argo-on-kubernetes-28c1a99616a9)
Here we discuss [Jenkins](https://www.jenkins.io/) and [Spinnaker](https://www.spinnaker.io/)
This discussion of workflow seems heavy on the CI side with Docker in Docker to build containers
and Ansible to run tests, I am not sure I'm on board with that being the critical item in my
problems, probably why Argo CI is not heavily supported. It is a well written article in covering
the detail of ArgoWF. 

The federation pattern of a master and client clusters is one that I like in providing seperation of
concern, so 
[this article I liked](https://admiralty.io/blog/running-argo-workflows-across-multiple-kubernetes-clusters/)
and there is this article from 
[Crossplane on ArgoWF]()

In the open source there are three ways to do federation:
The [KubeFed](https://github.com/kubernetes-sigs/kubefed) is better suited for greenfield where you
are building something from ground up which requires clients to use new, federated APIs, 
e.g., federated deployment templates, placements, and overrides.

The Admiralty and Crossplane are better suited in using existing kubernetes resources and wrapping them
in the CRDs and annotations to proxy them to target clusters.  
 
At this time Admiralty and Crossplane have different goals, Admiralty is focused on primitives around
federating clusters at the pod level, and seamlessly scheduling workloads across multiple clusters.
Working with kubernetes Pod is a start and other low level elements like
[Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 
and service mesh like Istio or Linkerd and other workloads like Knative.

Crossplane is focused on Multi-Cloud and Application level deployment as their Workloads.
[The vision more grand in terms of what they want to accomplish](https://github.com/crossplane/crossplane/blob/master/design/design-doc-complex-workloads.md),
and the level of elements they want to orchestrate is more
coarse and not as aware of the underlying elements that are federated to remote cluster controllers. 
[This is a good description of that pattern](https://github.com/crossplane/crossplane/blob/master/design/design-doc-complex-workloads.md#federation-style-envelope-resources).

## ArgoWF Execution
At the top level you have the steps and you could also have a higher level 
[Directed Acyclic Graph (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph). DAG would
encapsulate the steps and allow execution to follow the desired graph.

The execution of the steps are done using a container Pod. In additional you can run
additional items along with the steps either with
[Daemon Containers](https://github.com/argoproj/argo/blob/master/examples/README.md#daemon-containers) 
that run for full workflow lifecycle or
[Sidecar Continaers](https://github.com/argoproj/argo/blob/master/examples/README.md#sidecars) 
for just the step in the same Pod as the step container. A special use of the sidecar pattern is the 
[Docker In Docker Sidecar](https://github.com/argoproj/argo/blob/master/examples/README.md#docker-in-docker-using-sidecars)
you can use for CI to build a contianer image and upload to registry.

## ArgoWF Review

The [ArgoWF Conditionals](https://github.com/argoproj/argo/tree/master/examples#conditionals) is
interesting aspect, the YAML for DAG and steps are fairly straightforward. 
Here are some examples...
***When*** conditionals:
```bash
        when: "{{steps.flip-coin.outputs.result}} == heads"
```
`when` conditional:
```yaml
        when: "{{steps.flip-coin.outputs.result}} == heads"
```

`retryStrategy` conditional that will dictate how failed or errored steps are retried:
```yaml
    retryStrategy:
      limit: 10
      retryPolicy: "Always"
      backoff:
        duration: "1"      # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
        factor: 2
        maxDuration: "1m"  # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
```
`exit-handler` invoked at the end of workflow:
```yaml
onExit: exit-handler                  # invoke exit-hander template at end of the workflow
  templates:
  # primary workflow template
  - name: intentional-fail
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo intentional failure; exit 1"]

  # Exit handler templates
  # After the completion of the entrypoint template, the status of the
  # workflow is made available in the global variable {{workflow.status}}.
  # {{workflow.status}} will be one of: Succeeded, Failed, Error
  - name: exit-handler
    steps:
    - - name: notify
        template: send-email
```
`activeDeadlineSeconds` to limit the elapsed time for a workflow:
```yaml
  templates:
  - name: sleep
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for 1m; sleep 60; echo done"]
    activeDeadlineSeconds: 10           # terminate container template after 10 seconds
```
Workflows can be suspended by
```bash
argo suspend WORKFLOW
```
Or by specifying a `suspend` step on the workflow:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: suspend-template-
spec:
  entrypoint: suspend
  templates:
  - name: suspend
    steps:
    - - name: build
        template: whalesay
    - - name: approve
        template: approve
    - - name: delay
        template: delay
    - - name: release
        template: whalesay

  - name: approve
    suspend: {}

  - name: delay
    suspend:
      duration: 20    # Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"

  - name: whalesay
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
```
Once suspended, a Workflow will not schedule any new steps until it is resumed. 
It can be resumed manually by
```bash
argo resume WORKFLOW
```
Or automatically with a `duration` limit as the example above.

## ArgoWF UI
It is primitive but does a good job showing you a view of what the ArgoWF controller sees, it is
not designed to be multi-tenant, so it lacks security in knowing who is logging in and what they
can see and do on ArgoWF. The ArgoWF UI is not on LoadBalancer so it is not directly exposed outside.

## ArgoCD
This is a [good tutorial on ArgoCD](https://www.youtube.com/watch?v=r50tRQjisxw)
from last KubeCon, which uses following
[Github repo for handon lab](https://github.com/gitops-workshop/my-app-deployment). I moved
the content here so that it would be easier to reference so now it
under `github.com/seizadi/argo/argocd/lab`
```bash
cd argocd/lab
kustomize build base
```
For the handson lab they used Digital Ocean we will do it on Minikube,
```bash
minikube start
```
[see ArgoCD installation instructions](https://argoproj.github.io/argo-cd/getting_started/):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default
```
Install ArgoCD CLI, I'm using MacOS:
```bash
brew tap argoproj/tap
brew install argoproj/tap/argocd
```
Access The Argo CD API Server, by default, the Argo CD API server is not exposed with an external IP. 
To access the API server, so we change the argocd-server service type to LoadBalancer:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```
