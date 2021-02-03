# CICD-Microservices

# Jenkins CI/CD pipeline for Microservices Deployment on Kubernetes

### Introduction
- The objective behind this guide is to help set up a simple and efficient CI/CD pipeline using Jenkins for hundreds of microservices running in Kubernetes on AWS EKS.
- The approach will be developer-friendly and straightforward, easy to maintain, and one that has proved to be effective on some large scale production deployments. Step by step guide for the entire setup is listed below

## Jenkins Installation
- To start with, we need Jenkins running on an EC2 instance. [ Running Jenkins inside the Kubernetes cluster has its own set of challenges to solve for docker builds ].
- The Internet is flooded with articles on How to install Jenkins, But it is recommended to follow the official. Some basic requirements also need to be fulfilled before this setup is ready for use.

    - Docker
    - AWS CLI
    - Helm 3

in order to get the setup up and running are:
- Slack Notification Plugin
- Role-based Authorization Strategy
- Authorize Project (Configure projects to run with specific authorization.)
- Amazon ECR plugin

One critical aspect is Jenkins’s authentication with AWS services like ECR and EKS. Always prefer to use IAM instance Role [ This role ARN also needs to be added to AWS-auth configmap in EKS cluster ]. Here is a list of IAM policies to be attached to IAM role

- AmazonEKSClusterPolicy
- AmazonEKSServicePolicy
- AmazonEKSWorkerNodePolicy
- ECRFullAccess
- EksClusterAutoscalerPolicy

# Git flow and Branching strategy
This article refers to a simple branching strategy, with 3 mainline branches mapped to 3 environments running in the Kubernetes cluster. Each environment is running in it’s Kubernetes namespace.

git-flow for deployment stages :

![](https://miro.medium.com/max/607/1*_l-JGFQlxQ6lt634_BRkYg.png)

## Build Instructions :
In a typical microservices-based architecture, each microservice can be developed in a different programming language or framework. This puts Docker to it’s best use case. All the build instructions for a particular microservice stay in Dockerfile placed at the root of each repository.

## Deployment Instructions
The deployments are packaged as helm charts that define all aspects of a microservice like — container images, placement constraints, networking, storage, configurations, etc.

These helm charts are stored in a separate git repository, which includes per service helm chart as well as other resources like common configmaps and secrets, ingress and different routing configuration, etc. These are fetched on Jenkins server as a dedicated Jenkins job with webhook configured to fetch updates automatically.

## Pipeline — Putting it all together
Consider a simple pipeline having the following stages.
1. Source
2. Build and Test
3. Artifact
4. Deploy
5. Cleanup ( Crucial part of the pipeline )
6. Notify

Since the environment is independent of the EKS cluster name, it should be decoupled. 

Hence we configure cluster names corresponding to each environment as Jenkins Global Environment Variables in Manage Jenkins→Configure System → Global properties → Environment variables. These environments are being fetched from Git branch names like if the BRANCH is under development the ENV will be DEV i.e. ENV=${GIT_BRANCH}_env

![](https://miro.medium.com/max/399/1*_dwiSi9a2e-50Yb3aDfdmg.png)

Global environment variables for branch and cluster settings per CI stage

Slack Configuration for build notifications:

![](https://miro.medium.com/max/538/1*L4KJuS5KdvXlcP5jqM6FCg.png)

### Git credentials: 
To setup Github webhooks, make sure Github Plugin is installed in Jenkins. Go to your Github repo -> Settings -> Webhooks . Add public URL of Jenkins as Payload URL, it will tell Github where to send the webhooks as below:

https://jenkins.example.com/github-webhook/

Now add Jenkins SSH public key to source code repositories. Add private ssh key to Jenkins credentials store and note the ID of the credential.

![](https://miro.medium.com/max/700/1*ZJTxinq2qw7CV3k8gKxnoQ.png)

### Go to Credentials → System → Global Credentials
Add Credentials of the Kind using the SSH Username with Private key ( for the public key added to GitHub )

![](https://miro.medium.com/max/700/1*WP6vt2bgohPmRRqLT0a8yg.png)

Jenkins file example with GitHub webhooks and slack notification.

```shell
def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
properties([pipelineTriggers([githubPush()])])
pipeline {
  agent any  
  environment {
    //put your environment variables
    doError = '0'
    DOCKER_REPO = "12345678.dkr.ecr.ap-south-1.amazonaws.com/${JOB_NAME}"
    AWS_DEFAULT_REGION = "ap-south-1"
    CHART_DIR="$JENKINS_HOME/workspace/helm-integration/helm"
    HELM_RELEASE_NAME = "api-service"
    ENV= """${sh(
  		returnStdout: true,
  		script: 'declare -n ENV=${GIT_BRANCH}_env ; echo "$ENV"'
    ).trim()}"""
  }
    options {
        buildDiscarder(logRotator(numToKeepStr: '20')) 
  }   
  stages {
    stage ('Build and Test') {
      steps {
        sh '''
        docker build \
        -t ${DOCKER_REPO}:${BUILD_NUMBER} .
        #put your Test cases
        echo 'Starting test cases'
        '''    
      }
    }  
    stage ('Artefact') {
      steps {
        sh '''
        $(aws ecr get-login --region ap-south-1 --no-include-email)
        docker push ${DOCKER_REPO}:${BUILD_NUMBER}
        '''
        }
    }   
    stage ('Deploy') {
      steps {
        sh '''
        declare -n CLUSTER_NAME=${ENV}_cluster
        aws eks --region $AWS_DEFAULT_REGION  update-kubeconfig --name ${CLUSTER_NAME}
        kubectl config set-context --current --namespace=$ENV
        helm upgrade --install ${HELM_RELEASE_NAME} ${CHART_DIR}/${HELM_RELEASE_NAME}/ \
        --set image.repository=${DOCKER_REPO} \
        --set image.tag=${BUILD_NUMBER} \
        --set environment=-${ENV} \
        -f ${CHART_DIR}/${HELM_RELEASE_NAME}/values.yaml \
        --namespace ${ENV}
        '''
      }
    }
    stage('Cleanup') {
      steps{
        sh "docker rmi ${DOCKER_REPO}:${BUILD_NUMBER}"
      }
  }
// slack notification configuration 
  stage('Error') {
    // when doError is equal to 1, return an error
    when {
        expression { doError == '1' }
    }
    steps {
        echo "Failure :("
        error "Test failed on purpose, doError == str(1)"
    }
}
  stage('Success') {
    // when doError is equal to 0, just print a simple message
    when {
        expression { doError == '0' }
    }
    steps {
        echo "Success :)"
    }
}
  }
    // Post-build actions
  post {
      always {
          slackSend channel: '#jenkins-notification',
              color: COLOR_MAP[currentBuild.currentResult],
              message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} More info at: $RUN_DISPLAY_URL"
      }
  }
}
```
This Jenkinsfile is independent of microservice or environment. You can have a single Jenkins file for all microservices or can use separate Jenkins files stored in git for every microservice.

Let’s configure a Multibranch Pipeline Jenkins job to use the above-mentioned pipeline file

# Creating a multibranch Pipeline
Create New Item and Choose Multibranch Pipeline
![](https://miro.medium.com/max/700/1*lht8UDuP3pAKrlG5bQ6zUA.png)

In Branch Source, update your Git repository and credential, then you can keep the default behaviour for auto discover branches, or you can filter branches by name.

![](https://miro.medium.com/max/700/1*4PfsuP7m4mm3hsSZR_Z4Rw.png)

In Build Configuration, use Jenkinsfile and update your file path.

In Scan Multibranch Pipeline Triggers, choose Scan by webhook and setup Trigger token or use periodically. In the case of Scan by webhook Multibranch Scan Webhook Trigger plugin is required.

![](https://miro.medium.com/max/700/1*qxMINvbxqcz79CxdNINA2w.png)

Save your configuration and then Click Open Blue Ocean, it would auto-discover your Github branch and the pipeline you defined in Jenkinsfile

![](https://miro.medium.com/max/700/1*JeIU9cULcnuyTiS00WitBA.png)

After making a commit to a specific branch, you should see the pipeline initiation of that branch, followed by the final deployment.

![](https://miro.medium.com/max/700/1*_oJfXU_shPtCvzrgJTr5xQ.png)

The Jenkins job is now ready for use. Replicate the same flow across all microservices.

# Jenkins Access control and Management

Install Jenkins role-based access control plugin to enable Manage and Assign Role option under Manage Jenkins

![](https://miro.medium.com/max/548/1*nwFKlxxxjiALj_tpUt91tA.png)

Add new roles under Manage Roles

![](https://miro.medium.com/max/700/1*xzWWyEhV8fwE1cDnDBV0Ug.png)

And assign roles to users 

Now you should be able to build and deploy your own pipelines using Jenkins. This will be your first step at establishing a continuous integration and continuous deployment cycle and incorporating a DevOps culture into your workflow. This method ensures efficient management of AWS & microservices deployed on AWS EKS

