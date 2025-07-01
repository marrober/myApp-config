# Setup

Ensure all solutions are installed and admin passwords are available.

## Create initial ArgoCD content to create the OCP projects

````bash
cd ~/data/git-repos/ci-cd/myApp-config
oc create -k argocd
````

Create a secret for access to the ACS CI/CD process
Select the myapp-ci project

````bash
oc project myapp-ci
````

Generate the CI/CD token inside ACS. Go to Platform configurations -> Integrations -> Authentication tokens.
Generate a new CI/CD Scoped token.
Execute the following command :

````bash
oc create secret generic acs-secret \
--from-literal=acs_api_token=<token from above step> \
--from-literal=acs_central_endpoint=$(oc get route/central -n stackrox -o jsonpath='{.spec.host}{":443"}')
````

## Pull images to the local cluster 

All tasks expect to find the required container images for Tekton in the local registry.

Pull the required images locally using :

````bash
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
oc import-image rhel9-nodejs-16 --from=registry.redhat.io/rhel9/nodejs-16 --confirm
oc import-image terminal --from=quay.io/marrober/devex-terminal-4:full-terminal-1.5 --confirm
oc import-image buildah --from=registry.redhat.io/rhel8/buildah:latest --confirm
oc import-image ubi --from=registry.access.redhat.com/ubi8/ubi:latest --confirm
oc import-image kustomize --from=quay.io/marrober/kustomize:latest --confirm
oc import-image s2i-rhel-8 --from=registry.redhat.io/ocp-tools-43-tech-preview/source-to-image-rhel8 --confirm
oc import-image gitops-commit-status --from=quay.io/redhat-developer/gitops-commit-status:v0.0.2 --confirm
oc import-image gosmee --from=quay.io/marrober/gosmee:latest --confirm
oc import-image oc-cli --from=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:878b31040c88f3eb56ca2bd2d77fa29128dad732850dd3fe779037ec9643bf02 --confirm
````

## ArgoCD Configuration
### ArgoCD user management 

````bash
argocd login --insecure $(oc get route/argocd-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')
argocd account list
````

Add the following to the spec section for the object : 
Kind : ArgoCD 
Name : openshift-gitops
Project : openshift-gitops

````bash
oc get ArgoCD/openshift-gitops -n openshift-gitops -o yaml > argocd-openshift-gitops.yaml
````

Edit the file as per below.

 extraConfig:
   accounts.mark: 'apiKey, login'

Example below : 

apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
 name: argocd
spec:
  extraConfig:
   accounts.mark: 'apiKey, login'
 controller:
   env:
   - name: "ARGOCD_K8S_CLIENT_BURST"
     value: "350"
   - name: "ARGOCD_K8S_CLIENT_QPS"
     value: "350"
 sso:
   dex:
     openShiftOAuth: true
   provider: dex

````bash
oc apply -f  argocd-openshift-gitops.yaml
argocd account update-password --account mark --new-password r3dh4t1!
````

## ArgoCD Create role and assign permissions to application
### Create the role definition and create the permission policy on the role

````bash
argocd proj role create myapp myapp-sync
argocd proj role add-policy myapp myapp-sync --action 'sync' --permission allow --object myapp-cd-development
argocd proj role add-policy myapp myapp-sync --action 'sync' --permission allow --object myapp-cd-qa
argocd proj role create-token myapp myapp-sync
````

<Copy token>

echo -n “<token>” | base64


Or if the file is not managed by ArgoCD create the secret by hand with :

oc create secret generic -n myapp-ci argocd-env-secret --from-literal=ARGOCD_AUTH_TOKEN=<token>

OR

oc create secret generic -n myapp-ci argocd-env-secret --from-literal=ARGOCD_PASSWORD=<token>
To test locally
export ARGOCD_AUTH_TOKEN=<token>

argocd account can-i sync applications myapp/myapp-cd-development 
argocd account can-i sync applications myapp/myapp-cd-qa
argocd app sync myapp-cd-development
argocd app sync myapp-cd-qa

# ACS read the Openshift Image Registry

````bash
oc get sa/pipeline -o yaml
````

## Get SA secret for ACS access to registry

````bash
oc get secret/pipeline-dockercfg-<whatever> -o 'go-template={{index .data ".dockercfg"}}' | base64 -d | jq .  
````

Take the password section from the item with index : default-route-openshift-image-registry.apps.cluster-.....

Base64 decode the content and copy the resulting token

In ACS go to Platform configurations -> Integrations -> Image integration -> Generic Docker Registry and press the ‘Create integration’ button.
Fill in the details as :
	Integration name : OCP Registry
	Endpoint : https://default-route-openshift-image-registry.apps.cluster-.....
	Username : admin
	Password : token from above
	Check the option : Disable TLS certificate validation (insecure)
Test the integration and save if successful.

## Create a Github access token

oc create secret generic github-access-token --from-literal=token=<token>

## Quay access 

oc create -f <filename.yaml>

# Testing

cd ../

## Test base image management

oc create -f base-image/02-pipelinerun/build-base-image-pipelineRun.yaml

## Test the build and deploy process 

oc create -f main-ci/07-pipelinerun/ci-pipelineRun.yaml

