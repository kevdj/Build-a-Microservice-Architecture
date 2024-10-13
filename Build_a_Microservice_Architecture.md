
# Build a Microservice Architecture

**Services used**: ECS, SQS, CloudWatch, Cloud9

## 1. Create the Service Code

We are going to be using 2 services, **ServiceA** and **ServiceB**. Below is the necessary code.

### Service A:
Generates random messages and sends them to SQS every 5 seconds.

```python
# producer_service.py
import boto3
import time
import random

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT_ID/your-queue-name'

def send_message():
    message = f"Hello from Service A {random.randint(1, 100)}"
    sqs.send_message(QueueUrl=queue_url, MessageBody=message)
    print(f"Sent message: {message}")

if __name__ == "__main__":
    while True:
        send_message()
        time.sleep(5)  # Produce a message every 5 seconds
```

### Service B:
Polls messages from the SQS queue every 2 seconds and deletes them from the queue.

```python
# consumer_service.py
import boto3
import time

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT_ID/your-queue-name'

def receive_messages():
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=1,
        WaitTimeSeconds=10,
    )

    messages = response.get('Messages', [])
    if messages:
        for message in messages:
            print(f"Received message: {message['Body']}")
            sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=message['ReceiptHandle'])

if __name__ == "__main__":
    while True:
        receive_messages()
        time.sleep(2)  # Poll the queue every 2 seconds
```

## 2. Create Docker Files Using Cloud9 IDE

In Cloud9, create a new environment and make two folders for each service: **ServiceA** and **ServiceB**.

Add the service files to each folder, and then create a `Dockerfile` in each.

![Cloud9 Directory](./Cloud9.png)

### Dockerfile for Service A

```dockerfile
# Use a lightweight Python image
FROM python:3.8-slim

# Copy your code to the container
COPY producer_service.py /app/producer_service.py

# Install dependencies
RUN pip install boto3

# Set the command to run the service
CMD ["python", "/app/producer_service.py"]
```

### Dockerfile for Service B

```dockerfile
# Use a lightweight Python image
FROM python:3.8-slim

# Install dependencies
RUN pip install boto3

# Copy your code to the container
COPY consumer_service.py /app/consumer_service.py

# Set the command to run the service
CMD ["python", "/app/consumer_service.py"]
```

Run the following commands to build the images:

```bash
cd serviceA
docker build -t service-a-producer .

cd serviceB
docker build -t service-b-consumer .
```

**Authenticate the Docker client to the Amazon ECR registry to push the image:**

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 139406489149.dkr.ecr.us-east-1.amazonaws.com
```

Check the created Docker images:

```bash
docker images
```

Tag your images:

```bash
docker tag service-a-producer:latest 139406489149.dkr.ecr.us-east-1.amazonaws.com/service-a-producer:latest
docker tag service-b-consumer:latest 139406489149.dkr.ecr.us-east-1.amazonaws.com/service-b-consumer:latest
```

Push the images to ECR:

```bash
docker push 139406489149.dkr.ecr.us-east-1.amazonaws.com/service-a-producer:latest
docker push 139406489149.dkr.ecr.us-east-1.amazonaws.com/service-b-consumer:latest
```

## 3. Create a Cluster, Task, and Task Definition in ECS

The next steps involve the creation of a Cluster, Task, Task Definition, and deploying your services on AWS ECS. Follow these detailed steps:

1. **Go to the ECS Console and Create a New Cluster:**

   - Navigate to the [Amazon ECS Console](https://console.aws.amazon.com/ecs/home).
   - Click on **"Clusters"** on the left panel and select **"Create Cluster"**.
   - Name your cluster (e.g., `Microservices-Cluster`).
   - For **Cluster Template**, select **Fargate**, which is a serverless compute engine for containers.
   - Proceed with the rest of the settings as default, then click **"Create"**.

2. **Create a Task Definition:**

   A Task Definition is like a blueprint for your Docker container, specifying the image, CPU/memory requirements, networking, and environment variables.

   - In the ECS Console, navigate to **"Task Definitions"** and click on **"Create New Task Definition"**.
   - Select **Fargate** as the **Launch Type**.
   - For **Task Role**, choose a role that has the necessary permissions to interact with AWS services (like SQS). If you don’t have one, click **"Create New Role"** to create one automatically.
   - **Network Mode**: Choose **awsvpc**. This gives each task its own network interface, providing full network isolation.
   - In the **Task Execution Role**, choose an appropriate role or create a new one with the required permissions to access ECS services and resources.
   - **Container Definitions**: 
     - Define the URI for your Docker image that you uploaded to Amazon ECR (Elastic Container Registry).
     - **Environment Variables**: Add your environment variables here.
       - For **Service A**:
         - Key: `QUEUE_URL`
         - Value: The SQS queue URL for **Service A**.
       - For **Service B**:
         - Key: `QUEUE_URL`
         - Value: The SQS queue URL for **Service B**.
   - **Logging Configuration**: Enable logging by selecting **"Enable CloudWatch Logs"** and specify the log group name for your service.

   Repeat these steps for both **Service A** and **Service B**.

3. **Deploy and Run Tasks for Service A and Service B:**

   - After defining the task, go to your **ECS Cluster** and click on **"Run Task"**.
   - Select **Fargate** as the launch type.
   - Select **Service** as the application type since we want the services to be constantly running.
   - Choose the Task Definition you created for **Service A** and select the **latest revision**.
   - Keep other configurations as default unless you have specific requirements for your setup.
   - Click **"Create"** to run the task.

   Repeat the same process for **Service B** using the task definition created for **Service B**.

### Possible Issues and Troubleshooting:

- **Failed Task Creation**: If the task fails to start, double-check that the SQS URL in the environment variables is correct. An incorrect URL can cause the task to fail during execution.
  
- **Logging and Monitoring**: After the task is successfully created and running, navigate to **CloudWatch Logs** to monitor the log output from both Service A and Service B.

## Monitoring CloudWatch Logs

- **Service A (Producer)** logs messages sent:
  
  ![Producer Logs](./Producer.png)

- **Service B (Consumer)** logs messages received:

  ![Consumer Logs](./Consumer.png)

---

**Note**: You can view the logs by selecting each task from your cluster and checking the logs section.

![CloudWatch Logs](./Screenshot_2024-10-12_161518.png)
