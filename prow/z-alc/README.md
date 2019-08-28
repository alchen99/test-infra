Setting up Prow
==========================================
For the most part we will follow the manual instructions from https://github.com/alchen99/test-infra/blob/master/prow/getting_started_deploy.md
*NOTE* the tackle and bazel utility has not worked for me

git clone https://github.com/alchen99/test-infra.git
git checkout ct-infra


Get GitHub Bot account
=========================================================
a) Login as GitHub bot user who is organization owner. Click on user profile. Go to settings. Go to developer settings.
   Go to Personal Access Tokens. Generate new token with repo and admin_org:hook scope.
   For the current setup we are using the token named prow


Set up GKE K8 cluster
=========================================================
gcloud auth login                                      # login as @informatica.com user
gcloud config set project $PROJECT_NAME
export PROJECT=$PROJECT_NAME
export ZONE=us-west1-b
export REGION=us-west1
export K8_NAME=prow

# set up region K8 cluster
# this will set up with 1 node in each region
gcloud container --project "${PROJECT}" clusters create $K8_NAME \
  --region ${REGION} --machine-type n1-standard-4 --num-nodes 1
  
# set up RBAC for K8 cluster
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin --user $(gcloud config get-value account)

# add other users to this role
kubectl edit clusterrolebinding cluster-admin-binding


Create GitHub Secrets in K8
=================================================================
hmac-token    # openssl rand -hex 20 > hmac-token.txt
oauth-token   # Should have already been created above

create file hmac-token.txt
create file oauth-token.txt

kubectl create secret generic hmac-token --from-file=hmac=./hmac-token.txt
kubectl create secret generic oauth-token --from-file=oauth=./oauth-token.txt


Add the prow components to the cluster
=================================================================
cd test-infra/prow/z-alc
kubectl apply -f ../cluster/starter.yaml


After a couple of minutes check the cluster components to make sure they are all running:
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
kubectl get deployments

NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deck               2         2         2            2           86s
hook               2         2         2            2           86s
horologium         1         1         1            1           85s
plank              1         1         1            1           86s
sinker             1         1         1            1           86s
statusreconciler   1         1         1            1           85s
tide               1         1         1            1           85s


Get ingress IP address
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
kubectl get ingress ing

NAME   HOSTS   ADDRESS         PORTS   AGE
ing    *       an.ip.addr.ess  80      2m37s

Go to http://an.ip.addr.ess/ and make sure that the "echo-test" job has a green check-mark next to it. 
At this point you have a prow cluster that is ready to start receiving GitHub events!


Add the webhook to GitHub
=================================================================
Configure github to send your prow instance application/json webhooks for specific repos and/or whole orgs.
It is not recommended to set this up for the whole org unless all repos are going to use Prow.

Go to the repository and click Settings -> Webhooks and add the webhook URL http://an.ip.addr.ess/hook.
A green check mark (for a ping event, if you click edit and view the details of the event) suggests everything is working!

You should either use the prow-test repository or create a new repository for testing this integration. If you use the
prow-test repository you should deactivate the old webhooks first


Enable some plugins by modifying the plugins.yaml file
=================================================================
Plugins are configured per repository. 

cd test-infra/prow/z-alc
# Replace all instances of alcOrg/prow-test with your repository name in the plugins.yaml
kubectl create configmap plugins --from-file=plugins.yaml=plugins.yaml --dry-run -o yaml \
  | kubectl replace configmap plugins -f -


Set up other configs in config.yaml
=================================================================
NOTE: A lot of things in this file is configured per repository

cd test-infra/prow/z-alc
# Replace all instances of alcOrg/prow-test with your repository name in the config.yaml
# Take note of the pod_namespace in this config.yaml and export this as POD_NAMESPACE environment variable
export POD_NAMESPACE=test-pods


Configure Cloud Storage
=================================================================
gcloud iam service-accounts create svc-prow
identifier="$(  gcloud iam service-accounts list --filter 'name:svc-prow' --format 'value(email)' )"
gsutil mb gs://alc-prow-artifacts/
gsutil iam ch "serviceAccount:${identifier}:objectAdmin" gs://alc-prow-artifacts
gcloud iam service-accounts keys create --iam-account "${identifier}" service-account.json
# add this credential to the $POD_NAMESPACE
kubectl -n $POD_NAMESPACE create secret generic gcs-credentials --from-file=service-account.json


Configure Plank in config.yaml
=================================================================
Before we can update plank's default_decoration_config we'll need to know the version we're using

PROW_VERSION=$(kubectl get pod -lapp=plank -o jsonpath='{.items[0].spec.containers[0].image}' | cut -d: -f2)

Then setup plank's default_decoration_config in config.yaml

plank:
  default_decoration_config:
    utility_images: # using the tag we identified above
      clonerefs: "gcr.io/k8s-prow/clonerefs:v20190823-9f991db2d"
      initupload: "gcr.io/k8s-prow/initupload:v20190823-9f991db2d"
      entrypoint: "gcr.io/k8s-prow/entrypoint:v20190823-9f991db2d"
      sidecar: "gcr.io/k8s-prow/sidecar:v20190823-9f991db2d"
    gcs_configuration:
      bucket: alc-prow-artifact # the bucket we just made
      path_strategy: explicit
    gcs_credentials_secret: gcs-credentials # the secret we just made

cd test-infra/prow/z-alc
kubectl create configmap config --from-file=config.yaml=config.yaml --dry-run -o yaml \
  | kubectl replace configmap config -f -


Configure branchprotector
===========================================================================================
https://github.com/alchen99/test-infra/tree/master/prow/cmd/branchprotector for more information

cd test-infra/prow/z-alc

# Edit config.yaml branch-protection section as needed
# Edit branchprotector-cron.yaml with $PROW_VERSION

kubectl apply -f branchprotector-cron.yaml    # Deploy branchprotector cron job



Configure label_sync
===========================================================================================
See https://github.com/alchen99/test-infra/tree/master/label_sync for more information

cd test-infra/prow/z-alc

# Edit label_sync/label_sync_cron_job.yaml with $PROW_VERSION

# create configmap
kubectl create configmap label-config --from-file=labels.yaml=label_sync/labels.yaml

# update configmap
kubectl create configmap label-config --from-file=labels.yaml=label_sync/labels.yaml --dry-run -o yaml \
  | kubectl replace configmap label-config -f -

kubectl apply -f label_sync/label_sync_cron_job.yaml            # deploy label_sync cron job



Setup approval2 webhook
===========================================================================================
git clone https://github.com/alchen99/approval-webhook

# create secret docker-registry-image-pull-secret to pull from ct-docker in Artifactory
kubectl create secret docker-registry docker-registry-image-pull-secret \
  --namespace=default \
  --docker-server=$PRIVATE_REGISTRY \
  --docker-username=$USERNAME \
  --docker-password=$API_KEY \
  --docker-email=$USEREMAIL

cd approval-webhook/k8s
kubectl apply -f approval2-deployment.yml
kubectl apply -f approval2-service.yml



Configure jobs by modifying/adding jobs to test-infra/prow/z-alc/jobs
===========================================================================================
See https://github.com/alchen99/test-infra/blob/master/prow/jobs.md

Per repo jobs should be configured in the z-alc/jobs/$REPO_NAME/*.yaml



Configure SSL
=================================================================
Use cert-manager for automatic LetsEncrypt integration. If you already have a cert then follow the official docs to set up HTTPS termination. Promote your ingress IP to static IP. On GKE, run:

gcloud compute addresses create [ADDRESS_NAME] --addresses [IP_ADDRESS] --region [REGION]
Point the DNS record for your domain to point at that ingress IP. The convention for naming is prow.org.io, but of course that's not a requirement.

Then, install cert-manager as described in its readme. You don't need to run it in a separate namespace
