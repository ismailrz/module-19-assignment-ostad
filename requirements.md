Assignment: Docker & Kubernetes Deployment on AWS EC2
Objective
Deploy the provided application using Docker and Kubernetes on an AWS EC2-based Kubernetes cluster.

Application Repository:
https://github.com/roy35-909/Module-3-deployment



Part 1 — Application Containerization
Clone the provided application repository.

Review the application structure and identify how the application is started.

Create a Dockerfile for the application.

Build the Docker image.

Run the application using the Docker image and verify that it works.

Tag the image appropriately.

Create/use a Docker Hub repository.

Push the application image to Docker Hub.

Verify that the image is available from Docker Hub.

Submission: Screenshots showing the Docker image working locally and the image available on Docker Hub.



Part 2 — Kubernetes Cluster on AWS EC2
Create the required AWS EC2 instances for a Kubernetes cluster.

Configure the instances with the required networking and access.

Install the required container runtime and Kubernetes components.

Create the Kubernetes cluster using kubeadm.

Configure the worker node(s) and join them to the cluster.

Verify that the cluster and nodes are working correctly.

Submission: Screenshots showing the AWS instances and healthy Kubernetes nodes.



Part 3 — Secure etcd
Identify where Kubernetes stores cluster state and sensitive Kubernetes objects.

Configure encryption at rest for Kubernetes data stored in etcd.

Create a Kubernetes Secret for testing.

Verify that the Secret is protected through encryption at rest.

Ensure that sensitive Secret data is not exposed publicly.

Submission: Screenshots showing the encryption-at-rest configuration and verification.



Part 4 — Static NGINX Pod
Create a static Pod running NGINX.

Configure it to serve the default NGINX page.

Verify that the NGINX Pod is running.

Verify that the default NGINX page can be accessed.

Submission: Screenshots showing the static Pod and NGINX default page.



Part 5 — Deploy the Application
Create a Kubernetes Deployment resource for the application.

Configure the Deployment to use the Docker image pushed to Docker Hub.

Configure the required number of replicas.

Verify that the application Pods are running successfully.

Verify that the Deployment is healthy.

Submission: Screenshots showing the Deployment, ReplicaSets, and Pods.



Part 6 — PostgreSQL StatefulSet
Create a Kubernetes StatefulSet for PostgreSQL.

Configure the required PostgreSQL container settings.

Configure persistent storage for PostgreSQL.

Verify that the PostgreSQL Pod is running.

The application Deployment does not need to connect to PostgreSQL for this assignment.

Submission: Screenshots showing the StatefulSet and PostgreSQL Pod running.



Part 7 — Persistent Volume and PVC
Create a PersistentVolume (PV).

Create a PersistentVolumeClaim (PVC).

Bind the PVC to the PV.

Configure the application Pod to use the PVC.

Verify that the PVC is successfully mounted by the application Pod.

Submission: Screenshots showing the PV, PVC, binding status, and Pod storage configuration.



Part 8 — Expose the Application
Create a Kubernetes NodePort Service for the application Deployment.

Configure the Service to route traffic to the application Pods.

Access the application using the NodePort.

Verify that the application is accessible from outside the cluster.

Submission: Screenshot showing the Service and the application accessible through the NodePort.



Part 9 — Kubernetes Resource Documentation
Prepare a short explanation of the following Kubernetes resources and concepts:

Pod

Deployment

ReplicaSet

StatefulSet

DaemonSet

Static Pod

Service

NodePort

PersistentVolume (PV)

PersistentVolumeClaim (PVC)

StorageClass

Secret

ConfigMap

For each resource, explain:

What it is

What problem it solves

When it should be used

One practical example



Part 10 — Kubernetes Security
Explain how a Kubernetes cluster can be secured.

Your documentation should cover:

Kubernetes Secrets

Encryption at rest

etcd security

RBAC

Service Accounts

Network Policies

TLS

API Server security

Node security

Container image security

Pod security

Kubernetes version and patch management

Limiting public exposure of cluster component

Part 11 — Deployment Automation
Explain how the application deployment process can be automated when a new Docker image version is released.

Your documentation should describe:

Building a new application image.

Tagging the new image version.

Pushing the image to Docker Hub.

Updating the Kubernetes Deployment.

Verifying the new version.

Automating the process using CI/CD.

Part 12 — Deployment Strategies
Explain the following Kubernetes deployment strategies:

Rolling Update
Explain:

How it works

Advantages

Limitations

When to use it

Recreate
Explain:

How it works

Advantages

Limitations

When to use it

Canary Deployment
Explain:

How it works

Advantages

Limitations

When to use it

Also explain the differences between these strategies and provide a suitable use case for each.

Submission Requirements
Submit a document containing:

Screenshots of every practical step working successfully.

Kubernetes resource documentation.

Kubernetes security documentation.

Deployment automation explanation.

Deployment strategy explanation.

Docker Hub image information.

AWS EC2 and Kubernetes cluster verification.

Screenshots should clearly show the relevant resource, status, and successful operation.

Do not submit only commands. The screenshots must demonstrate that each required component is working.
