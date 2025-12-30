##Task 1: Prework – Set up environment, enable APIs and create clusters

###Set up environment variables

Set up environment variables for your Project ID (this is important as it is used in several configuration files below):
'''Bash

export PROJECT_ID=$(gcloud config get-value project)

export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

export REGION=us-central1

gcloud config set compute/region $REGION
 
Enable required Google Cloud services
Copy code
Bash
gcloud services enable \
container.googleapis.com \
clouddeploy.googleapis.com \
artifactregistry.googleapis.com \
cloudbuild.googleapis.com
Enable required permissions for Kubernetes and Cloud Deploy
Copy code
Bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
--role="roles/clouddeploy.jobRunner"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
--role="roles/container.developer"
Create a Cloud Storage bucket for Cloud Build
Copy code
Bash
gsutil mb -p $PROJECT_ID gs://${PROJECT_ID}_cloudbuild
Create an Artifact Registry repository
Create a repository named cicd-challenge to store Docker images:
Copy code
Bash
gcloud artifacts repositories create cicd-challenge \
--description="Image registry for tutorial web app" \
--repository-format=docker \
--location=$REGION
Create Google Kubernetes Engine (GKE) clusters
Create two single-zone, single-node clusters for Staging and Production:
Copy code
Bash
gcloud container clusters create cd-staging \
--node-locations=us-central1-c \
--num-nodes=1 \
--async

gcloud container clusters create cd-production \
--node-locations=us-central1-c \
--num-nodes=1 \
--async
Task 2: Build the images and upload to the repository
Clone the lab repository
Copy code
Bash
cd ~/
git clone https://github.com/GoogleCloudPlatform/cloud-deploy-tutorials.git
cd cloud-deploy-tutorials
git checkout c3cae80 --quiet
cd tutorials/base
Create the skaffold.yaml configuration
Copy code
Bash
envsubst < clouddeploy-config/skaffold.yaml.template > web/skaffold.yaml
cat web/skaffold.yaml
The web directory now contains the skaffold.yaml file, which instructse Skaffold to build the container image.
Build the application image and push to Artifact Registry
Use the full Artifact Registry path created earlier:
Copy code
Bash
cd web
skaffold build --interactive=false \
--default-repo $REGION-docker.pkg.dev/$PROJECT_ID/cicd-challenge \
--file-output artifacts.json
cd ..
Create the Delivery Pipeline
Run the following commands to copy the pipeline template file:

Create the delivery-pipeline resource using the delivery-pipeline.yaml file:
cp clouddeploy-config/delivery-pipeline.yaml.template clouddeploy-config/delivery-pipeline.yaml
sed -i "s/targetId: staging/targetId: cd-staging/" clouddeploy-config/delivery-pipeline.yaml
sed -i "s/targetId: prod/targetId: cd-production/" clouddeploy-config/delivery-pipeline.yaml
sed -i "/targetId: test/d" clouddeploy-config/delivery-pipeline.yaml
Copied!
Set the deployment region using the deploy/region configuration parameter.
Apply the pipeline configuration you created above using the gcloud beta deploy command
Verify the delivery pipeline was created using the command below:
gcloud beta deploy delivery-pipelines describe web-app
Copied!
Configure the deployment targets
Two delivery pipeline targets will be created - one for each of the GKE clusters.

Ensure that the clusters are ready
The two GKE clusters should now be running but it's useful to verify this.

Get the status of the clusters:
gcloud container clusters list --format="csv(name,status)"
Copied!
All clusters should be in the RUNNING state, as indicated in the output below. If they are not yet marked as RUNNING, retry the command above until their status has changed to RUNNING.

Create a context for each cluster
Use the commands below to get the credentials for each cluster and create an easy-to-use kubectl context for referencing the clusters later:
CONTEXTS=({INSERT YOUR TARGETS HERE})
for CONTEXT in ${CONTEXTS[@]}
do
    gcloud container clusters get-credentials ${CONTEXT} --region ${REGION}
    kubectl config rename-context gke_${PROJECT_ID}_${REGION}_${CONTEXT} ${CONTEXT}
done
Copied!
Create a namespace in each cluster
Use the commands below to create a Kubernetes namespace (web-app) in each of the clusters:
for CONTEXT in ${CONTEXTS[@]}
do
    kubectl --context ${CONTEXT} apply -f kubernetes-config/web-app-namespace.yaml
done
Copied!
Create the delivery pipeline targets
Create a target definition file for each of the targets using the commands below (no changes needed):
envsubst < clouddeploy-config/target-staging.yaml.template > clouddeploy-config/target-cd-staging.yaml
envsubst < clouddeploy-config/target-prod.yaml.template > clouddeploy-config/target-cd-production.yaml

sed -i "s/staging/cd-staging/" clouddeploy-config/target-cd-staging.yaml
sed -i "s/prod/cd-production/" clouddeploy-config/target-cd-production.yaml
Copied!
Apply the target files to Cloud Deploy.
The targets are described in a yaml file. Each target configures the relevant cluster information for the target.

Display the details for the staging target:

cat clouddeploy-config/target-cd-staging.yaml
Copied!
Verify that the Cloud Deploy targets have been created.
Task 4. Create a Release
Create a release using the gcloud beta deploy releases command and the skaffold and artifacts.json files you created earlier.
Name the release web-app-001 and use the delivery-pipeline web-app.
HINT: your source directory should be web/

Verify that your application has been deployed to the staging environment (cd-staging) via the command below or in the console.
gcloud beta deploy rollouts list \
--delivery-pipeline web-app \
--release web-app-001
Copied!
Cloud Deploy Pipeline

Verify the release to the Staging environment
Task 5. Promote your application to production
Promote your application from the Staging (cd-staging) environment to the Production (cd-production) environment.
HINT: Don't forget to approve the deployment!

Verify the release to the Production environment
Task 6. Make a change to the application and redeploy it
Using the editor, open the cloud-deploy-tutorials/tutorials/base/web/leeroy-app/ directory and modify the app.go file. Change line 24 to say: fmt.Fprintf(w, "leeroooooy app v2!!\n")
Build the application and push to the Artifact Registry.
Create a new release on your pipeline you created earlier. Name the release web-app-002
Verify the new version has been deployed to the staging environment.
gcloud beta deploy rollouts list \
--delivery-pipeline web-app \
--release web-app-002
Copied!
Task 7. Rollback The Change
Oh No! Your QA Engineers have found a bug in your release to staging so you will need to rollback to the previous version.

Use Cloud Deploy to rollback to the original version of the application - web-app-001
Verify that the original version is running.
