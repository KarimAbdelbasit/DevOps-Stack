# Project Name: Dockerized Web Application with Jenkins, Ansible, and Kubernetes

## Overview
This project demonstrates the integration of Docker, Jenkins, Ansible, and Kubernetes for automating the deployment of a web application across multiple servers. The pipeline executes a series of stages, including Git checkout, building Docker images, pushing them to Docker Hub, and deploying to Kubernetes. Additionally, the project is configured with a Webhook for automated triggers.

## Project Structure
The project consists of the following key files:

- **Dockerfile**: Defines the Docker image for the web application.
- **ansible-playbook.yml**: Ansible playbook for managing deployment.
- **Deployment.yml**: Configuration for deployment settings in Kubernetes.
- **Service.yml**: Kubernetes service configuration.
- **spring-html**: Additional resources for the application.

## Architecture
The project utilizes three servers:
1. **Jenkins Server**: Handles CI/CD pipeline execution.
2. **Ansible Server**: Manages deployment automation.
3. **Kubernetes Server**: Orchestrates containerized applications.

## Webhook Configuration
The project includes a Webhook configured to trigger the Jenkins pipeline automatically upon changes in the GitHub repository. This allows for seamless and continuous deployment whenever updates are pushed.

## Jenkins Pipeline
The Jenkins pipeline is defined in the following stages:

```groovy
pipeline {
    agent any

    stages {
        stage('Git checkout') {
            steps {
                git 'https://github.com/KarimAbdelbasit/DevOps-Stack.git'
            }
        }
        
        stage('send file to ansible') {
            steps {
                sshagent(['ssh-agent']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118'
                    sh 'scp -o StrictHostKeyChecking=no -r /var/lib/jenkins/workspace/pipeline-myproject/* ubuntu@172.31.37.118:/home/ubuntu/'
                }
            }
        }
        
        stage('Docker build image') {
            steps {
                sshagent(['ssh-agent']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 "cd /home/ubuntu/ && docker build -t $JOB_NAME:V1.$BULD_ID ."
                    '''
                }
            }
        }
        stage('Docker build image tagging'){
            steps {
                sshagent(['ssh-agent']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 cd /home/ubuntu/'
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 docker image tag $JOB_NAME:V1.$BULD_ID karimabdelbasit/$JOB_NAME:V1.$BULD_ID'
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 docker image tag $JOB_NAME:V1.$BULD_ID karimabdelbasit/$JOB_NAME:latest'
                }
            }
        }
        stage('login') {
            steps {
                sshagent(['ssh-agent']) {
                    withCredentials([string(credentialsId: 'docker_password', variable: 'dockerhub_password')]) {
                        sh '''
                            echo $dockerhub_password | docker login -u karimabdelbasit --password-stdin
                        '''
                    }
                }
            }
        }
        stage('push images to docker hub') {
            steps {
                sshagent(['ssh-agent']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 cd /home/ubuntu/'
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 docker image push karimabdelbasit/$JOB_NAME:V1.$BULD_ID'
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 docker image push karimabdelbasit/$JOB_NAME:latest'
                }
            }
        } 
        
        stage('send file from ansible to kubernetes'){
            steps {
                sshagent(['kubernetes_server']) {
                    sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.17.98 "echo Connected"'
                    sh 'scp -o StrictHostKeyChecking=no -r /var/lib/jenkins/workspace/pipeline-myproject/* ubuntu@172.31.17.98:/home/ubuntu/'
                }
            }
        }
        stage('Kubernetes Deployment using ansible') {
            steps {
                sshagent(['ssh-agent']) {
                    sh '''
                      ssh -o StrictHostKeyChecking=no ubuntu@172.31.37.118 "cd /home/ubuntu/ && ansible-playbook ansible-playbook.yml"
                    '''
                }
            }
        }
    }
}
