   <h1 align="center"><b>WEATHER APP</b></h3>

<p>It is a mobile application that provides information about a weather of the city you want.</p>

&nbsp;

## How it works:

![weather-app-diagram](https://user-images.githubusercontent.com/77735480/183253699-705b9c38-8a1b-4231-a46c-ab996e499c5e.png)

At a high level, 'Weather App' consists of a frontend that runs directly on a mobile device and a backend that provides with weather report. 

   - Frontend is a 'React Native' based app, there user can 'sign up', 'login' or 'skip login' to access the functionality of the app. To authenticate the user frontend connects to the 'Firebase Auth' service. After that user can enter the city name, which will be sent to the backend to get the weather report. In our case 'Frontend app' will run on the Android studio emulator.  

   - Backend app is written in 'Node JS'. When the 'city name' is received it sends request to the 'Google API' to get a geocode of the location. Furthermore, backend sends the received geocode to the 'OpenWeather API' and gets a weather report of that location from it. Backend app will run as a deployment in the Kubernetes cluster. 'Backend app' will have it's domain address to which the 'Frontend app' will send request to.

Moreover, backend returns the weather report to the frontend. Frontend then shows the information on the screen of the device. Additionally, Frontend app connects to the 'Firestore Cloud Database' to store the request details for further analytics.

   - 'Firestore Cloud Database' is a NoSQL document database to easily store, sync, and query data for mobile and web apps - at global scale.

   - 'Firebase Auth' provides methods to create and manage users that use their email addresses and passwords to sign in.

More details on general architecture in each of the `backend` and `frontend` folders.

## 📦 Backend Installation instructions

## How to run:

  1. Install **Node** latest version [*Run node -v if version shown so node properly installed*]
  
  2. Install all npm dependencies using below command:
     ```
     npm install
     ```
     
  3. Run node server using below command:
     ```
     npm start
     ```
     
  4. Here you go server is up at *http://localhost:3000/*
  
## ⚙️ How to test:

     *http://localhost:3000/weather?location={CITY_NAME}*
     
   ### For instance:
     
     *http://localhost:3000/weather?location=bishkek*


## 📦 Frontend Installation instructions

## How to run:

  1. Install **Node** latest version [*Run node -v if version shown so node properly installed*]
  
  2. Install all npm dependencies using below command:
     ```
     npm install
     ```
     
  3. Run the app on an android device using below command:
     ```
     npx react-native run-android
     ```
     The app was tested on android devices, but with an additional configuration it is possible to run it on ios emulator.

## Deploying in Kubernetes
When building a backend Docker image you should specify a **GOOGLE_API_KEY** and **WEATHER_FORECAST_KEY** as environment variables.

**GOOGLE_API_KEY** will contain key for the Google API that is used for geocoding.
**WEATHER_FORECAST_KEY** will contain key for the 'OpenWeather API'.

For the frontend app you don't need to create pipeline for it and you don't need to deploy it in kubernetes. Frontend app should be tested on Android Studio.


## Learn More
1. This app is written using React Native for Android & iOS, Check the getting started guide here: https://reactnative.dev/docs/environment-setup .

   <h1 align="center"><b>WEATHER APP BACKEND CI/CD</b></h3>

<p>Configure the backend of the Weather app to be automatically built and deployed via GitHub Actions using Helm charts, with distinct pipelines for development, staging, and production environments.</p>

## Prerequisites
Ensure the availability of the following prerequisites:
- Docker, Helm, and the AWS CLI, all installed and correctly configured on your system.
- Access privileges to AWS ECR and an Amazon EKS cluster.
- A GitHub account, with a forked version of the WeatherApp repository.
- Domain names configured according to the specified hostname structure for the application.

## Configuration Steps

- Set Up Continuous Integration

- Set up GitHub Actions in conjunction with AWS.

- Create an IAM role named GitHubActionsWeatherCICDrole

- Ensure it has necessary permissions to Build and Push images to ECR

- Create a repository for weather-backend with immutability enabled

- Using Trust policy, be sure to limit access specifically to your organization (YOUR_USERNAME) and relevant repositories. Refer to Configuring OpenID Connect in Amazon Web Services - GitHub Docs 

- Create dev, staging, and prod environment in GitHub Repository settings

- Create weather-app namespace for app to be deployed in.

- Configure Environment level variable IAM_ROLE in GitHub Repository to store the ARN of the IAM role created.

#### Workflow Example
```yaml
name: Weather app CICD

on:
  push:
    branches:
      - feature/**
      - staging
  # pull_request:
  #   branches: [ "main" ]

# required permissions by GitHub Actions for assuming AWS IAM role
permissions:
  id-token: write # This is required for requesting the JWT
  contents: read # This is required for actions/checkout
env:
  AWS_REGION: "us-east-1"
  ENVIRONMENT_STAGE: ${{ (github.ref == 'refs/heads/main' && 'production') || (github.ref == 'refs/heads/staging' && 'staging') || 'dev' }}
  IAM_ROLE: "arn:aws:iam::905418100201:role/GitHubActionsWeatherCICDrole" #example IAM role
  IMAGE_TAG: ${{ github.sha }}
jobs:
  app-build-and-deploy:
    environment: ${{ (github.ref == 'refs/heads/main' && 'production') || (github.ref == 'refs/heads/staging' && 'staging') || 'dev' }}
    runs-on: ubuntu-latest
    steps:
      # checkout code
      - name: Checkout Branch
        uses: actions/checkout@v3

      # login to AWS
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.IAM_ROLE }}
          role-session-name: GitHubActionsWeatherCICDrole
          aws-region: ${{ env.AWS_REGION }}

      # Login to ECR of the current account
      - name: Login to Amazon ECR
        id: login-ecr
        if: github.ref != 'refs/heads/main'
        uses: aws-actions/amazon-ecr-login@v2

      # Build and push into ECR
      - name: Backend - Build, tag, and push docker image to Amazon ECR
        if: github.ref != 'refs/heads/main'
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          REPOSITORY: weather-backend
        working-directory: ./backend
        run: |
          docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG

      - name: Login to EKS - update kubeconfig
        run: aws eks update-kubeconfig --name example-eks-cluster-${{ env.ENVIRONMENT_STAGE }} --region ${{ env.AWS_REGION }} #example

      - name: Backend - Deploy to Kubernetes
        run: |
          helm upgrade --install weather-app-backend ./helm-chart \
            --values ./helm-chart/values/backend/${{ env.ENVIRONMENT_STAGE }}-values.yaml \
            --namespace weather-app --create-namespace \
            --set image.tag="$IMAGE_TAG"
```        
For Detailed information on what each step does, please visit the Github actions docs.
Includes SHA image tagging for unique versions each push.

You also need to build a helm chart template app.yaml and a *-values.yaml file for it.  Need a dev-values.yaml, staging-values.yaml, and prod-values.yaml.  Should look variablized something like this:

#### app.yaml example
````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}
  namespace: {{ .Values.appNamespace }}
  labels:
    app: {{ .Values.appName }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appName }}
  template:
    metadata:
      labels:
        app: {{ .Values.appName }}
    spec:
      serviceAccountName: {{ .Values.serviceAccount }}
      containers:
      - name: {{ .Values.appName }}
        image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.port }}
        env:
        - name: GOOGLEAPIKEY # example names of secrets for api keys
          valueFrom:
            secretKeyRef:
              name: {{ .Values.awsSecrets.k8sSecretName }}
              key: GOOGLE_API_KEY
        - name: WEATHERFORECASTKEY  # example names of secrets for api keys
          valueFrom:
            secretKeyRef:
              name: {{ .Values.awsSecrets.k8sSecretName }}
              key: WEATHER_FORECAST_KEY
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store-inline
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: {{ .Values.awsSecrets.secretProviderClassName }}



---
apiVersion: v1
kind: Service
metadata:
  name: "{{ .Values.appName }}-svc"
  namespace: {{ .Values.appNamespace }}
spec:
  selector:
    app: {{ .Values.appName }}
  ports:
    - protocol: {{ .Values.serviceProtocol }}
      port: {{ .Values.servicePort }}
      targetPort: {{ .Values.port }}

---
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "{{ .Values.appName}}-ingress"
  namespace: {{ .Values.appNamespace }}
  annotations:
    kubernetes.io/ingress.class: nginx # creates DNS record automatically
spec:
  rules:
  - host: {{ .Values.hostName }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: "{{ .Values.appName }}-svc"
            port:
              number: {{ .Values.servicePort }}
  ingressClassName: nginx
  tls:
  - hosts:
      - {{ .Values.hostName }}
    secretName: {{ .Values.tlsSecret }}
{{- end }}
````

**Each Helm chart typically contains the following key components**:
- **Chart.yaml**: Contains metadata like the chart's version, name, and description.
- **values.yaml**: Houses default configuration values, which can be overridden by specific values files like `dev-values.yaml`, `staging-values.yaml`, and `prod-values.yaml`.
- **templates/**: Directory for Kubernetes resource templates that are populated with values from `values.yaml`.

#### dev-values.yaml Helm Chart EXAMPLE w/ Example values (will need to create your own names for values needed)
````yaml

replicaCount: 1
image:
  repo: "905418100201.dkr.ecr.us-east-1.amazonaws.com/weather-backend"
port: 3000
servicePort: 80
serviceProtocol: TCP
appName: weather-backend
hostName: yourhostname.com
ingress:
  enabled: true
appNamespace: weather-app
serviceAccount: weather-app-backend-sa
tlsSecret: yourdomain.com-tls
# create K8s secret from AWS Secrets Manager secret using SecretProviderClass
awsSecrets:
  enabled: true
  secretProviderClassName: "weather-backend-aws-secrets"
  k8sSecretName: "weather-api-credentials"
  awsSecretName: "dev/weather-app/api-keys"
  secretKeys:
    - key: "GOOGLE_API_KEY"
    - key: "WEATHER_FORECAST_KEY"
# Provide values for Application
appConfig:
  secrets:
    - name: GOOGLEAPIKEY
      secretName: "weather-api-credentials"
      key: GOOGLE_API_KEY
    - name: WEATHERFORECASTKEY
      secretName: "weather-api-credentials"
      key: WEATHER_FORECAST_KEY
````

This includes all integral parts for the backend app to make it work and secure with examples of variables in the values file to be passed to app.yaml template.

Including:
Necessary variables for keeping the API keys secret within AWS secret manager.
tlsSecret variable for TLS certificate secret.

### IAM Role and Permissions Configuration
For secure AWS service integration, create an IAM role named GitHubActionsCICDrole. This role facilitates GitHub Actions workflows to interact with AWS services like ECR and EKS by establishing a trust relationship using OpenID Connect (OIDC).

Setup for GitHub Actions:
OIDC Identity Provider: Create an OIDC identity provider in AWS IAM to represent GitHub Actions. Use https://token.actions.githubusercontent.com as the provider URL and sts.amazonaws.com as the audience.

IAM Role Creation: Create the GitHubActionsCICDrole with the OIDC provider as the trusted entity. Define a condition to ensure that only workflows from your specific GitHub repository can assume this role, enhancing security.

Attach Policies: Assign the necessary permissions to the role, allowing it to interact with ECR for Docker image storage and EKS for Kubernetes cluster operations. Ensure the policies adhere to the principle of least privilege.

Kubernetes Cluster Access: To grant the GitHubActionsCICDrole permissions within your EKS cluster, update the aws-auth ConfigMap as follows:

Retrieve the current aws-auth ConfigMap:

```bash
kubectl get configmap/aws-auth -n kube-system -o yaml > aws-auth.yaml
```
Edit the aws-auth.yaml file to include the GitHubActionsCICDrole. Add the following snippet under the mapRoles section, substituting <your-aws-account-id> with your actual AWS account ID:

```yaml
- rolearn: arn:aws:iam::<your-aws-account-id>:role/GitHubActionsWeatherCICDrole
  username: github-actions-weather
  groups:
    - system:masters
```
Apply the updated aws-auth ConfigMap to the cluster:
```bash
kubectl apply -f aws-auth.yaml
```
This setup ensures a secure bridge between GitHub Actions and AWS, enabling CI/CD workflows to deploy and manage your application seamlessly.


#### Ingress Setup

**Using an Existing Ingress Controller**: If your Kubernetes cluster already has an Ingress controller, like NGINX, you're ready to define Ingress resources to route external traffic to your services. If your environment lacks an Ingress controller, follow the [official installation guide](https://kubernetes.github.io/ingress-nginx/deploy/#aws) for AWS deployment.

**Defining Ingress Resources**: Define Ingress rules to route external HTTP(S) traffic to the correct services within your cluster. Use Helm charts or Kubernetes manifests for this purpose. Below is an example Ingress resource using NGINX as the Ingress controller, with annotations for dynamic DNS configuration. Look at the above YAML file for examples of how it should look.

**Certificate Management**: You can obtain TLS certificates from a Certificate Authority (CA) like Let's Encrypt, which offers free certificates suitable for securing web applications. Alternatively, if you have an existing wildcard certificate that matches your domain naming scheme, you can also use that.

**Incorporating TLS in Ingress**: Modify your Ingress resource to include TLS settings, referencing the Kubernetes Secret that stores your TLS certificate and private key:
Secure your application by configuring TLS, either by obtaining certificates from Let's Encrypt or using an existing wildcard certificate

```yaml
spec:
  tls:
  - hosts:
    - "<your-application-domain>"
    secretName: "<tls-secret-name>"
  rules:
  ...
```
Note: The secret name should refer to a Kubernetes Secret that contains your TLS certificate (tls.crt) and private key (tls.key). Ensure this Secret exists in the same namespace as your Ingress resource.

Follow this guide to incorporate secrets into kubernetes:
https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html

The secrets you need to incorporate are the API Keys for the backend of the weather app.

Once done and all proper code is incorporated mirroring the example yaml helm chart files provided above, you'll need to create a bootstrap.sh script that will import the secret environment variable into the app before it starts.  This is an example that will work:

``` s
#!/bin/bash

echo "GOOGLE_API_KEY = $GOOGLEAPIKEY" > /usr/src/app/.env
echo "WEATHER_FORECAST_KEY = $WEATHERFORECASTKEY" >> /usr/src/app/.env

exec npm start

```

Also will need to change the dockerfile to reflect bootstrap.sh script changes to happen before app starts. Should look like this

``` s

FROM node:16

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production

# Bundle app source and remove boostrap file from main directory
COPY . .
RUN rm /usr/src/app/bootstrap.sh

# Copy the bootsrap script
COPY bootstrap.sh /bin/bootstrap.sh

EXPOSE 3000

# Start app when container launces
# Insert env variables before app starts
CMD ["/bin/bootstrap.sh"]

```

Following all the steps above, should ensure a secure and working CICD for the backend of the weather app.