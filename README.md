# Coworking Space Service Extension
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

## Getting Started

### Dependencies
#### Local Environment
1. Python Environment - run Python 3.6+ applications and install Python dependencies via `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `minikube` - quickly sets up a local Kubernetes cluster on macOS, Linux, and Window

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. Kubernetes Environment with AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code

### Setup
#### 1. Create an EKS cluster
Using AWS CLI to create EKS cluster and update the context in the local Kubeconfig file

1. Create EKS cluster
    ```bash
    eksctl create cluster --name my-cluster --region us-east-1 --nodegroup-name my-nodes --node-type t3.small --nodes 1 --nodes-min 1 --nodes-max 2
    ```

2. After EKS cluster is created, update current context
    ```bash
    aws eks --region us-east-1 update-kubeconfig --name my-cluster
    ```

#### 2. Configure a Database
Set up a Postgres database

1. Inside *deployment* folder, run the following commands to create a Postgres deployment
    ```bash
    kubectl apply -f configmap.yaml
    kubectl apply -f pvc.yaml
    kubectl apply -f pv.yaml
    kubectl apply -f postgresql-deployment.yaml
    ```

2. Test database connection<br>
    - View the pod
        ```bash
        kubectl get pods
        ```
    - Run the following command to open bash into the pod, replace the corresponding POD_NAME
        ```bash
        kubectl exec -it <POD_NAME> -- bash
        ```
    - Once you are inside the pod, you can run
        ```bash
        psql -U myuser -d mydatabase
        ```
    - Ensure to change the username and password, as applicable to you. Once you are inside the postgres database, you can list all databases
        ```bash
        \l
        ```

3. Expose the Postgres service using port-forwarding approach
    ```bash
    # Create service
    kubectl apply -f postgresql-service.yaml
    ```

    ```bash
    # Set up port-forwarding to `postgresql-service`
    kubectl port-forward service/postgresql-service 5433:5432 &
    ```

4. Run Seed Files
We will need to run the seed files in `db/` in order to create the tables and populate them with data.

    ```bash
    export DB_PASSWORD=mypassword
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5433 < <FILE_NAME.sql>
    ```

#### 3. Deploy Analytics to remote resource 
Deploy analytics application to AWS EKS

1. Create an Amazon ECR repository on your AWS console.

2. Create an Amazon CodeBuild project that is connected to your project's GitHub repository

Remember to set up ENV variables in CodeBuild console

* `AWS_DEFAULT_REGION`
* `AWS_ACCOUNT_ID`
* `IMAGE_REPO_NAME`
* `DB_USERNAME`
* `DB_PASSWORD`

3. Click on the "Start Build" button on your CodeBuild console and then check out Amazon ECR to see if the Docker image is created/updated.

4. Get db host
    ```bash
    kubectl get svc
    ```

5. Update db host to configmap.yaml

    ```
    kubectl apply -f configmap.yaml
    ```

6. Update docker image uri from AWS ECR to coworking.yaml

7. Deploy coworking analytics application
    ```bash
    kubectl apply -f coworking.yaml
    ```

8. Verify
    ```bash
    kubectl get svc
    ```
#### 4. Setup AWS Cloudwatch
1. Attach the **CloudWatchAgentServerPolicy** IAM policy to your worker nodes:
    ```
    aws iam attach-role-policy \
        --role-name my-worker-node-role \
        --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy 
    ```
    Replace `my-worker-node-role` with EKS cluster's Node Group's IAM role.<br/>
    To find out the name of the IAM role, open up the Compute tab of your EKS cluster, click on the Node Group.<br/>
    And from there, click on the IAM role ARN under the Details tab.

2. Use AWS CLI to install the Amazon CloudWatch Observability EKS add-on:
    ```bash
    aws eks create-addon --addon-name amazon-cloudwatch-observability --cluster-name my-cluster-name
    ```
    Replace `my-cluster-name` with your EKS cluster's name.

3. Verify by trigger logging by accessing your application.

#### 5. Verifying The Application
* Generate report for check-ins grouped by dates
`curl <BASE_URL>/api/reports/daily_usage`

* Generate report for check-ins grouped by users
`curl <BASE_URL>/api/reports/user_visits`

### Setup locally
#### 1. Set up minikube cluster

1. Start minukube
    ```
    minikube start
    ```
    This command somewhat parallels the command you have learned earlier to connect kubectl with Amazon EKS:
    ```
    aws eks --region <REGION> update-kubeconfig --name <CLUSTER_NAME>
    ```
2. Change the Docker environment so that minikube can "see" your Docker images

    ```
    eval $(minikube docker-env)
    ```

3. Remember to change type `Load Balancer` to `NodePort`

    ```
    spec:
        type: LoadBalancer
    ```
    to
    ```
    spec:
        type: NodePort
    ```

#### 2. Set up database
Set up Postgres db

1. Inside *deployment-local* folder, run the following commands to create a Postgres deployment
    ```bash
    kubectl apply -f configmap.yaml
    kubectl apply -f pvc.yaml
    kubectl apply -f pv.yaml
    kubectl apply -f postgresql-deployment.yaml
    ```

2. Test database connection<br>
    - View the pod
        ```bash
        kubectl get pods
        ```
    - Run the following command to open bash into the pod, replace the corresponding POD_NAME
        ```bash
        kubectl exec -it <POD_NAME> -- bash
        ```
    - Once you are inside the pod, you can run
        ```bash
        psql -U myuser -d mydatabase
        ```
    - Ensure to change the username and password, as applicable to you. Once you are inside the postgres database, you can list all databases
        ```bash
        \l
        ```

3. Expose the Postgres service using port-forwarding approach
    ```bash
    # Create service
    kubectl apply -f postgresql-service.yaml
    ```

    ```bash
    # Set up port-forwarding to `postgresql-service`
    kubectl port-forward service/postgresql-service 5433:5432 &
    ```

4. Run Seed Files
We will need to run the seed files in `db/` in order to create the tables and populate them with data.

    ```bash
    export DB_PASSWORD=mypassword
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5433 < <FILE_NAME.sql>
    ```

#### 3. Build analytics application locally
Build analytics application

1. From root directory, build docker image
    ```bash
    docker build -t $IMAGE_REPO_NAME:latest ./analytics --build-arg DB_USERNAME=$DB_USERNAME --build-arg DB_PASSWORD=$DB_PASSWORD
    ```
2. Update docker image uri in deployment-local/coworking.yaml

3. Update configmap and apply

    ```bash
    kubeclt apply -f configmap.yaml
    ```

4. Deploy
    ```bash
    kubectl apply -f coworking.yaml
    ```

5. Verify
    ```bash
    kubectl get svc
    ```

#### 5. Verifying The Application
minikube does not allowed to access application directly on MacOS (as it state from [minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/))

> The network is limited if using the Docker driver on Darwin, Windows, or WSL, and the Node IP is not reachable directly.

So we need to create a tunnel to access the application

```bash
minikube service coworking-service --url
```

After running the above command, replace `BASE_URL` with the returned url and trigger api
```bash
# Generate report for check-ins grouped by dates
curl <BASE_URL>/api/reports/daily_usage
```

```bash
# Generate report for check-ins grouped by users
curl <BASE_URL>/api/reports/user_visits
```

## Project Instructions
1. Set up a Postgres database with a Helm Chart
2. Create a `Dockerfile` for the Python application. Use a base image that is Python-based.
3. Write a simple build pipeline with AWS CodeBuild to build and push a Docker image into AWS ECR
4. Create a service and deployment using Kubernetes configuration files to deploy the application
5. Check AWS CloudWatch for application logs

### Deliverables
1. `Dockerfile`
2. Screenshot of AWS CodeBuild pipeline
3. Screenshot of AWS ECR repository for the application's repository
4. Screenshot of `kubectl get svc`
5. Screenshot of `kubectl get pods`
6. Screenshot of `kubectl describe svc <DATABASE_SERVICE_NAME>`
7. Screenshot of `kubectl describe deployment <SERVICE_NAME>`
8. All Kubernetes config files used for deployment (ie YAML files)
9. Screenshot of AWS CloudWatch logs for the application
10. `README.md` file in your solution that serves as documentation for your user to detail how your deployment process works and how the user can deploy changes. The details should not simply rehash what you have done on a step by step basis. Instead, it should help an experienced software developer understand the technologies and tools in the build and deploy process as well as provide them insight into how they would release new builds.


### Stand Out Suggestions
Please provide up to 3 sentences for each suggestion. Additional content in your submission from the standout suggestions do _not_ impact the length of your total submission.
1. Specify reasonable Memory and CPU allocation in the Kubernetes deployment configuration
2. In your README, specify what AWS instance type would be best used for the application? Why?
3. In your README, provide your thoughts on how we can save on costs?

### Best Practices
* Dockerfile uses an appropriate base image for the application being deployed. Complex commands in the Dockerfile include a comment describing what it is doing.
* The Docker images use semantic versioning with three numbers separated by dots, e.g. `1.2.1` and  versioning is visible in the  screenshot. See [Semantic Versioning](https://semver.org/) for more details.