# 🚀 **Cloud-Native Monitoring App**  

A **Python-based monitoring application** built using **Flask** and **psutil**, containerized with **Docker**, and deployed on **AWS EKS** using Kubernetes.  

![Cloud-Native App](https://imgur.com/SWCzzy9.png)

---

## 🛠️ **Features**  

✅ **Real-time monitoring** using `psutil`  
✅ **Containerized with Docker**  
✅ **Deployed on AWS ECR & EKS**  
✅ **Automated Kubernetes Deployments**  
✅ **Infrastructure as Code (IaC) with Python**  

---

## 📌 **Prerequisites**  

Before you begin, ensure you have the following:  

- 🐍 **Python** installed → [Download Python](https://www.python.org/downloads/)  
- 🐳 **Docker** installed & running  
- ☁️ **AWS CLI** configured (`aws configure`)  
- 📦 **Kubernetes CLI (kubectl)** installed  
- 🛠️ **Python dependencies** installed:  

  ```bash
  pip3 install -r requirements.txt
  ```

---

## 🚀 **Step 1: Run the Application Locally**  

1️⃣ Navigate to the project folder:  

   ```bash
   cd Cloud-Native-Monitoring-App
   ```  

2️⃣ Start the Flask app:  

   ```bash
   python3 app.py
   ```  

3️⃣ Open in browser: [http://localhost:5000/](http://localhost:5000/)  

---

## 🐳 **Step 2: Dockerizing the Flask App**  

1️⃣ **Create a `Dockerfile`** in the root directory:  

   ```dockerfile
   # Use a lightweight Python base image
   FROM python:3.9-slim-buster

   # Set a non-root user for better security
   RUN addgroup --system appgroup && adduser --system --group appuser

   # Set the working directory
   WORKDIR /app

   # Copy dependencies file and install required packages
   COPY requirements.txt ./
   RUN pip3 install --no-cache-dir -r requirements.txt

   # Copy the rest of the application
   COPY . .

   # Change ownership to non-root user
   RUN chown -R appuser:appgroup /app

   # Switch to non-root user
   USER appuser

   # Set environment variables
   ENV FLASK_APP=app.py
   ENV FLASK_RUN_HOST=0.0.0.0
   ENV FLASK_ENV=production

   # Expose the application port
   EXPOSE 5000

   # Use gunicorn for better performance in production
   CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]
   ```  

2️⃣ **Build the Docker image**:  

   ```bash
   docker build -t my-flask-app .
   ```  

3️⃣ **Run the container**:  

   ```bash
   docker run -p 5000:5000 my-flask-app
   ```  

📌 Now, access the app at **[http://localhost:5000/](http://localhost:5000/)**.

---

## 📦 **Step 3: Push Docker Image to AWS ECR**  

1️⃣ **Create an ECR repository** using Python (`ecr.py`):  

```python
import boto3
import botocore

# AWS Session (Optional: Modify if using multiple profiles)
session = boto3.session.Session()
ecr_client = session.client('ecr')

# Define repository name
repository_name = "my_monitoring_app_image"

try:
    # Check if the repository already exists
    existing_repos = ecr_client.describe_repositories()
    repo_names = [repo['repositoryName'] for repo in existing_repos.get('repositories', [])]

    if repository_name in repo_names:
        print(f"✅ Repository '{repository_name}' already exists.")
    else:
        # Create a new ECR repository
        response = ecr_client.create_repository(repositoryName=repository_name)
        repository_uri = response['repository']['repositoryUri']
        print(f"🎉 Successfully created repository: {repository_name}")
        print(f"🔗 Repository URI: {repository_uri}")

except botocore.exceptions.ClientError as e:
    error_code = e.response['Error']['Code']
    if error_code == "RepositoryAlreadyExistsException":
        print(f"⚠️ Repository '{repository_name}' already exists.")
    else:
        print(f"❌ An error occurred: {e}")
```  

Run the script:  

```bash
python3 ecr.py
```  

2️⃣ **Push Docker image to ECR** (replace `<ECR_URI>` with your repo URI):  

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin <ECR_URI>
docker tag my-flask-app:latest <ECR_URI>:latest
docker push <ECR_URI>:latest
```

---

## ☁️ **Step 4: Deploy on AWS EKS**  

1️⃣ **Create an EKS cluster & node group** (`cloud-native-cluster`) via AWS Console.  
2️⃣ **Update kubeconfig**:  

```bash
aws eks update-kubeconfig --name cloud-native-cluster
```  

3️⃣ **Deploy the application using Python (`eks.py`)**:  

```python
import logging
from kubernetes import client, config

# Configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")

def load_kube_config():
    """Loads Kubernetes configuration (either local or in-cluster)."""
    try:
        config.load_kube_config()  # Use for local development
        logging.info("Loaded kubeconfig from local environment.")
    except Exception:
        config.load_incluster_config()  # Use when running inside a Kubernetes cluster
        logging.info("Loaded in-cluster Kubernetes configuration.")

def create_deployment(api_instance):
    """Creates a Kubernetes Deployment for the Flask app."""
    deployment = client.V1Deployment(
        metadata=client.V1ObjectMeta(name="my-flask-app"),
        spec=client.V1DeploymentSpec(
            replicas=1,
            selector=client.V1LabelSelector(match_labels={"app": "my-flask-app"}),
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(labels={"app": "my-flask-app"}),
                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name="my-flask-container",
                            image="568373317874.dkr.ecr.us-east-1.amazonaws.com/my_monitoring_app_image:latest",
                            ports=[client.V1ContainerPort(container_port=5000)]
                        )
                    ]
                )
            )
        )
    )

    try:
        api_instance.create_namespaced_deployment(namespace="default", body=deployment)
        logging.info("Deployment created successfully.")
    except Exception as e:
        logging.error(f"Failed to create deployment: {e}")

def create_service(api_instance):
    """Creates a Kubernetes Service for the Flask app."""
    service = client.V1Service(
        metadata=client.V1ObjectMeta(name="my-flask-service"),
        spec=client.V1ServiceSpec(
            selector={"app": "my-flask-app"},
            ports=[client.V1ServicePort(port=5000, target_port=5000)]
        )
    )

    try:
        api_instance.create_namespaced_service(namespace="default", body=service)
        logging.info("Service created successfully.")
    except Exception as e:
        logging.error(f"Failed to create service: {e}")

def main():
    """Main function to deploy the application."""
    load_kube_config()
    api_client = client.ApiClient()
    
    # Create Deployment and Service
    create_deployment(client.AppsV1Api(api_client))
    create_service(client.CoreV1Api(api_client))

if __name__ == "__main__":
    main()
```  

Run the script:  

```bash
python3 eks.py
```  

---

## 📡 **Step 5: Expose the App via Kubernetes Service**  

1️⃣ **Create `service.yaml`**:  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  labels:
    app: flask-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Use Network Load Balancer for better performance (AWS-specific)
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: LoadBalancer
```  

2️⃣ **Deploy the service**:  

   ```bash
   kubectl apply -f service.yaml
   ```  

3️⃣ **Get the service URL**:  

   ```bash
   kubectl get svc flask-service
   ```

---

## 🎯 Verify Deployment  

- Check deployments:  

  ```bash
  kubectl get deployments
  ```  

- Check pods:  

  ```bash
  kubectl get pods
  ```  

- Check services:  

  ```bash
  kubectl get svc
  ```  

- If needed, **edit the deployment**:  

  ```bash
  kubectl edit deployment my-flask-app
  ```  

- Expose the service (if LoadBalancer is not available):  

  ```bash
  kubectl port-forward service/flask-service 5000:5000
  ```  

  Now, access **[http://localhost:5000](http://localhost:5000/)** 🎉.

---

## 📌 **Conclusion**  

✅ **Python Flask Monitoring App** 🐍  
✅ **Dockerized & Hosted on AWS EKS** ☁️  
✅ **Automated Kubernetes Deployment** 🚀  
✅ **Scalable & Cloud-Native** 🔥  

🔹 **Enjoy Cloud-Native Observability!** 🚀  

---

### 🔗 **Want to Learn More?**

📖 Check out the [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html).  
📖 Explore Kubernetes [Official Docs](https://kubernetes.io/docs/home/).  

---

## 🤝 **Contributing**  

Contributions are welcome! If you'd like to improve this project, feel free to submit a pull request.  

---

## **Hit the Star!** ⭐

**If you find this repository helpful and plan to use it for learning, please give it a star. Your support is appreciated!**

---

## 🛠️ **Author & Community**  

This project is crafted by **[Haswanth](https://github.com/Haswanthkondamadugula)** 💡
I’d love to hear your feedback! Feel free to share your thoughts.  

