# secure-pipeline

## Setup process

## Install operators

Install the OpenShift Pipelines and OpenShift GitOps pipelines.

### Create initial namespaces

`oc new-project secure-pipeline`

`oc new-project secure-pipeline-ci`


### Get the ACS and the ArgoCD Credentials

*Use the script below to get the ACS credentials.*

`oc -n rhacs-operator get secret central-htpasswd -o go-template='{{index .data "password" | base64decode}}'`

`echo ""`

`oc get route -n rhacs-operator central -o jsonpath='{"https://"}{.spec.host}{"\n"}'`

*Use the script below to get the ArgoCD credentials.*

`oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-`

`echo ""`

`oc get route/openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'`

### ACS secret

Get ACS CI integration token and the ACS web UI address to create the acs-secret.

Within ACS, go to Platform configurations -> Integrations -> Authentication tokens.
Generate a new CI/CD Scoped token.

Execute the following command :


`oc create secret generic acs-secret \`
`--from-literal=acs_api_token=<token from above step> \`
`--from-literal=acs_central_endpoint=<url-for-rhacs-server>:443`


### Pull images

Patch the image registry to make it accessible externally.

`oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge`

Pull container images from Red Hat Registries to accelerate access during pipeline execution.

`oc import-image rhel9-nodejs-16 --from=registry.redhat.io/rhel9/nodejs-16 --confirm`

`oc import-image terminal --from=quay.io/marrober/devex-terminal-4:full-terminal-1.4 --confirm`

`oc import-image buildah --from=registry.redhat.io/rhel8/buildah:latest --confirm`

`oc import-image ubi --from=registry.access.redhat.com/ubi8/ubi:latest --confirm`

## Enable ACS to read from the OCP image registry

### Get the secret

Get the Builder service account authentication token from the secret.

`oc get secret/$(oc get sa/builder -o jsonpath='{.secrets[0].name}') -o jsonpath='{.metadata.annotations}' | jq '."openshift.io/token-secret.value"' `

Extract the field : openshift.io/token-secret.value into the copy-paste buffer.

### Create the integration in ACS

In ACS go to Platform configurations -> Integrations -> Image integration -> Generic Docker Registry and press the ‘Create integration’ button.

If the ACS cluster is to be accessed for CI/CD vulnerability and resource testing from the cluster on which ACS is installed then the Endpoint is : 

`https://image-registry.openshift-image-registry.svc:5000`

If the ACS cluster is to be accessed for CI/CD purposes from another cluster then the Endpoint is : 

`https://default-route-openshift-image-registry.apps.<cluster-name>.demolab.local`

Fill in the details as :

	Integration name : OCP Registry

	Endpoint : `https://image-registry.openshift-image-registry.svc:5000`

	Username : serviceaccount

	Password : <as copied from the secret previously>

	Check the option : Disable TLS certificate validation (insecure)

Test the integration and save if successful.


## GitHub integration

Github integration
To allow pipeline operations to push to github required the following : 

In Github create an access token :
Go to settings in the top right menu Github (after logging in).
Scroll down on the left menu to developer settings.
Select personal access tokens
Create a personal access token with the following scope :
Repo - all settings
Admin:org - read:org
Save the token and paste it into the command below :
Execute the command : 

`oc create secret generic github-access-token --from-literal=token=<token>`

## Quay.io Access

Generate a quay authentication secret using the menu within quay.io : 

Select username (top right)
Select generate encrypted password 
Enter the current password
Select Kubernetes secret as the type of authentication.
Download the yaml file.
Change the name of the yaml object in the file to be quay-auth-secret
The file should appear like :

apiVersion: v1
kind: Secret
metadata:
  name: quay-auth-secret
data:
  .dockerconfigjson: <secret-info>n0=
type: kubernetes.io/dockerconfigjson

Create the object using the command : 

`oc create -f <filename.yaml>`

## Add GPG key for source code commit verification

Add GPG key to a secret for CI verification of source code commits

`gpg --armor --export > public.key`

`oc create secret generic gpg-public-key --from-file=public.key`

`rm public.key`

### Get the secret key using the passphrase

	`gpg -o my-key.asc --export-secret-key --pinentry-mode=loopback --passphrase  <passphrase> <signature-id>`

	Create the secret to hold the key :

	`oc create secret generic gpg-secret-key --from-file=my-key.asc`

	Create a secret to hold the passphrase :

	`oc create secret generic gpg-secret-key-passphrase --from-literal=passphrase=<passphrase>`

If it is necessary to generate a signature first use the command `gpg --full-generate-key`

If it is necessary to delete the signature use the command :

`gpg --list-signatures` to get the full identifier, then use (example) :

`gpg --delete-secret-keys ABFC49A27C4C25E03689DF750F96D12F5EF84BB6`

`gpg --delete-key ABFC49A27C4C25E03689DF750F96D12F5EF84BB6`


## Cosign for signing container images

### Create the signature

Create a signing key using cosign

`COSIGN_PASSWORD=openshift cosign generate-key-pair k8s://openshift-pipelines/signing-secrets`


### Create the integration in ACS

Import the policy from here : ACS/image-signature-validation-policy.json

Create a new signature integration in ACS to validate the cosign public key. Add the public key generated by the above command, to the integration.
Enable the ACS policy.

Enable the policy and select the cosign signature to tell the policy what to use.

## Create ArgoCD project and projects

From the root of the cloned secure-pipeline repository execute the command :

`oc apply -k argocd-app`

### Validate the ArgoCD projects

From the ArgoCD web UI validate that the project and applications have been created :

* secure-pipeline-mgmt-app
* secure-pipeline-ci-base-image
* secure-pipeline-ci-app
* secure-pipeline-app-dev

## Test the pipeline build

From the root of the cloned secure-pipeline repository execute the command :

`oc create -f ci/application-pipeline/02-pipelineRun/pipelineRun.yaml`

## Update the SMEE process for DemoLab / RHPDS

If SMEE is used to connect triggered processes then ensure that the URL in the SMEE deployment apps is correct for the current cluster.

Check the files : ci/application-pipeline/05-smee/deployment.yaml

## Update the Cron job for the base image rebuild

Get the route to the trigger for the base image rebuild process :

`oc get route/base-image-github-ci-listener -o jsonpath='{"http://"}{.spec.host}{"/\n"}'`

Add this to the curl call in the file ci/base-image-pipeline/06-cronjobs/base-image-check-and-rebuild.yaml

## Testing

### Application build

Test the application build usng the command :

`oc create -f ci/application-pipeline/02-pipelineRun/pipelineRun.yaml`

Test the application build via a trigger by making a trivial change to the source code and committing / pusing to GitHub.

### Base image build

Test the base image build process using the command :

`oc create -f ci/base-image-pipeline/02-pipelinerun/build-base-image-pipelineRun.yaml`

Test the triggered build of the base image using the command :

`curl -k -s -d "{}" http://base-image-github-ci-listener-secure-pipeline-ci.apps.skylake.demolab.local/`



