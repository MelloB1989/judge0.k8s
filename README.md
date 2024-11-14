# Judge0 on AKS Cluster Setup

This project demonstrates how to deploy **Judge0** on a Kubernetes (K8s) cluster running on **Azure Kubernetes Service (AKS)**. It highlights the challenges encountered when using Judge0, which requires **cgroup v1**, and how we worked around these challenges to get Judge0 running on the cluster.

## Table of Contents

- [Introduction](#introduction)
- [What is cgroup and why is it necessary?](#what-is-cgroup-and-why-is-it-necessary)
- [Deploying Judge0 on AKS](#deploying-judge0-on-aks)
- [Configuring cgroup v1 on AKS](#configuring-cgroup-v1-on-aks)
    - [Option 1: SSH Access to Node](#option-1-ssh-access-to-node)
    - [Option 2: Using a DaemonSet to Revert cgroup](#option-2-using-a-daemonset-to-revert-cgroup)

## Introduction

Judge0 is an open-source **online code execution API** that runs code in various languages. When deploying Judge0 on Kubernetes (AKS in this case), we encountered a significant issue: Judge0 requires **cgroup v1** to function correctly, but by default, modern systems like AKS use **cgroup v2**. This README outlines how we resolved this issue and successfully deployed Judge0 on AKS. These are a few screenshots of me running the Judge0 APIs on Bruno, PS: it's not running on my local machine, I port-forwarded to my pod as I did not want the endpoint to be public, `kubectl port-forward judge0-server-55cff5cf7b-8k5cb 8080:2358 --namespace judge0`

![Judge0 Error Screenshot](https://github.com/MelloB1989/judge0.k8s/blob/main/screenshots/languages.png?raw=true)  
*Successfully running Languages API*

![Judge0 Error Screenshot](https://github.com/MelloB1989/judge0.k8s/blob/main/screenshots/submissions.png?raw=true)  
*Successfully running submissions API*

![Judge0 Error Screenshot](https://github.com/MelloB1989/judge0.k8s/blob/main/screenshots/error.png?raw=true)  
*Error when trying to run Judge0 due to cgroup v2*

## What is cgroup and why is it necessary?

**Control Groups (cgroups)** are a Linux kernel feature that allows for the allocation of resources (CPU, memory, I/O, etc.) to processes or containers. cgroups ensure that system resources are allocated properly, preventing one process from overusing resources and affecting others.

Judge0 uses cgroups to manage the resources for the code execution containers. It requires **cgroup v1** for its operations. If the system is using **cgroup v2**, Judge0 will not function correctly, leading to issues like failing to start or improperly managing resources.

![Cgroup Version Screenshot]([<URL_TO_CGROUP_VERSION_SCREENSHOT>](https://github.com/MelloB1989/judge0.k8s/blob/main/screenshots/check.png?raw=true))  
*Checking cgroup version in the system*

## Deploying Judge0 on AKS

1. **Cluster Setup**: We deployed Judge0 on a Kubernetes cluster running on **Azure AKS**. The application was built using **Helm charts** and includes **Redis**, **PostgreSQL**, and other dependencies.
   
2. **Helm Chart Customization**: The default Helm chart was modified to deploy Judge0 on AKS with the required environment variables, such as database credentials and the Redis host.

## Configuring cgroup v1 on AKS

Since AKS uses **cgroup v2** by default, Judge0’s reliance on **cgroup v1** required changing the cgroup configuration on the nodes. There are two primary methods to change the cgroup configuration:

### Option 1: SSH Access to Node

If you have direct access to the node, you can modify the cgroup configuration as follows:

1. SSH into the node:
   ```bash
   ssh azureuser@<node-ip>
   ```

2.	Edit the /etc/default/grub file to set systemd.unified_cgroup_hierarchy=0:
    ```bash
    sudo sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"/' /etc/default/grub
    ```

3.	Update GRUB and reboot the system:
    ```bash
    sudo update-grub
    sudo reboot
    ```



### Option 2: Using a DaemonSet to Revert cgroup

Since we did not have direct access to the nodes in AKS, we used a DaemonSet to run a script that checks if the node is using cgroup v2 and reverts it to cgroup v1.

Here’s the basic workflow we followed:
	1.	We created a DaemonSet with a container running a script to check and change the cgroup version on the nodes.
	2.	The script checked if the cgroup version was cgroup2fs and applied the necessary GRUB changes to switch to cgroup v1.
	3.	The script also labeled the node with cgroup-version=v1 to track the nodes that were successfully reverted.
	4.	The DaemonSet was deployed.
  5.	Although I did not test this completely, I was getting `Enable succeeded: \n[stdout]\ntmpfs\n\n[stderr]\n` on running `stat -fc '%T' /sys/fs/cgroup/` to check the cgroups version.

## Challenges with Hardcoding Values

I apologize for hardcoding the values for Redis and PostgreSQL hosts and passwords in the Helm chart. Unfortunately, due to time constraints with my exam tomorrow, I couldn’t implement a more flexible configuration with environment variables or external secret management systems. I plan to address this in the future to ensure that the configuration is more dynamic and secure.
