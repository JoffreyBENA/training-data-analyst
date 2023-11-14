# App Dev - Deploying the Application into Kubernetes Engine: Node.js

## Task 1. Preparing the case study application : 

In this section, you access Cloud Shell, clone the git repository containing the Quiz application, configure environment variables, and run the application.

### Clone source code in Cloud Shell

1. In Cloud Shell, clone the repository for the class:

git clone --depth=1 https://github.com/GoogleCloudPlatform/training-data-analyst

2. Create a soft link as a shortcut to your working directory:

ln -s ~/training-data-analyst/courses/developingapps/v1.3/nodejs/containerengine ~/containerengine

### Configure the case study application and review code

1. Change the working directory:

cd ~/containerengine/start

2. Configure the Quiz application:

. prepare_environment.sh

Note: This script file:
  - Creates a Google App Engine application.
  - Exports environment variables GCLOUD_PROJECT and GCLOUD_BUCKET.
  - Runs npm install.
  - Creates entities in Google Cloud Datastore.
  - Creates a Google Cloud Pub/Sub topic.
  - Creates a Cloud Spanner Instance, Database, and Table.
  - Prints out the Google Cloud Platform Project ID.

3. In Cloud Shell, click Open Editor.

4. Navigate to containerengine/start.

Note: The folder structure for the Quiz application changes to reflect how it is deployed in Kubernetes Engine.
The web application is in a folder called frontend.
The worker application code that subscribes to Cloud Pub/Sub and processes messages is in a folder called backend.
There are configuration files for Docker (a Dockerfile in the frontend and backend folder) and Kubernetes Engine (\*.yaml).

## Task 2. Creating a Kubernetes Engine cluster

In this section, you create a Kubernetes Engine cluster to host the Quiz application.

### Create a Kubernetes Engine cluster

1. In the Cloud Platform Console, on the Navigation menu, click APIs & Services.

2. Scroll down in the list of enabled APIs and confirm that both of these APIs are enabled:

  - Kubernetes Engine API
  - Container Registry API

If either API is missing, click + ENABLE APIS AND SERVICES at the top. Search for the above APIs by name and enable each for your current project. (You noted the name of your GCP project above.)

3. In the Cloud Platform Console, on the Navigation menu, click Kubernetes Engine.

4. Click Create.

5. Click Configure for GKE Standard mode.

6. Configure the cluster using the following table:
   
  - Name : quiz-cluster
  - Zone : us-central1-b
  - In the Node pools area, click default-pool : In the Security > Access scopes area, Select Allow full access to all Cloud APIs

7. Click Create.

Note: The cluster takes a couple of minutes to provision.

### Connect to the cluster

1. After the cluster is ready, click the three vertical dots on the right side and then click Connect.

2. In Connect to the cluster, copy the first command to the clipboard then click OK.

Note: The command will be in the form:
gcloud container clusters get-credentials quiz-cluster --zone us-central1-b --project <Project-ID>.

3. Paste the command into Cloud Shell and press ENTER.

4. List the pods in the cluster:

kubectl get pods

Note: The response should indicate that there are no pods in the cluster.
This confirms that you have configured security to allow the kubectl command-line tool to perform operations against the cluster.

## Task 3. Building Docker images using Cloud Build

In this section, you create a Dockerfile for the application frontend and backend and employ Cloud Build to build images and store them in the Container Registry.

### Create the Dockerfile for the frontend and backend

1. In the Cloud Shell code editor, open frontend/Dockerfile.

2. After the existing text, enter the Dockerfile commands to initialize the creation of a custom Docker image using Google's NodeJS App Engine image as the starting point.

Note: The image you are going to use is: gcr.io/google_appengine/nodejs.

3. Add the Dockerfile command to copy the contents from the current folder to a destination folder in the image /app/.

4. Add the Dockerfile command to execute npm install -g npm@8.1.3 as part of the build process to ensure that the container runs a compatible version of npm for the application.

5. Add the Dockerfile command to execute npm update.

6. Complete the Dockerfile by entering the statement, npm start, which executes when the container runs:

...frontend/Dockerfile
FROM gcr.io/google_appengine/nodejs
RUN /usr/local/bin/install_node '>=0.12.7'
COPY . /app/
RUN npm install -g npm@8.1.3 --unsafe-perm || \
  ((if [ -f npm-debug.log ]; then \
      cat npm-debug.log; \
    fi) && false)
RUN npm update    
CMD npm start

7. Save the Dockerfile.

8. Repeat the previous steps for the backend/Dockerfile file:

...backend/Dockerfile
FROM gcr.io/google_appengine/nodejs
RUN /usr/local/bin/install_node '>=0.12.7'
COPY . /app/
RUN npm install -g npm@8.1.3 --unsafe-perm || \
  ((if [ -f npm-debug.log ]; then \
      cat npm-debug.log; \
    fi) && false)
RUN npm update
CMD npm start

9. Save the second Dockerfile.

### Build Docker images with Cloud Build

1. In Cloud Shell, build the frontend Docker image:

gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-frontend ./frontend/

Note: The files are staged into Cloud Storage, and a Docker image is built and stored in the Container Registry. This takes a few minutes.

2. Build the backend Docker image:

gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-backend ./backend/

3. In the Cloud Platform Console, on the Navigation menu, click Container Registry.

Note: You should see two items: quiz-frontend and quiz-backend.

4. Click quiz-frontend.

Note: You should see the image name, tags (latest), and size (around 275 MB).

## Task 4. Creating Kubernetes Deployment and Service resources

In this section, you modify template yaml files that contain the specification for Kubernetes Deployment and Service resources, and then create the resources in the Kubernetes Engine cluster.

### Create a Kubernetes Deployment file

1. In the Cloud Shell code editor, open the frontend-deployment.yaml file.

Note: The file skeleton has been created for you. Your job is to replace placeholders with values specific to your project.

2. Replace the placeholders in the frontend-deployment.yaml file using the following values:

  - [GCLOUD_PROJECT] : GCP Project ID (Display the Project ID by entering echo $GCLOUD_PROJECT in Cloud Shell)
  - [GCLOUD_BUCKET] : Cloud Storage bucket name for the media bucket in your project (Display the bucket name by entering echo $GCLOUD_BUCKET in Cloud Shell)
  - [FRONTEND_IMAGE_IDENTIFIER] : The frontend image identified in the form gcr.io/[Project_ID]/quiz-frontend

Note: The quiz-frontend deployment provisions three replicas of the frontend Docker image in Kubernetes pods, which are distributed across the three nodes of the Kubernetes Engine cluster.

3. Save the file.

4. Replace the placeholders in the backend-deployment.yaml file using the following values:
Placeholder Name

  - [GCLOUD_PROJECT] : GCP Project ID (Display the Project ID by entering echo $GCLOUD_PROJECT in Cloud Shell)
  - [GCLOUD_BUCKET] : Cloud Storage bucket name for the media bucket in your project (Display the bucket name by entering echo $GCLOUD_BUCKET in Cloud Shell)
  - [FRONTEND_IMAGE_IDENTIFIER] : The frontend image identified in the form gcr.io/[Project_ID]/quiz-backend

Note: The quiz-backend deployment provisions two replicas of the backend Docker image in Kubernetes pods, which are distributed across two of the three nodes of the Kubernetes Engine cluster.

5. Save the file.

5. Review the contents of the frontend-service.yaml file.

Note: The service exposes the frontend deployment using a load balancer. The load balancer will send requests from clients to all three replicas of the frontend pod.

### Execute the Deployment and Service files

1. In Cloud Shell, provision the quiz frontend Deployment:

kubectl create -f ./frontend-deployment.yaml

2. Provision the quiz backend Deployment:

kubectl create -f ./backend-deployment.yaml

3. Provision the quiz frontend Service:

kubectl create -f ./frontend-service.yaml

Note: Each command provisions resources in Kubernetes Engine. This takes a few minutes to complete the process.

## Task 5. Testing the Quiz application

In this section you review the deployed Pods and Service and navigate to the Quiz application.

### Review the deployed resources

1. In the Cloud Console, on the Navigation menu, click Kubernetes Engine.

2. Click Kubernetes Engine > Workloads.

Note: You should see two items, quiz-frontend and quiz-backend.
You may see that the pod status is OK or in the process of being created. You may need to click refresh a couple times until you see OK.

3. Click quiz-frontend.

4. Scroll down to Managed pods.

Note: You should see that there are three quiz-frontend pods.

5. Click Kubernetes Engine > Services & Ingress.

Note: You may see that the quiz-frontend load balancer is being created or is OK.
Wait until the Service is OK before continuing.
You should see an IP address endpoint when the service is ready.

6. Under Endpoints, click the Service IP address.
Note: You should see the Quiz application.

7. Create a question or take a test.
Note: The application works as expected!

Note: Review
Which Docker command is used to execute a command when the container is being constructed?
FROM
COPY
RUN
CMD
Which Docker command is used to execute a command when the container has been deployed?
FROM
COPY
RUN
CMD
Which Kubernetes command is used to retrieve the list of pods running on a cluster?
kubectl pods list
kubectl deployments list
kubectl get pods
kubectl get deployments
