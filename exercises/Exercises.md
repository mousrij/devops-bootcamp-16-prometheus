## Exercises for Module 16 "Monitoring with Prometheus"
<br />

Use repository: https://gitlab.com/devops-bootcamp3/bootcamp-java-mysql/-/tree/feature/monitoring

**Context**

You and your team are running the following setup in the K8s cluster:

Java application that uses Mysql DB and is accessible from browser using Ingress. It's all running fine, but sometimes you have issues where Mysql DB is not accessible or Ingress has some issues and users can't access your Java application. And when this happens, you and your team spend a lot of time figuring out what the issue is and troubleshooting within the cluster. Also, most of the time when these issues happen, you are not even aware of them until an application user writes to the support team that the application isn't working or developers write you an email that things need to be fixed.

As an improvement, you have decided to increase visibility in your cluster to know immediately when such issues happen and proactively fix them. Also, you want a more efficient way to pinpoint the issues right away, without hours of troubleshooting. And maybe even prevent such issues from happening by staying alert to any possible issues and fixing them before they even happen.

Your manager suggested using Prometheus, since it's a well known tool with a large community and is widely used, especially in K8s environment.

So you and your team are super motivated to improve the application observability using Prometheus monitoring.

<details>
<summary>Exercise 1: Deploy your Application and Prepare the Setup</summary>
<br />

**Tasks:**

- Create a K8s cluster
- Deploy Mysql database for your Java application with 2 replicas (You can use the following helm chart: https://github.com/bitnami/charts/tree/master/bitnami/mysql)
- Deploy Java Maven application with 3 replicas that talks to the Mysql DB
- Deploy Nginx Ingress Controller (You can use the following helm chart: https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx)
- Now configure access to your Java application using an Ingress rule

You can use the Ansible playbook from Ansible exercises 7 & 8 with a few adjustments to configure this setup. 

**Solution:**



</details>

******