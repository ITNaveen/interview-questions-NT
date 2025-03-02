Python for DevOps: Practical Guide

This guide explains where to write DevOps Python scripts, how to execute them (automated or manual), and their use case in a real-world scenario.

# 1. Jenkins Job Monitor (Check Build Status & Logs)

📌 What Does This Script Do?

Monitors a Jenkins job status (SUCCESS/FAILURE).

If the job fails, it fetches the last 500 lines of logs.

🏗 Where to Put This Script?

Inside Jenkins Pipeline (Jenkinsfile) as a new stage.

🔧 How to Automate It?

Add the script inside your Jenkinsfile so that it runs automatically after every build.

📝 Jenkinsfile Example:

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                // Your build steps here
            }
        }
        
        stage('Monitor Job') {
            steps {
                script {
                    sh 'python jenkins_monitor.py'
                }
            }
        }
    }
}

📝 Python Script (jenkins_monitor.py):

import requests

JENKINS_URL = "http://your-jenkins-url"
JOB_NAME = "your-job-name"
USER = "your-username"
TOKEN = "your-api-token"

# Fetch job status
build_url = f"{JENKINS_URL}/job/{JOB_NAME}/lastBuild/api/json"
response = requests.get(build_url, auth=(USER, TOKEN)).json()

# Print job result
print(f"Job Status: {response['result']}")

# If failed, fetch logs
if response["result"] == "FAILURE":
    log_url = f"{JENKINS_URL}/job/{JOB_NAME}/lastBuild/consoleText"
    logs = requests.get(log_url, auth=(USER, TOKEN)).text
    print("==== Last 500 Lines of Logs ====")
    print(logs[-500:])


# 2. AWS EC2 Health Check (Check Running Instances)

📌 What Does This Script Do?

Checks if EC2 instances are running or stopped.

Can be automated via cron job or AWS Lambda.

🏗 Where to Put This Script?

On a Jenkins server OR as a cron job on a monitoring instance.

🔧 How to Automate It?

Option 1: Add to a Jenkins pipeline.

Option 2: Run as a cron job on a monitoring instance every 10 minutes.

📝 Python Script (ec2_health.py):

import boto3

ec2 = boto3.client("ec2")
instances = ec2.describe_instance_status(IncludeAllInstances=True)

for instance in instances["InstanceStatuses"]:
    id = instance["InstanceId"]
    state = instance["InstanceState"]["Name"]
    print(f"Instance {id} is {state}")

⏳ Automate with Cron Job (Run Every 10 min):

Run crontab -e and add:

*/10 * * * * python /path/to/ec2_health.py

# 3. Kubernetes Auto-Restart for Failing Pods

📌 What Does This Script Do?

Detects failing Kubernetes pods and restarts them.

🏗 Where to Put This Script?

On a Kubernetes worker node.

🔧 How to Automate It?

Schedule a Kubernetes CronJob to run this script every 5 minutes.

📝 Python Script (k8s_restart.py):

from kubernetes import client, config

config.load_kube_config()
v1 = client.CoreV1Api()

pods = v1.list_namespaced_pod(namespace="default")

for pod in pods.items:
    name = pod.metadata.name
    state = pod.status.phase
    
    if state != "Running":
        print(f"Restarting pod: {name}")
        v1.delete_namespaced_pod(name=name, namespace="default")

⏳ Automate with Kubernetes CronJob:

Apply this YAML file:

apiVersion: batch/v1
kind: CronJob
metadata:
  name: restart-failed-pods
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: restart-failed-pods
            image: python:3.8
            command: ["python", "k8s_restart.py"]
          restartPolicy: OnFailure

# 4. Log Analysis (Detect Errors in Logs)

📌 What Does This Script Do?

Reads log files and detects errors (ERROR, FAIL, EXCEPTION).

🏗 Where to Put This Script?

On a log server.

🔧 How to Automate It?

Set up a cron job to check logs every hour.

📝 Python Script (log_checker.py):

LOG_FILE = "/var/log/app.log"
with open(LOG_FILE, "r") as file:
    logs = file.readlines()
error_count = sum(1 for line in logs if "ERROR" in line)
print(f"Found {error_count} errors in logs.")

# 5. AWS Cost Cleanup (Find Unused Resources)

📌 What Does This Script Do?

Finds stopped EC2 instances and unused EBS volumes to save cost.

🏗 Where to Put This Script?

On a monitoring instance.

🔧 How to Automate It?

Set up a weekly cron job.

# 6. Config File Update (Modify Configs Before Deployment)

📌 What Does This Script Do?

Reads a config.json file and updates database settings.

🏗 Where to Put This Script?

On a deployment server.

🔧 How to Automate It?

Run before deploying an application.

📝 Python Script (config_update.py):

import json
with open("config.json", "r") as file:
    config = json.load(file)
config["database"]["host"] = "new-db.example.com"
with open("config.json", "w") as file:
    json.dump(config, file, indent=4)
print("✅ Config updated!")

Final Thoughts: How to Use These Scripts?

Jenkins tasks run inside Jenkinsfile.

EC2 checks are automated with cron jobs.

Kubernetes tasks run inside Kubernetes CronJobs.

Log analysis is automated with a cron job.

Config file updates are part of the deployment process.

