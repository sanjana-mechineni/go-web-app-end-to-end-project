Project Overview:

This project demonstrates a complete DevOps workflow for a Go (Golang) web application, taking it all the way from source code on a developer's machine to a running service on a Kubernetes cluster in AWS. The goal is a fully automated pipeline where a single code change flows through continuous integration, container image creation, a Helm chart update, and GitOps-based deployment — with no manual kubectl commands in the final delivery step.
In short, every time a developer commits code, the pipeline automatically: builds and tests the application, performs static code analysis, builds and pushes a new Docker image, updates the Helm chart with the new image tag, and lets Argo CD sync that change to the Kubernetes cluster. This is the essence of CI/CD + GitOps.
What you'll learn:  containerization with multi-stage Docker builds, writing Kubernetes manifests, exposing apps with an Ingress controller, packaging with Helm, automating CI with GitHub Actions, and continuous delivery with Argo CD on Amazon EKS.

2. Architecture:

The application follows a GitOps delivery model. The flow of a code change through the system is:
Developer → GitHub → GitHub Actions (CI) → Docker Hub → Helm values update → Argo CD (CD) → Amazon EKS → Service → Ingress → Users
How the pieces connect
●	Developer & GitHub — source code and Kubernetes/Helm config live in one Git repository, the single source of truth.
●	GitHub Actions (CI) — triggered on every commit; builds, tests, scans, and pushes the Docker image, then updates the image tag in the Helm chart.
●	Docker Hub — stores the built container images that Kubernetes will pull.
●	Argo CD (CD) — continuously watches the Helm chart in Git and syncs any change to the cluster, so the cluster state always matches Git.
●	Amazon EKS — the managed Kubernetes cluster where the app actually runs.
●	Ingress & Service — route external user traffic to the correct application pods.

3. Tech Stack & Tools:

●	Language / App: Go (Golang) web application
●	Containerization: Docker (multi-stage build with a distroless base image)
●	Registry: Docker Hub
●	Orchestration: Kubernetes on Amazon EKS
●	Packaging: Helm charts
●	Ingress: NGINX Ingress Controller (host-based routing)
●	CLI tooling: AWS CLI, kubectl, eksctl, git
●	CI: GitHub Actions
●	CD / GitOps: Argo CD

4. Prerequisites:

Before you begin, make sure you have the following ready on your machine and cloud account:
●	An AWS account and an IAM user with permissions to create EKS clusters.
●	AWS CLI installed and configured (aws configure).
●	kubectl — the Kubernetes command-line tool.
●	eksctl — CLI to create and manage EKS clusters.
●	Helm — the Kubernetes package manager.
●	Docker — to build and run container images, plus a Docker Hub account.
●	Git and a GitHub repository for the project.

5. Repository Structure:

Organize the repository so that application code, Kubernetes manifests, the Helm chart, and the CI workflow all live together:
go-web-app/
├── main.go                  # application source
├── go.mod                   # Go module definition
├── Dockerfile               # multi-stage build
├── k8s/
│   └── manifests/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
├── helm/
│   └── go-web-app-chart/    # Helm chart (templates + values.yaml)
└── .github/
    └── workflows/
        └── ci.yaml          # GitHub Actions CI pipeline

6. Important Concepts Explained:

A few core ideas make this pipeline work. Here is a concise explanation of each.
Multi-stage Docker builds & distroless image:
A multi-stage Dockerfile uses more than one build stage. Stage 1 uses a full Go image to download dependencies and compile the binary. Stage 2 copies only the compiled binary into a minimal distroless base image. The result is a much smaller image (no compilers, shells, or package managers left behind) which is both faster to pull and more secure, since a smaller attack surface means fewer things to exploit.

Kubernetes manifests: Deployment, Service, Ingress:
Deployment declares the desired state of your app — which image to run and how many replicas. Service gives those pods a stable network identity inside the cluster. Ingress defines rules for routing external HTTP traffic to a Service, typically based on hostname or path.

Ingress vs. Ingress Controller:
An Ingress resource is just a set of routing rules — on its own it does nothing. An Ingress Controller (here, NGINX) is the component that actually watches for Ingress resources and provisions a load balancer to fulfill them. The ingressClassName on the Ingress tells the controller "this resource is for you to handle." Without it, the controller ignores the resource. This project uses host-based routing, so only requests for the configured hostname (go-web-app.local) reach the app.

Helm:
Helm is the package manager for Kubernetes. Instead of applying many static YAML files, you bundle them into a chart with templated values. Variables like the image tag are pulled from values.yaml (for example, image: {{ .Values.image.tag }}). This means the CI pipeline can update a single value to roll out a new version, and Helm handles installing, upgrading, and rolling back the whole application as one unit.

CI with GitHub Actions:
Continuous Integration automatically validates and packages every code change. The GitHub Actions workflow here runs four stages: (1) build and unit test, (2) static code analysis, (3) build and push the Docker image, and (4) update the Helm chart with the new image tag. It is triggered on every push, so problems are caught early and a deployable artifact is always ready.

CD & GitOps with Argo CD:
GitOps means Git is the single source of truth for what runs in the cluster. Argo CD continuously watches the Helm chart in the repository. When the CI pipeline updates the image tag in values.yaml, Argo CD detects the change, pulls the updated chart, and syncs it to the EKS cluster automatically — no manual kubectl apply needed. If the live cluster ever drifts from Git, Argo CD flags it and can restore the desired state.

7. Step-by-Step Implementation:

The following stages walk through the entire project, from building the application locally to a fully automated GitOps deployment. Each stage includes the commands used and the screenshots captured while reproducing the project.
Step 1 — cloone the source code repo, build and run the app locally
Start by building the Go binary, and running it locally to confirm the application works before containerizing it.
go build -o main .
./main

Open a browser and confirm the application is serving pages on localhost.
 
Figure: Cloning the repo, building the Go binary (go build -o main) and running it with ./main.
 
Figure: The Go web application running locally in the browser — "Learn DevOps from Basics".

Step 2 — Containerize with Docker & push to Docker Hub

Write a multi-stage Dockerfile: stage 1 downloads dependencies and compiles the binary; 
stage 2 uses a distroless base image to keep the final image small and secure. Then build and run the image locally.

•	vim Dockerfile
•	docker build -t <your-dockerhub-username>/go-web-app:v1 .
•	docker run -p 8080:8080 -it <your-dockerhub-username>/go-web-app:v1
 
Figure: Multi-stage Docker build using a distroless base image, then running the container.

Once the image runs correctly, push it to Docker Hub so Kubernetes can pull it later.
●	docker push <your-dockerhub-username>/go-web-app:v1
 
Figure: The image published to Docker Hub under the go-web-app repository.

Concept recap:  containerization is complete once the Dockerfile is written, the image builds and runs, and it is pushed to a registry.

Step 3 — Write Kubernetes manifests
Create a k8s/manifests folder and add three manifests that describe how the app runs in Kubernetes:
●	deployment.yaml — runs the container image as pods.
●	service.yaml — gives the pods a stable in-cluster endpoint.
●	ingress.yaml — host-based routing so only requests for the configured hostname reach the app.

Why ingressClassName matters:  the Ingress Controller only watches resources that carry its class name. Omit it, and your Ingress is silently ignored.

Step 4 — Create the EKS cluster & deploy manifests
Install AWS CLI, KUBECTL, EKSCTL
●	kubectl – A command line tool for working with Kubernetes clusters. For more information, see Installing or updating kubectl.
●	eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see Installing or updating.
●	AWS CLI – A command line tool for working with AWS services, including Amazon EKS. For more information, see Installing, updating, and uninstalling the AWS CLI in the AWS Command Line Interface User Guide. After installing the AWS CLI, we recommend that you also configure it. For more information, see Quick configuration with aws configure in the AWS Command Line Interface User Guide.

●	With kubectl, eksctl and the AWS CLI configured, create an EKS cluster, then apply the manifests to deploy the application.
●	# install AWS CLI
sudo apt update
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws –version

●	# install kubectl
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

●	# install eksctl (Linux example)
ARCH=amd64
       PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
       tar -xzf eksctl_${PLATFORM}.tar.gz && sudo mv eksctl /usr/local/bin/
eksctl version

●	# configure AWS CLI
aws configure
●	# Create a EKS cluster
eksctl create cluster --name demo-cluster --region us-east-1
 
•	kubectl apply -f k8s/manifests/deployment.yaml
•	kubectl apply -f k8s/manifests/service.yaml
•	kubectl apply -f k8s/manifests/ingress.yaml
•	kubectl get all

eksctl create cluster --name demo-cluster --region us-east-1

you did not explicitly mention EC2 instance type or number of nodes, but eksctl still creates worker nodes using default values.
eksctl automatically creates:
1.	An EKS control plane (managed by AWS) 
2.	A managed node group 
3.	EC2 worker instances for that node group 

The defaults depend on the eksctl version and configuration, but commonly:
•	Instance type: m5.large (in many default setups) 
•	Desired node count: 2 nodes 
•	Min nodes: 2 
•	Max nodes: 2 
•	So internally, it is similar to creating:
managedNodeGroups:
  - name: demo-cluster-ng
    instanceType: m5.large
    desiredCapacity: 2

since, we are using AWS free tier, created EKS cluster using the below command and EC2 type:

eksctl create cluster \
--name demo-cluster \
--region us-east-1 \
--node-type m7i-flex.large \
--nodes 2 \
--nodes-min 2 \
--nodes-max 2

# To check the node group
●	eksctl get nodegroup \
--cluster demo-cluster \
--region us-east-1

 
Figure: kubectl get all — the deployment, pods and service created on the EKS cluster.

To quickly verify the Service, temporarily change its type to NodePort, get the assigned port, whitelist that port in the EC2 security group's inbound rules, and access the app via the node's external IP.
•	kubectl edit svc          # change type: ClusterIP -> NodePort
•	kubectl get svc           # note the NodePort and external IP
 
Figure: The application reachable through the NodePort service.

Step 5 — Configure the Ingress Controller
Install the NGINX Ingress Controller. It watches Ingress resources and automatically provisions an AWS load balancer to route traffic according to your Ingress rules.
•	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
•	kubectl get pods -n ingress-nginx
•	kubectl get ing
 
Figure: The Ingress Controller pods running and the Ingress resource with its load balancer address.

The controller creates a load balancer (visible in EC2 → Load Balancers). Resolve its DNS name to an IP, then map the hostname locally so the browser sends requests to the right place.
•	nslookup <load-balancer-dns-name>
•	sudo vim /etc/hosts     # add:  <LB-IP>  go-web-app.local
 
Figure: Mapping the load balancer IP to go-web-app.local in /etc/hosts.

Now browse to the host and the request flows through the load balancer and Ingress to the app.
 
Figure: The application served through the Ingress at go-web-app.local.

Step 6 — Package the app with Helm
Convert the raw manifests into a reusable Helm chart so the image tag can be templated and updated automatically by CI.
•	mkdir helm && cd helm
•	helm create go-web-app-chart
# remove default templates, then copy your manifests in:
•	cd go-web-app-chart/templates && rm -rf *
•	cp ../../../k8s/manifests/* .
# in deployment.yaml, template the image tag:
#   image: <your-dockerhub-username>/go-web-app:{{ .Values.image.tag }}
# you need to update the values.yaml accordingly
 
Figure: Creating the Helm chart scaffold with helm create.
 
Figure: The templated deployment referencing {{ .Values.image.tag }} and the values.yaml file.

•	Before installing via Helm, clean up any resources created earlier from the raw manifests to avoid conflicts.
•	kubectl delete deploy go-web-app
•	kubectl delete svc go-web-app
•	kubectl delete ing go-web-app
 
Figure: Removing the previously applied deployment, service and ingress.

Now install the whole application as a single Helm release, then confirm the resources.
•	helm install go-web-app ./go-web-app-chart
•	kubectl get all
 
Figure: helm install deploys the full application as one release (STATUS: deployed).
 
Figure: The deployment, service and ingress recreated by the Helm release.
Tip:  helm uninstall go-web-app removes everything the release created in one command — handy for clean resets before automating with Argo CD.
 
Step 7 — Continuous Integration with GitHub Actions
Create the workflow file at .github/workflows/ci.yaml. On every push it runs four stages:
●	Stage 1 — Build and unit test the Go application.
●	Stage 2 — Static code analysis (linting / code quality).
●	Stage 3 — Build the Docker image and push it to Docker Hub.
●	Stage 4 — Update the Helm chart's values.yaml with the new image tag.
●	mkdir -p .github/workflows
●	vim .github/workflows/ci.yaml   # define the four stages above

Add the required credentials as repository secrets (Settings → Secrets and variables → Actions) so the workflow can push to Docker Hub and commit the Helm tag update.
 
Figure: Docker Hub credentials stored as encrypted GitHub repository secrets.
On each commit the pipeline runs automatically. A green run confirms build, code-quality, push and the Helm tag update all succeeded.
 
Figure: A successful GitHub Actions run: build, code-quality, push and update-tag-in-helm-chart.

Step 8 — Continuous Delivery with Argo CD (GitOps)
Argo CD watches the Helm chart in Git. When CI bumps the image tag in values.yaml, Argo CD pulls the change and syncs it to the EKS cluster — completing the automated GitOps loop. Install Argo CD and expose its UI:
●	kubectl create namespace argocd
●	kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# expose the Argo CD UI via a load balancer:
●	kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
●	kubectl get svc argocd-server -n argocd
 
Figure: Patching the argocd-server service to LoadBalancer and reaching the Argo CD login page.
 
Figure: The Argo CD login screen — "Let's get stuff deployed!".

The default username is admin. Retrieve the auto-generated initial password from the cluster secret:
●	kubectl get secrets -n argocd
●	kubectl edit secret argocd-initial-admin-secret -n argocd
# the password field is base64-encoded, decode it:
●	echo <encoded-value> | base64 --decode
 
Figure: Retrieving and decoding the Argo CD initial admin password from the secret.
Log in, create an application pointing at your Helm chart path in the repo, and Argo CD renders the resource tree and keeps it in sync with Git — showing Healthy and Synced status.
 
 
Figure: The Argo CD application view: the full resource tree, Healthy and Synced.
 
Figure: The final application delivered end-to-end through the pipeline — "Learn DevOps from Basics By Sanjana".

8. Publishing This Project to GitHub
To share this project on your own GitHub account, create a new empty repository on GitHub (without a README so there is nothing to merge), then from the project folder run:
cd go-web-app
git init
git add .
git commit -m "Initial commit: Go web app DevOps CI/CD pipeline"
git branch -M main
git remote add origin https://github.com/<your-username>/go-web-app.git
git push -u origin main

Note:  GitHub renders images from relative paths in the repo, e.g. ![Local build](assets/screenshots/01-local-build.png). Commit the images together with the README so they display for everyone.
9. Conclusion
This project ties together the full modern DevOps toolchain into one automated pipeline. A developer simply commits code; GitHub Actions builds, tests, scans and ships a container image, updates the Helm chart, and Argo CD delivers it to Amazon EKS — all without manual deployment steps. Along the way you practiced multi-stage Docker builds, Kubernetes manifests, Ingress routing, Helm packaging, CI automation and GitOps continuous delivery: the same patterns used by production teams to ship software safely and repeatedly.
