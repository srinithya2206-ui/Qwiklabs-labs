# Qwiklabs-labsexport PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
export REGION=us-central1
gcloud config set compute/region $REGION
gcloud services enable \
container.googleapis.com \
clouddeploy.googleapis.com \
artifactregistry.googleapis.com \
cloudbuild.googleapis.com
Enable permissions for both Kubernetes and Cloud Deploy using the following commands:
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
--format="value(projectNumber)")-compute@developer.gserviceaccount.com \
--role="roles/clouddeploy.jobRunner"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member=serviceAccount:$(gcloud projects describe $PROJECT_ID \
--format="value(projectNumber)")-compute@developer.gserviceaccount.com \
--role="roles/container.developer"
Copied!
Create a Cloud Storage bucket for Cloud Build to store sources and logs.
gsutil mb -p $PROJECT_ID gs://${PROJECT_ID}_cloudbuild
Copied!
Create an Artifact Repository
Create a repository for storing your Docker containers.

Name the repository: cicd-challenge

gcloud artifacts repositories create cicd-challenge \
--description="Image registry for tutorial web app" \
--repository-format=docker \
--location=$REGION
Copied!
Create the Google Kubernetes Engine clusters.
Create two GKE clusters for Staging and Production named cd-staging and cd-production. Clusters should be single zone and single node.

gcloud container clusters create cd-staging --node-locations=us-central1-c --num-nodes=1 --async
gcloud container clusters create cd-production --node-locations=us-central1-c --num-nodes=1 --async
Copied!
Task 2. Build the images and upload to the repository
Clone the repository for the lab into your home directory using the commands below:
cd ~/
git clone https://github.com/GoogleCloudPlatform/cloud-deploy-tutorials.git
cd cloud-deploy-tutorials
git checkout c3cae80 --quiet
cd tutorials/base
Copied!
Create the skaffold.yaml configuration using the command below:
envsubst < clouddeploy-config/skaffold.yaml.template > web/skaffold.yaml
cat web/skaffold.yaml
Copied!
The web directory now contains the skaffold.yaml configuration file, which provides instructions for Skaffold to build a container image for your application.

Run the skaffold command to build the application and deploy the container image to the Artifact Registry repository previously created:
HINT: use the full path to the repository you created earlier

cd web
skaffold build --interactive=false \
--default-repo <INSERT YOUR ARTIFACT REPOSITORY HERE> \
--file-output artifacts.json
cd ..
