# Synthesizing and validating NetworkPolicies using Tekton

This is a step-by-step demo of integrating NP-Guard components into a tekton-based CI/CD pipeline. The result should be an implementation of the following diagram.

![CI-integration](https://github.com/np-guard/np-guard.github.io/raw/master/ci-integration-option.png).

## Prerequisites
* A Kubernetes cluster version 1.15 or higher with:
  1. RBAC enabled 
  2. `cluster-admin` role granted for the current user
  3. Tekton Pipelines v0.21 or later installed 
  4. Tekton Triggers v0.18 or later installed
  5. A way to expose a service to the outside internet (e.g., OpenShift Route or Ingress).
  6. A [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) called `my-pvc` with at least 1GB capacity and any write access-mode

* A mirror GitHub repository of this repository. Check [these instructions](https://docs.github.com/en/repositories/creating-and-managing-repositories/duplicating-a-repository).
* A clone of the mirror repository on your local file system

### Install required tasks
```commandline
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.5/git-clone.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-cli/0.3/git-cli.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/github-open-pr/0.2/github-open-pr.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/github-add-comment/0.6/github-add-comment.yaml
kubectl apply -f https://raw.githubusercontent.com/np-guard/netpol-synthesizer/master/tekton/netpol-synthesis-task.yaml
kubectl apply -f https://raw.githubusercontent.com/np-guard/network-config-analyzer/master/tekton/netpol-report-task.yaml
kubectl apply -f https://raw.githubusercontent.com/np-guard/network-config-analyzer/master/tekton/netpol-diff-task.yaml
kubectl apply -f https://raw.githubusercontent.com/np-guard/baseline-rules-verifier/master/tekton/netpol-verify-task.yaml
```

### Install GitHub access secrets
```commandline
kubectl create secret generic my-ssh-credentials --from-file=~/.ssh/known_hosts --from-file=~/.ssh/id_rsa
kubectl create secret generic github --from-literal token="MY_TOKEN"  # replace with a real GitHub token
```

### Install Service Account and RBAC required for EventListener to work
See [ob-sa-rbac.yaml](ob-sa-rbac.yaml) in this directory.
```commandline
cd <path to local clone>/tekton
kubectl apply -f ob-sa-rbac.yaml
```
### Install the syntheis pipeline and the PR-commenting pipeline
See [netpol-synthesis-pl.yaml](netpol-synthesis-pl.yaml) and [netpol-pr-comments-pl.yaml](netpol-pr-comments-pl.yaml) in this directory
```commandline
kubectl apply -f netpol-synthesis-pl.yaml
kubectl apply -f netpol-pr-comments-pl.yaml
```

### Install the machinery for triggering the PR-commenting pipeline on PR events
See [netpol-pr-comments-tt.yaml](netpol-pr-comments-tt.yaml), [netpol-pr-comments-el.yaml](netpol-pr-comments-el.yaml) and [netpol-pr-comments-tb.yaml](netpol-pr-comments-tb.yaml) in this directory.
```commandline
kubectl apply -f netpol-pr-comments-tt.yaml
kubectl apply -f netpol-pr-comments-el.yaml
kubectl apply -f netpol-pr-comments-tb.yaml
```

### Expose the service behind the EventListener
Follow [these instructions](https://tekton.dev/docs/triggers/eventlisteners/#exposing-an-eventlistener-outside-of-the-cluster) to expose the EventListener you just created outside of the cluster.

### Create a webhook in the GitHub repo
Follow [these instructions](https://docs.github.com/en/developers/webhooks-and-events/webhooks/creating-webhooks) to create a new webhook in your mirror repository. Set `Payload URL` to the URL you exposed in the previous step, set `Content type` to `application/json`, then choose `Pull requests` as the event that triggers the webhook.

### Run the Synthesis pipeline
Modify the value of the `repo-full-name` param in `netpol-synthesis-plr.yaml` to reflect the path to your mirror repository. Then run
```commandline
kubectl create -f netpol-synthesis-plr.yaml
```
This should run the `netpol-synthesis` pipeline to create a new pull request with the synthesized NetworkPolicies in your GitHub mirror repository. Opening the pull request should trigger the PR-commenting pipeline which adds connectivity reports to the PR as comments.
