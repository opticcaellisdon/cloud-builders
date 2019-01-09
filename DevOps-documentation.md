# DevOps Platform 

#### CICD Pipeline steps

The pipeline contains the following steps:
    - **Checkout & Build**: Checking out the source repository and test the code compilation. The step runs the command `gradle clean assemble`. 
    - **Unit & Integration tests**: Testing the code for bugs. The step runs the command `gradle test`. 
    - **Sonarqube tests**: Scanning the code quality and analyzing bad patterns. The sonar server is running on top of AZURE cluster. This step is running concurrently with the Unit tests step. 
    - **Image build**: Building the container image after the successful tests. This step cannot be running if one of the two previous steps has not been executed successfully. 
    - **Image Push**: Pushing the container image to the Google Container Registry if the build was triggered by a push to develop or master branch. 
    - **Deploy to IST**: This step has a branch conditionals. If the build was triggered by a push to develop, the container image with the tag `latest` will be deployed on top of GKE cluster (IST). If the build was triggered by a push to master, the container image recently built and pushed to the registry will be deployed on IST cluster. 
    
**Note:** 
    - The Vulnerability tests step will be added after deciding the tool to implement. 
    - We only deploy to the GKE cluster running on the same project as Cloud Build and presenting the IST environment. 
    
#### Sonarqube Scanning

Google Cloud Build executes a build as a series of build steps. Each build step is running in a Docker container. So, in order to use a community-contributed Docker image as a build step, you need to download its source code and build the image.

##### Sonarqube Builder

The **Sonarqube tests** step is using a Sonarqube builder provided by GCP. This builder allows you to run static code analysis using Sonarqube on your code. Thus, to use sonarqube as a build step, you have to download its source code from this link: https://github.com/GoogleCloudPlatform/cloud-builders-community/tree/master/sonarqube and run the following command: 

```sh
gcloud builds submit . --config=cloudbuild.yaml
```    
This command build the Docker image of sonarqube and you can find it in Google Container Registry.

##### Using the build step with Cloud Build build 

Once you have built the Docker image of Sonarqube, you can use it as a build step in the Cloud Build pipeline. 
Here are some of the args of **Sonarqube tests** step:
    - `-Dsonar.host.url`: It mentions the url to the sonarqube server running on top of AZURE.
    - `-Dsonar.login`: It is the token to log in with. 
    - `-Dsonar.projectKey`: It is the key of a new project created for this purpose: `CompContact GCP`.  
    
A keystore file was created for the SSL certificate of sonar to allow the connection and avoid the ssl handshake exception.

#### kubectl Builder

For the deployment step, we need to run `kubectl` command on top of GKE cluster. Thus, kubectl builder is required for this step.  
In order to use this builder, you need to download its source code from this link: https://github.com/GoogleCloudPlatform/cloud-builders/tree/master/kubectl and run the following command:  

```sh
gcloud builds submit . --config=cloudbuild.yaml
``` 
This command build the Docker image of kubectl builder and you can find it in Google Container Registry.

##### Using the  kubectl builder step with Google Kubernetes Engine

To use this builder (kubectl docker image) in the pipeline, it is mendatory to grant the sufficient IAM permissions to the builder service account (cloud Build).

Running the following command will give Cloud Build Service Account `container.developer` role access to the Kubernetes Engine cluster, which is sufficient to be able to deploy container images on a GKE cluster:

```sh
PROJECT="$(gcloud projects describe \
    $(gcloud config get-value core/project -q) --format='get(projectNumber)')"

gcloud projects add-iam-policy-binding $PROJECT \
    --member=serviceAccount:$PROJECT@cloudbuild.gserviceaccount.com \
    --role=roles/container.developer
```

kubectl will need to be configured to point to a specific GKE cluster. You can configure the cluster by setting environment variables:
    - `CLOUDSDK_COMPUTE_ZONE`: The cluster's zone.
    - `CLOUDSDK_CONTAINER_CLUSTER`: The cluster's name. 
    
Setting the environment variables above will cause this step's entrypoint to first run a command to fetch cluster credentials as follows:

```sh
gcloud container clusters get-credentials --zone "$CLOUDSDK_COMPUTE_ZONE" "$CLOUDSDK_CONTAINER_CLUSTER"
```
Then, **kubectl** will have the configuration needed to talk to the GKE cluster.

#### Running builds using Google Cloud Build GitHub app

Cloud Build provides a **Google Cloud Build GitHub app**, which automatically builds your code each time you create a new branch or push code to GitHub. Here are the steps that have been followed to be able to use this application:

##### Installing the Google Cloud Build app

During the installation and set up process of the app, we specify the repository where to install the app. Also, we give an authorization to connect to Google Cloud Platform. After authorizing, we select the GCP project and billing account. GCP will associate the specified GCP project with GitHub. 

##### Building using the Google Cloud Build app

After the installation, it is possible to launch the build by committing changes to the code. This push event will make the GitHub app launch the cloud build pipeline. 

#### Enabling required reviews for pull requests

A Branch protection rule was created for each of the main branches `develop` and `master`. This rule requires 1 approving review and a successful status check before merging. The status check will be executed by the **Google Cloud Build app**. Thus, after a PR creation, you can go to the **Checks** tab and see the build result. By clicking on **View more details on Google Cloud Build**, the **Build details** page in GCP Console opens where you can see build information such as status, logs, and build steps. 

**Note:** The Pull Request creation event cannot trigger the build steps. 

#### Slack notifications

In order to set up the Slack notifications, the following steps have been followed: 
    - Creating a new Slack app: A new Slack app was created on top of an internal Slack Channel. 
    - Enabling incoming webhooks: A webhook was created for the new Slack app and it is pointing to the internal Slack channel. 
    - Writing the Cloud Function: A cloud function was created to listen to the Cloud Build topic and send a message to the slack channel whenever the build status is changed. You will find the Cloud Function files under **slack-notifications** Cloud Storage bucket.
    - Deploying the Cloud Function: The Cloud Function was deployed with Cloud Build as a trigger to its execution by running the following command: 
    
```sh
gcloud functions deploy subscribe --stage-bucket BUCKET_NAME --trigger-topic cloud-builds
```
After the deployment of the Cloud Function, whenever a build event occurs, we receive a Slack notification. 
