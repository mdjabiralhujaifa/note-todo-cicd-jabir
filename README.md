Note‑Todo | End‑to‑End DevSecOps on Kubernetes

A complete DevSecOps pipeline that builds, scans, ships, and runs a Node.js “Todo” app on Kubernetes with Jenkins, SonarQube, OWASP Dependency‑Check, Trivy, Docker, Prometheus, and Grafana. Optional logs shipping via Fluent Bit.


1) What this project does
	•	Build & Test: Jenkins pulls code from GitHub, installs Node deps, runs quality checks.
	•	Secure by default: Static analysis (SonarQube), dependency & image scans (OWASP DC, Trivy).
	•	Containerize & Publish: Docker image built and pushed to Docker Hub with dynamic tag (v${BUILD_NUMBER}).
	•	Deploy: Kubernetes rolling update to the todo-app Deployment.
	•	Observe: Prometheus scrapes metrics; Grafana visualizes dashboards.
	•	(Optional) Log to S3: Fluent Bit DaemonSet forwards container logs.


2) Architecture (at a glance)

.---------------------------------------------------- KUBERNETES CLUSTER ----------------------------------------------------.
|                                                                                                                            |
|  .---------------------.                                     .---------------------.                                       |
|  |  CONTROL-PLANE      |                                     |  WORKER NODE        |                                       |
|  |  (master)           |                                     |  (apps run here)    |                                       |
|  |  - kube-apiserver   |                                     |  - kubelet          |                                       |
|  |  - controller mgr   |                                     |  - kube-proxy       |                                       |
|  |  - scheduler        |                                     |  - container runtime|                                       |
|  '---------------------'                                     '---------------------'                                       |
|                                                                                                                            |
|  .--------------------------- NAMESPACES ---------------------------------.  .--------------------------- NAMESPACES -----.|
|  |  default                                                               |  |  - monitoring                               |
|  |  - Deployment: todo-app                                                |  |  - Deployments: prometheus, grafana         |
|  |  - Service:    todo-service (NodePort :31000)                          |  |  - DaemonSet: node-exporter                 |
|  '------------------------------------------------------------------------'  |  - kube-state-metrics (Service)             |
|                                                                               '-------------------------------------------'|
|  .--------------------------- NAMESPACES ---------------------------------------------------------------------------------.|
|  |  - logging                                                                                                              |
|  |  - DaemonSet: fluent-bit  -->  (optional) forward container logs to S3                                                  |
|  '----------------------------------------------------------------------------------------------------------------------'  |
'----------------------------------------------------------------------------------------------------------------------------'

CI/CD Flow

GitHub  --push-->  Jenkins
  |                   |
  '-------- pull ----'
                      |  stages:
                      |  1) Clean workspace
                      |  2) SonarQube analysis
                      |  3) OWASP DC + Trivy FS
                      |  4) Docker build & push
                      |  5) Trivy image scan
                      |  6) Docker run (local smoke)
                      |  7) kubectl set image (K8s)
                      v
                 Kubernetes



3) Repository layout (suggested)

.
├─ Jenkinsfile
├─ Dockerfile
├─ package.json
├─ src/ , views/ , etc.
├─ monitoring/
│  ├─ grafana-deployment.yaml
│  ├─ grafana-service.yaml
│  ├─ kube-state-metrics.yaml
│  ├─ node-exporter.yaml
│  ├─ prometheus-configmap.yaml
│  ├─ prometheus-deployment.yaml
│  └─ prometheus-service.yaml
└─ logging/
   └─ s3-bucket-setup.yaml                # (optional) Fluent Bit + config



4) Prerequisites
	•	Kubernetes cluster (1 control‑plane + ≥1 worker).
	•	kubectl configured on the Jenkins/master node.
	•	Docker and Docker Hub account.
	•	Jenkins with these tools/credentials:
	•	Global tools: jdk21, node24, sonar-scanner
	•	Credentials:
	•	docker (Docker Hub username/password or token)
	•	k8s (Kubeconfig or ServiceAccount token for the cluster)
	•	SonarQube server named sonar-server.
	•	NodePorts open in Security Group (from your IP only is recommended):
	•	App: 31000
	•	Prometheus: 30090
	•	Grafana: 30030
	•	Node Exporter: 30100
	•	(Your ports can differ; check with kubectl get svc -n monitoring)


5) Jenkins pipeline (Jenkinsfile)

Key bits from your pipeline:
	•	Environment
	•	SCANNER_HOME = tool 'sonar-scanner'
	•	BUILD_VERSION = "v${BUILD_NUMBER}" → image tags like v34, v35, …
	•	Stages
	1.	Clean Workspace → cleanWs()
	2.	Git → Pull main branch of your repo
	3.	SonarQube → withSonarQubeEnv('sonar-server') ... sonar-scanner
	4.	NPM Install → npm install
	5.	OWASP + Trivy FS → dependency‑check & trivy fs .
	6.	Docker Build + Push → build/tag/push jabiralhujaifa/todo:${BUILD_VERSION}
	7.	Trivy Image Scan → trivy image ...
	8.	Docker Run (local) → quick container smoke check
	9.	Kubernetes → kubectl set image deployment/todo-app todo=...:${BUILD_VERSION}



That’s it!

regards,
jabir
