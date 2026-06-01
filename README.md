# Learning-GCP
Knowledge repository for learning GCP 

# Disk Creation in GCP

## Creating extra disk and attaching it to a compute instance

* Follow the steps and create a new disk with the desired storage capacity. \
{NOTE: Make sure to select the same zone where the instance is present.} This is done to ensure no issues in attaching disk.
* Edit the settings of the instance and attach existing disk.
* Now the disk is attached physically but logically it must be properly formatted and mounted on the instance.

The following commands are to be used after attaching disk.

```bash
lsblk  #To check the file system
sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0 discard /dev/sdb
sudo mkdir -p /mnt/data
sudo mount -o discard,defaults /dev/sdb /mnt/data
df -h
sudo resize2fs /dev/sdb
```


# Startup script for a managed instance group
```bash
#!/bin/bash
apt-get update
apt-get install -y apache2
systemctl start apache2
systemctl enable apache2
HOSTNAME=$(hostname)
echo "<html><body><h1>Hello from $HOSTNAME</h1><p>This page is served by VM: $HOSTNAME</p></body></html>" |tee /var/www/html/index.html
```
# Google Kubernetes Engine

**Kubectl**: It is the official command line tool and it communicates with the Cluster's Control Plane Server.

**Troubleshooting Commands**: 
* When **kubectl get pods** shows an error, use **kubectl describe pod [pod-name]**.
* You connect **kubectl** to a GKE Cluster by running the below command.
```bash
gcloud container clusters get-credentials --location[LOCATION]
```

* A __container__ is a standard, shippable package bundling code and all the dependencies required to run the application.

## Building a simple python web application

We will build a simple python application using the flask framework. Below given is the code of the same.

```bash
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, World! This is my first GKE app"

if __name__ == "__main__":
    app.run(host='0.0.0.0',port=int(os.environ.get('PORT',8080)))
```
Then the requirements.txt. This file is simple as we only have to install the Flask dependency for this application.

**requirements.txt**
```bash
flask
```

## Dockerfile

Below given is a sample dockerfile to build a simple Python application.
 ```bash
 FROM python:3.11-slim

 # Set the working directory in the container to /app
 WORKDIR /app

 # Copy the requirements file first to leverage Docker cache
 COPY requirements.txt .

 # Install any needed packages specified in the requirements.txt file
 RUN pip install -r requirements.txt

 # Copy the rest of the application's source code 
 COPY . .

 # Run app.py when the container launches
 CMD ["python","app.py"]
 ```

The above three files namely **app.py**, **requirements.txt** and *Dockerfile** are required to successfully run the web application.
Now we have to containerize and package the above three files. This is done using the below command.

```bash
docker build . -t my-app:v1
```

# Google Artifact Repository
It is a service in GCP used to store, manage and securely build artifacts like container images and packages.

## Assignment of roles for least privilege
* __Artifact Registry Reader__: Used in GKE, Cloud Run. The purpose is to pull images from the repository.
* __Artifact Registry Writer__: Used in Cloud Build and by developers creating images. The purpose is to push images to the repository after creation.
* __Artifact Registry Admin__: Used by DevOps Admins. The purpose of this role is to manage the repositories.

## Commands for creating repository

* Creation of repo requires location of region and repo format.
```bash
# gcloud artifacts repositories create [REPO-NAME] --repository-format=[any containerization platform] --location=us-central1 --description="My first container image repo"

gcloud artifacts repositories create my-app-repo --repository-format=docker --location=us-central1 --description="My first container image repo"
```
* Then use the below command to configure Docker so it can authenticate with Google Artifact Registry
```bash
gcloud auth configure-docker us-central1-docker.pkg.dev
```
* Then use the below command to tag a local Docker image with a new name that points to a repository present in Google Artifact Registry.
```bash
docker tag my-app:v1 us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-app-repo/my-app:v1
```
* Then use the below command to push the locally built image to the repository created in the Google Artifact Registry.
```bash
docker push us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-app-repo/my-app:v1
```