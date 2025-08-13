
# GitOps with GitHub Actions: Contact Form Application

This repository contains the source code for a PHP-based contact form application, its `Dockerfile`, Kubernetes manifest files, and Helm charts. A GitHub Actions pipeline is configured to build the application image, push it to ECR, and deploy it to an EKS cluster.

This application is designed to be deployed to the infrastructure provisioned in a separate repository by the [infra-actions](https://github.com/Thormie-Harshey/infra-actions) repository

### **Project Overview**

The goal of this project is to demonstrate a complete GitOps workflow for a microservice. The GitHub Actions pipeline is triggered by code changes, automating the build, containerization, and deployment of the application to a pre-existing EKS cluster.

### **Related Repository**

* **EKS Infrastructure:** The AWS environment required to run this application is defined and managed in the [infra-actions](https://github.com/Thormie-Harshey/infra-actions) repository. You must deploy the infrastructure first before attempting to deploy this application.

### **Architecture and Technologies**

The application deployment workflow leverages the following technologies:
The infrastructure architecture is defined as follows:
| Category        | Tools / Services |
|----------------|------------------|
| **Application** | PHP Contact Form |
| **Containerization** | Docker |
| **CI/CD** | GitHub Actions |
| **Deployment Tool** |  Helm |
| **Container Registry** |  AWS ECR (provisioned by the infrastructure pipeline) |
| **Ingress** |  NGINX Ingress Controller (provisioned by the infrastructure pipeline) |
---

### **Getting Started**

Before you can deploy this application, you must first have the necessary infrastructure running. The required values from the infrastructure deployment must be configured in this repository's secrets and variables. The required values would be printed out in the console log for you after deployment. They include:
-   `EKS_CLUSTER_NAME`: The name of the EKS cluster.
-   `REGISTRY`: The URL of the ECR registry.

These particular set of detailed need to be gotten from the output of your running infrastructure and configured in your repository's secrets and variables. Some others, you already have.

#### **GitHub Repository Secrets**
Some other GitHub repository secrets are required for this project and they include: 
1.  `AWS_ACCESS_KEY_ID`: Your AWS access key.
2.  `AWS_SECRET_ACCESS_KEY`: Your AWS secret key.
3.  `REGISTRY`: The ECR registry URL obtained from the infrastructure repository's outputs.
4.  `KUBECONFIG`: The kubeconfig file content obtained from the infrastructure repository's outputs.

#### **GitHub Repository Variables**

1.  `AWS_REGION`: The AWS region where resources will be deployed (e.g., `us-east-1`).
2.  `ECR_REPOSITORY`: The name of the ECR repository obtained from the infrastructure repository's outputs.

### **Usage**

#### **1. Deploying the Application**

The deployment workflow is defined in `.github/workflows/deploy.yml`. It is triggered on a `push` to the `code-actions` branch.

1.  Before making a push, ensure the EKS cluster and ECR repository have been successfully provisioned by the `infra-actions` repository.
2.  Push your code changes or update the Helm charts and Kubernetes manifests to the `code-actions` branch.
3.  The GitHub Actions workflow will then:
    * Build the Docker image.
    * Push the image to ECR.
    * Deploy the application to the EKS cluster using Helm.
    * You can monitor the progress of the deployment in the "Actions" tab.

In deploying to EKS, one of the crucial steps the pipeline will run is to authenticate with ECR. This is necessary because ECR is a private repository that stores the Docker images to be used by EKS. In order to authenticate, one must first run the command `kubectl get secret regcred` in order to know if there has been any previously created registry credentials. If not, then we go ahead to create a new one. This ensures EKS can access the required images from ECR. 
```YAML
        - name: Login to ECR
          run: |
            kubectl get secret regcred || \
            kubectl create secret docker-registry regcred \
              --docker-server=${{ secrets.REGISTRY }} \
              --docker-username=AWS \
              --docker-password=$(aws ecr get-login-password)
```

#### **2. Helm Charts**

This repository uses Helm charts to manage the deployment of multiple Kubernetes resources:

* **`helm/appcharts/templates/`:** Contains the manifest files for the PHP web application, phpMyAdmin, and their respective services and ingress rules.
* The image tag is dynamically updated during the pipeline run, avoiding hardcoding.
```YAML
cluster-name: ${{ env.EKS_CLUSTER }}
chart-path: helm/appcharts
namespace: default
values: appimage=${{ secrets.REGISTRY }}/${{ env.ECR_REPOSITORY }},apptag=${{ github.run_number }}
name: webapp-stack
```
### **DNS Configuration**

Once the application is successfully deployed, you can access it via the internet using custom domain names, but we first have to add the URL of the Network Load Balancer that NGINX Ingress has created as a CNAME record on GoDaddy.

1.  In your DNS provider (e.g., GoDaddy), create two `CNAME` records.
2.  Point both records to the URL of the AWS Network Load Balancer (NLB) created by the NGINX Ingress Controller. You can find this URL in the logs of the infrastructure deployment or on the AWS console. You can also run the command `kubectl get ingress` if you do not want to go to your console for any reason.

This setup allows the NGINX Ingress controller to route traffic to the correct application based on the domain name.


## **Key Concepts** 
* **Dynamic Tagging:** Images are automatically tagged using `github.run_number` for version control. It helped to avoid pipeline failures as a result of incorrect image tagging.
*  **Helm Templating:** The pipeline dynamically injects the image name and tag into the Helm charts, avoiding hardcoded values. 
* **Private Registry Access:** A Kubernetes `regcred` secret is created to allow EKS to securely pull images from ECR by either creating new registry credentials or by making use of one which already exists.
* **Ingress Routing:** The application and its phpMyAdmin database are exposed via an Ingress controller, routing traffic to different subdomains (`webapp.yourdomain.com` and `phpmyadmin.yourdomain.com`). 

## **Outcomes**
 * **Automated CI/CD:** A pipeline for building and deploying the application.
 * **Functional Application:** The contact form and its phpMyAdmin database were successfully deployed and made accessible on the internet via custom domain names. 
