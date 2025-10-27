### Day 2 : Continuous Integration & Deployment (CI/CD)

Our goal is to automate the full lifecycle from code → build → container → deploy → observable service, using:

1. CI/CD (GitHub Actions) to build/test/deploy automatically
2. Registry (GitHub Container Registry (GHCR) to store container images
3. Deployment (Kubernetes) to run the app in a cluster
4. Observability (Jaeger + Prometheu) to keep traces & metrics flowing

#### Part 1 : Why CI/CD?

Without automation: developers manually build Docker images, push them, and kubectl-apply manifests → slow, error-prone.

With CI/CD: every merge to main triggers a pipeline that:
	1.	Runs tests
	2.	Builds the Docker image
	3.	Pushes it to GHCR
	4.	Deploys automatically to Kubernetes
	5.	Keeps observability and health checks intact

Your platform team’s mission is to make this entire chain effortless for developers.

#### Part 2 — Project Prep

Make sure your repo has these at the root:

```
.
├── app/
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── Dockerfile
├── requirements.txt
└── .github/
    └── workflows/
        └── ci-cd.yaml
```

#### Part 3 — Set Up GitHub Container Registry (GHCR)

 1. Log in to GHCR locally (to test):  `echo $GITHUB_TOKEN | docker login ghcr.io -u magsther --password-stdin`
 2. Enable GHCR for your account
  	•	Go to https://github.com/settings/packages
  	•	Confirm “Allow GitHub Actions to push and pull packages”

#### Part 4 — Create the GitHub Actions Workflow

1. Create /.github/workflows/ci-cd.yaml

   ```
   name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}:latest

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest || echo "no tests yet"

      - name: Build Docker image
        run: docker build -t $IMAGE_NAME .

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push image
        run: docker push $IMAGE_NAME

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
    ```

    **Secrets Needed**
    Add via GitHub Repo → Settings → Secrets → Actions.
    
    KUBECONFIG or KUBE_CONFIG_DATA Base64-encoded kubeconfig for your Kind/AKS cluster


#### Part 5 — Update Kubernetes Manifests

1. Create k8s/deployment.yaml should reference GHCR image:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fastapi-demo
  template:
    metadata:
      labels:
        app: fastapi-demo
    spec:
      containers:
        - name: fastapi-demo
          image: ghcr.io/magsther/platforming:latest
          ports:
            - containerPort: 8080
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4318/v1/traces"
```


2. k8s/service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: fastapi-demo
spec:
  selector:
    app: fastapi-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```


#### Part 6 — Test CI/CD End-to-End

	1.	Commit everything:

  ```
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

	2.	Go to GitHub → Actions → watch the workflow run.
	3.	When complete:
  	•	Check GHCR → package platforming
  	•	Run locally to confirm image works: `docker run -p 8080:8080 ghcr.io/magsther/platforming:latest`

  4.	Deploy to your Kind cluster manually to validate:

     ```
     kubectl apply -f k8s/
     kubectl get pods
    ```

	5.	Open port forward to test:
  ```
kubectl port-forward svc/fastapi-demo 8080:80
curl localhost:8080/checkout
```

 If that works, your pipeline is now fully automated.

#### Part 7 — Observability Check-In

Since your app already exports metrics + traces:
	•	After each deploy, you can confirm observability still works:
	•	Visit http://localhost:16686 → traces
	•	Visit Prometheus or Grafana → metrics

You’ve now automated not just builds, but the observability lifecycle.

###  Day 2 Outcome

1. Automated build & test on push
2. Container pushed to GHCR
3. Automated Kubernetes deployment
4. Observability persists post-deploy



