# Gitlab Kubernetes Automation.

<details>

<summary>Gitlab Primer(Why use Gitlab?)</summary>

# What is Gitlab?

GitLab is a web-based DevOps lifecycle tool that provides a Git repository manager, wiki, issue tracking, and continuous integration/continuous deployment (CI/CD) pipeline features, using an open-source license. Here are the main aspects that set GitLab apart from other CI/CD solutions:

## Key Features of GitLab
1. **Complete DevOps Platform**: GitLab offers a full DevOps platform delivered as a single application, providing all the necessary tools for the entire software development lifecycle, from planning to monitoring.

2. **Integrated Source Control**: GitLab integrates Git repository management, enabling efficient code collaboration, version control, and project management.

3. **CI/CD Capabilities**: GitLab CI/CD allows you to automate the building, testing, and deploying of your code with its robust pipeline configuration, using a .gitlab-ci.yml file.

4. **Issue Tracking and Project Management**: GitLab includes issue tracking, project planning, and agile management tools such as milestones, boards, and burndown charts.

5. **Container Registry**: GitLab provides a built-in Docker container registry for storing and managing Docker images.

6. **Security Features**: GitLab includes various security features like code quality analysis, container scanning, dependency scanning, and dynamic application security testing (DAST).

7. **Auto DevOps**: GitLab’s Auto DevOps feature automates the software delivery process, from code commit to production deployment.

8. **Self-Managed and SaaS Options**: GitLab can be used as a self-hosted solution (GitLab Community Edition or GitLab Enterprise Edition) or as a SaaS service (GitLab.com).

## Differences from Other CI/CD Solutions
1. **All-in-One Platform**: Unlike many CI/CD solutions that specialize in a particular aspect of the DevOps lifecycle (e.g., Jenkins for CI/CD, Jira for issue tracking), GitLab offers a comprehensive suite of tools in a single application.

2. **GitLab Runner**: GitLab CI/CD uses GitLab Runner to run jobs and send the results back to GitLab, allowing for flexible and distributed builds.

3. **Native Git Integration**: GitLab's seamless integration with Git repositories eliminates the need for external version control systems and makes it easier to manage code and pipelines in one place.

4. **Open Source**: GitLab Community Edition is open source, providing transparency, community contributions, and the ability to customize and extend the platform.

5. **Security and Compliance**: GitLab integrates various security and compliance features directly into the CI/CD pipelines, enabling continuous security and compliance monitoring.

6. **Ease of Use**: GitLab's UI is designed to be user-friendly, making it easier for teams to adopt and use the platform without extensive training.

7. **Collaboration Features**: GitLab emphasizes collaboration with features like merge requests, code reviews, and real-time editing, enhancing team productivity and communication.

8. **Scalability**: GitLab is designed to scale with the needs of both small teams and large enterprises, offering features like clustering and high availability in its self-hosted options.

In summary, GitLab's all-in-one approach, integrated features, open-source nature, and focus on collaboration and security set it apart from other CI/CD solutions in the market.

</details>

# Gitlab Installation 

Gitlab can be installed on a variety of OS. It can also be installed as a Microservice or a Monolith. They also offer a SaaS solution in case you dont prefer a self managed one.

Below are the steps to install Gitlab on a monolithic style on a RHEL system.

For in depth pre-requisite analysis, Please visit https://docs.gitlab.com/ee/install/

## Installation Steps

To install GitLab on a Red Hat Enterprise Linux (RHEL) system, follow these steps:

1. **Update your system.**
```
sudo yum update -y
```
2. **Install Dependencies.**
```
sudo yum install -y curl policycoreutils-python openssh-server perl
sudo systemctl enable sshd
sudo systemctl start sshd
```
3. **Install Postfix for email notifications:**
```
sudo yum install -y postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```
4. **Add the GitLab Repository and Install GitLab**
```
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash;
sudo EXTERNAL_URL="http://gitlab.example.com" yum install -y gitlab-ee
```
5. **Reconfigure Gitlab**
```
sudo gitlab-ctl reconfigure
```
6. **Open the required firewall ports(in case firewall is enabled):**
```
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

## Connect Gitlab to Kubernetes
1. **Install the GitLab Agent on Kubernetes:**

- First, create a GitLab project if you haven't already.
- In your GitLab project, go to **Infrastructure** > **Kubernetes clusters** > **Connect a cluster** > **Agent**.
- Follow the instructions to create an agent configuration file.
2. **Create a configuration file for the GitLab Agent:**

- In the GitLab project, create a file named .gitlab/agents/my-agent/config.yaml with the following content:
```
ci_access:
  projects:
    - id: my-group/my-project
```
Above YAML snippet grants CI access to project named **my-project** in group **my-group** 

3. **Apply GitLab Agent config in Kubernetes:**

- Follow the provided instructions to install the GitLab Agent. This usually involves running a kubectl command like:
```
kubectl apply -f https://gitlab.example.com/my-group/my-project/raw/main/.gitlab/agents/my-agent/config.yaml
```
4. **Configure Kubernetes Cluster in GitLab:**
- Go to **Infrastructure** > **Kubernetes clusters** in your GitLab project.
- Click **Connect** existing cluster and fill in the necessary details, including the **API URL** and **token**.

5. **Set up GitLab Runner on Kubernetes** (optional but recommended):
```
helm repo add gitlab https://charts.gitlab.io
helm repo update
helm install --namespace gitlab-runner gitlab-runner gitlab/gitlab-runner \
  --set gitlabUrl=http://gitlab.example.com/ \
  --set runnerRegistrationToken=YOUR_REGISTRATION_TOKEN
```
6. **Verify Connection**
- **Access GitLab**: Open your web browser and go to the URL where GitLab is hosted.
- **Log in to GitLab**: Use the default root account (root user and the password you set during installation) to log in.
- **Navigate to Infrastructure** > **Kubernetes clusters** and verify the status of your connected cluster.

## Demo Gitlab Pipeline to build and deploy containers to Kubernetes Clusters

1. Create a **.gitlab-ci.yml** file in your project repository to define the CI/CD pipeline.

2. **Deploy to Kubernetes**: Use the GitLab CI/CD pipeline to deploy applications to your connected Kubernetes cluster.

Here is a simple example of a **.gitlab-ci.yml** file to deploy an application to a Kubernetes cluster:
```YAML
image: docker:latest

services:
  - docker:dind

variables:
  KUBERNETES_NAMESPACE: your-namespace
  KUBERNETES_SERVER: https://your-kubernetes-api-url
  KUBERNETES_TOKEN: your-kubernetes-token
  KUBERNETES_CA_CERTIFICATE: your-ca-certificate

stages:
  - build
  - deploy

build:
  stage: build
  script:
    - echo "Building the application..."
    - docker build -t your-image-name:latest .

deploy:
  stage: deploy
  script:
    - echo "Deploying to Kubernetes..."
    - kubectl config set-cluster k8s-cluster --server=$KUBERNETES_SERVER --certificate-authority=$KUBERNETES_CA_CERTIFICATE
    - kubectl config set-credentials deployer --token=$KUBERNETES_TOKEN
    - kubectl config set-context default --cluster=k8s-cluster --user=deployer --namespace=$KUBERNETES_NAMESPACE
    - kubectl config use-context default
    - kubectl apply -f kubernetes/deployment.yaml

```
