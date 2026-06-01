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

* We can also update the image by changing the source code of the app.py file. It is mentioned below.
  ```bash
    from flask import Flask
    import os

    app = Flask(__name__)

    @app.route('/')
    def hello():
        return """
        <!DOCTYPE html>
        <html>
        <head>
            <title>My GKE App</title>
            <style>
                body {
                    margin: 0;
                    height: 100vh;
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    background-color: #0b3d91; /* dark blue */
                    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
                }
                .box {
                    background-color: white;
                    padding: 40px;
                    border-radius: 12px;
                    box-shadow: 0px 4px 20px rgba(0,0,0,0.3);
                    text-align: center;
                    color: #0b3d91;
                    font-size: 24px;
                    font-weight: 600;
                }
            </style>
        </head>
        <body>
            <div class="box">
                Hello world! This is my first GKE app
            </div>
        </body>
        </html>
        """

    if __name__ == "__main__":
        app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
    ```
* Build the image again and give out a new version number this time around to create the new edits.
  ```bash
  docker build . -t my-app:v2
  ```
* Also tag the image appropriately according the version number to correctly push it to the repository.
  ```bash
  docker tag my-app:v2 us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-app-repo/my-app:v2
  ```

**Kubernetes** is used to deploy the application using the created image and expose it to different levels.
Below are some of the commands used to deploy the services using K8s.
* The below command is used to create a Kubernetes deployment.
    ```bash
    kubectl create deployment hello-app --image=us-central1-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-app-repo/my-app:v1
    ```
* There are three types of service types used to deploy services in Kubernetes.\
 1] **ClusterIP**: This is the default selection. Gives the service a stable IP inside the Kubernetes cluster. It is perfect for internal backend services like a database etc.\
 2] **NodePort**: Exposes the service on a static port on each node's IP. It is mostly used for debugging, and not in production.\
 3] **LoadBalancer**: Exposes the service to the public internet.

* The below command exposes the deployment to the public internet using the LoadBalancer service type.
    ```bash
    kubectl expose deployment hello-app --type=LoadBalancer --port=80 --target-port=8080
    ```
* The below commands are useful to check the details of the deployed services.
    ```bash
    kubectl get svc # gets info of all the services
    kubectl get services hello-app # gets info of any particular service
    ```

* There is an alternative way to push images into repositories using DockerHub.\
 DockerHub is a global open source service that contains repositories hosting images. It is accessible using all different cloud providers.\
 Firstly one has to create an account on DockerHub and have note of the credentials which will be later used to authenticate.\
 Below given are the commands of the same.
    ```bash
    docker build . -t sh1reesh2003/my-app:v2

    docker login [enter required credentials]

    docker push sh1reesh2003/my-app:v2

    kubectl create deployment my-app --image=sh1reesh2003/my-app:v2

    kubectl expose deployment my-app --type=LoadBalancer --port=80 --target-port=8080
    ```

 * If there are some changes to be made in the build config. Make the changes, save them and rebuild the docker image with a new version.\
   Then use the below command to change the image.
   ```bash
   kubectl set image deployment/hello-app my-app=sh1reesh2003/my-app:v2
   ```