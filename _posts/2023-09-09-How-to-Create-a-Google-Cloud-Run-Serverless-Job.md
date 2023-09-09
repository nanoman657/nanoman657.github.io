---
layout: post
title: "How to Create a Google Cloud Run Serverless Job"
thumbnail-img: https://storage.googleapis.com/gweb-cloudblog-publish/original_images/Cloud_Run.jpg
tags:
  - python
  - Docker
  - Google-Cloud-Platform(GCP)
  - Serverless-Computing
  - Tutorial
---
Serverless computing is an excellent approach when you just want to run simple code on the cloud. without needing to consider hardware scaling. This tutorial shows how you can easily deploy code to Google Cloud Run with Docker and Python. Before you start, make sure that you have Docker Desktop and the gcloud CLI installed. You can check the [Learning Resources](#learning-resources) section for more info on how to do that.

### **Create the Project Structure**
   This project structure will contain all of the files we need to create our microservice.

1. First, create a folder named "cloud_run_project" to contain your project files. Open your terminal and execute the following commands:

    ```bash
    mkdir cloud_run_project
    cd cloud_run_project
    echo "print('something')" > print_something.py
    ```
	Here's a brief explanation of the code block:
	* `mkdir` will create a directory called "cloud_run_project"
	* `cd` enters the directory
	* `echo` creates a simple Python file
	* "print_something.py" is a very simple python file that prints the word "something." Most Cloud Run jobs you create will be much more complex, but the idea here is to get it running and add complexity later.

### **Create a Docker Container**
The microservice will be run on [a Docker container.](https://www.docker.com/resources/what-container/) You can think of a container as a package that contains your code, and everything it needs to run. That way, when you *deploy* your code, everything will work right out of the box, or rather the container. To do this, we need to define what that container is by using a Dockerfile.
	
1. In your terminal, create a Dockerfile *without a file extension*:
	```bash
	    touch Dockerfile # Creates the file
	    vi Dockerfile # Opens the file for editing
	    ```
2. Add the following code to your Dockerfile.
    ```Dockerfile
    FROM --platform=linux/amd64 python:3.10.6-slim-bullseye
    ADD print_something.py .
    CMD ["python", "./print_something.py"]
    ```
	Here's a brief explanation of the code block:
	 * `FROM` specifies the Docker image we want to use. An image is essentially the instructions needed to build a running container. Docker has a large number of images to select from on [Docker Hub](https://hub.docker.com/). We want a container that can run *on Linux computers that run an AMD64 chip.* If that sounds oddly specific, it's because of [Google Cloud Run requirements](https://cloud.google.com/run/docs/container-contract#languages). We also want the container to have Python installed.
	* `ADD` adds `print_something.py` to the current directory *inside the container*.
	* `CMD` uses python to run `print_something.py`.
3. **Build the Docker Image**
    In the "cloud_run_project" directory, build your Docker image using the following command:
    ```bash
    docker build -t cloud_run_project .
    ```
	That will 'build' the image for your container, tag it as `cloud_run_project`, and use the contents of the current directory to do it. The image is tagged so that you can refer to it with the simple tag name.

### **Configure Google Artifact Registry (GAR)**

1. **Determine your desired Google Artifact Registry (GAR) [location](https://cloud.google.com/artifact-registry/docs/repositories/repo-locations)**.
    GAR is where you will store your image to run on Google Cloud Run. Technically, the location you choose will impact the latency or time it takes to download your image in the future. If you need to access your data (the image) and you live in Iowa, it makes more sense to host the image closer to  `us-central1`rather than `australia-southeast1`. For our demo here, it won't make much of a material difference.
2. **Authenticate with Google Cloud**

    Authenticate Docker with Google Cloud by running the following command, replacing "[your-location]" with your chosen location:

    ```bash
    gcloud auth configure-docker [your-location]-docker.pkg.dev
    ```

3. **Create a GAR Repository**

    Create a repository to store your Docker images. 

    ```bash
    gcloud artifacts repositories create [repo-name] --location=[location-name] --repository-format=docker
    ```

4. **Tag and Push Your Docker Image**

    Now, we'll give the Docker image *another tag* to push it successfully to GAR. Your first image tag (e.g. 'cloud_run_project') will be used here instead of a very long hash id. Tag and push your Docker image to GAR using the following commands, replacing the bracketed values with your information:

    ```bash
    docker tag [first-image-tag] [your location name]-docker.pkg.dev/[google-cloud-project-id]/[your-repo-name]/[first-image-tag]:[your-tag]
    docker push [your location name]-docker.pkg.dev/[google-cloud-project-id]/[your-repo-name]/[first-image-tag]:[your-tag]
    ```

### **Create and Run a Cloud Run Job**

1. **Create the Cloud Run Job** 
   
   Use the following command to create a Cloud Run job:
```bash
    gcloud run jobs create [your-job-name] \
      --image=[your-location]-docker.pkg.dev/[project-id]/[folder]/[image-name]@sha256:[hash-code] \
      --region=[your-location]
```
   Now you'll see a message on your CLI confirming your success! 

   >✓ Creating job... Done.                                              

2. **Launch Your Cloud Run Project**
   Finally, run your Cloud Run job with the following command:

	```bash
	gcloud run jobs execute [cloud-run-job-name] --region=[your-location]
	```

	You should now see this message in your CLI:

> Execution [cloud-run-job-name] has successfully started running.

Congratulations! You now know how to create a minimalistic Google Cloud Run serverless job! As always, try iterating on this foundation to build expertise in Google Cloud Run.

### Learning Resources

- [gcloud CLI Installation](https://cloud.google.com/sdk/docs/install)
- [Docker Desktop Installation](https://www.docker.com/products/docker-desktop/)
- [Google Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Docker Official Documentation](https://docs.docker.com/)
- [Google Artifact Registry](https://cloud.google.com/artifact-registry)
