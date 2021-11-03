---
title: Domain Join with AWS secrets service
description: When configuring windows EC2 instances, this script allows a EC2 instance to join a domain and keep the credentials safe in AWS secrets service.
author: chad-m-smith
tags: AWS, Active Directory, EC2
date_published: 2021-10-22
---

Chad Smith | Technical Alliance Architect at Teradici | HP

<p style="background-color:#CAFACA;"><i>Contributed by Teradici employees.</i></p>

Credit to http://beta.awsdocs.com/ for baseline documents, this guide alters scripts, add additonal secrects and tightens access serects. 

This script is designed to run on a freshly deployed Windows EC2 instance. Its function is to make a request to the AWS Secrets Manager to get the proper Active Directory Service Account credentials of a user that has delegated control to perform domain join operations. This negates the necessity of having hard coded username and password values sitting in the unattend.xml or the accompanying windows domain join powerShell script saved within the base AMI. The script performs the following actions:


## Objectives

+ Create key/value pair in AWS Secrets
+ Create an IAM role/policy to lock down access to Secrets
+ Allocate a AWS EC2 instance from AWS Console.
+ Drop in deployment script in User Defined (or) Windows local GPO
+ Start EC2 instance, verify domain join.

## Costs

This tutorial uses billable components of AWS Cloud and assumes Teradici subscription, including the following:
+   [AWS Nvidia EC2 Instance](https://aws.amazon.com/nvidia/), including vCPUs, memory, disk, and GPUs
+   [Internet egress and transfer costs](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/), for PCoIP and other applications communications.

Use the [AWS pricing calculator](https://calculator.aws/#/) to generate a cost estimate based on your projected usage.

## Before you begin

In this section, you set up some basic resources that the tutorial depends on.

1. Instructions in this guide assume that you have a [AWS account](https://aws.amazon.com/free/) 

1. Familiarize yourself with [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)

1. Understand [IAM roles for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) 


## Creation of AWS Secrets

In this section, you create and configure a series of Secrets key/value pairs for authentication functions in scripts.

1. Access [AWS Secrets manager](https://us-west-2.console.aws.amazon.com/secretsmanager/home?region=us-west-2#!/home) is region specfic, so enter the region you want to deploy EC2 instances. In this example we will be using *us-west-2*.

1. Select the **Store new secret** button in the upper-right hand corner of the page.

1. In the **New secrets creation** page, select:
    + In secret type: Select **Other type of secrets**.
    + In the Secrets key/value field create 2 key/vaule pairs: key named **ServiceAccount** with a AD user/service account that can add machines to AD.  Also a named **Password** and its assoicated the AD password created for the user/service created in AD.
    + Keep the Select the encryption key the default **DefaultEncryptionKey**
    
    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Create_New_Secret.jpg)
 
 1. In the **Store a new secret** page, select
    + In the Secret name field: Type **Windows/Service/DomainJoin**
    + (optionally) enter a description
    + (optionally) enter a Tag
    + Skip the Resource Permissions (optional) section for now, we will use IAM service role and policy instead. 
    + Select **Next** to continue

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Secret_name.jpg)
    
 1. In the **Store a New Secret** page, take the default options then **Next** to continue.

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/StoreNewSecret.jpg)
    
 1. Confirm the setting for the new Secrets page, then **Store** to finish the creation of secrets.

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/ConfirmSecrets.jpg)
    
 1. Once the Secret has been successfully been created, you need to find its assoicated ARN. **Select** the newly created secret and **double-click** Secret Name.

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/List_Secrets.jpg)    
 
1. Once, In the **Secret Details** page, locate the ARN section and **copy** the associated ARN to later use in the IAM role creation. 

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Locate_ARN.jpg)
     
## Create a IAM Policy

Create an IAM role for EC2 instance to read the secrets through the installation script to join the domain.

1. Go to IAM -> Policy -> Create Policy. 
    
1. Select the **Create Policy** button

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Create_Policy_Button.jpg)
    
Select the **JSON** tab, it will open in separate web browser tab. Click on tab JSON and paste the following text. Also add in the **ARN** for the **Secrets** captured in the previous step when creating the secret.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "arn:aws:secretsmanager:us-west-2:455311239824:secret:Windows/ServiceAccounts/DomainJoin-8KUbyW",
            "Effect": "Allow"
        }    
    ]
}
```
1. Select Option tag then **Next** to continue

1. Review the setting to the IAM role, **Name** the Policy then select the **Create Policy** button to finish creating the role.

## Assign a policy to a IAM Role

Within the IAM Management Console, select the **Create role** option. 

1. In the IAM Role section select **Create** Role button.

1. In the Role creation section, you will select the **AWS Service** and **EC2** under common use case.

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Create_role.jpg)

1. Search for the already created Policy name, that was set in the previous step. *(Example: ec2_domain_join_script)*

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Select_existing_role.jpg)
    
1. Next add optional Tags, Select **Next** to continue.

1. Finally, provide a name to the Role, review the Role and ensure the previous policy is assigned. 

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/Finish_role.jpg) 
  

## Procure the EC2 Nvidia Instance

In this section, you procure will procure a EC2 instance through the EC2 Dashboard. This section isn't an exhaustive explanation instead rather focusing on domain joing script. For more details directions on the actual installation process, refer to [EC2 Nvidia](https://github.com/ChadSmithTeradici/Teradici-PCoIP-Ultra-deployment-script-for-AWS-NVIDIA-EC2-instances) and [EC2 standard](https://github.com/ChadSmithTeradici/Teradici-PCoIP-Standard-deployment-script-for-AWS-EC2-instances) installation guides

1.  Launch a EC2 instance, On the [EC2 Dashboard](https://console.aws.amazon.com/ec2), choose **Launch Instance*

1. On the **Choose AMI** page, select the [Windows 2019 Base](https://aws.amazon.com/marketplace/pp/prodview-bd6o47htpbnoe?ref=cns_srchrow) AMI, then press **Select** button.

1. On the **Choose Instance Type** page, chose a instance type and choose **Next: Configure Instance Details**.

1. On the **Configure Instance Details** page, at a minimum fill in **Networking/Subnet/Auto-Assign Public-IP** based on desired Network topology. Take remaining configuration details based your requirements, until you reach the **IAM Roles**, then select the name of the IAM role previous created.

    ![image](https://github.com/ChadSmithTeradici/DomainJoin-with-AWS-Secrets-for-Windows-EC2-instances/blob/main/images/IAM_Role_Domain_Join.jpg)

Scroll down to the ** **User data** field in the Advanced Details section.

    ![image](https://github.com/ChadSmithTeradici/Teradici-PCoIP-deployment_script-for-AWS-NVIDIA-Instances/blob/main/images/User_Data_Field.jpg)
 
 1. Based on your EC2 Instance desired OS you will need either a Windows Powershell script (or) Centos Bash script. These scripts are maintained and updated quartly by Teradici and are avaible on the [Teradici GitHub repo](https://github.com/teradici)
 
    + For **[Windows 2019]** (works with other windows flavors) **Copy** all the contents of this script and **Paste** it into the **User data** field
    
      You will need to enter your **Teradici registration code** into the script after it is pasted in the User data field. For Windows that field is on line 6. Also there is an (optional)set local administrator password as well on line 7.
    
        Registration codes look like this: ABCDEFGH12@AB12-C345-D67E-89FG
    
    + For **[CentOS 7]**  **Copy** all the contents of this script and **Paste** it into the **User data** field.
    
      You will need to enter your **Teradici registration code** into the script after it is pasted in the User data field. For CentOS that field is on line 6. Also there is an (optional)to create a user and password combination to establishment of PCoIP session on line 7.

        Registration codes look like this: ABCDEFGH12@AB12-C345-D67E-89FG
 
 
    For the remaining configuration details, make any selections you prefer. Then, choose **Next: Add Storage**.

1. On the **Add Storage** page, choose the Size (GiB) cell and increase the volume based on your requirements. Then, choose **Next: Add Tags**.

1. On the **Add Tags page**, optionally add any Key:Value tags to your instance. Then, choose **Next: Configure Security Group**.

1. On the Configure Security Group page, make the following selections:

    + For **Assign a security group**, choose **Create a new security group**.
    + For **Security group name**, type a descriptive name, such as *pcoip ssh rdp*.
    + For **Description**, optionally add a description.
    + For **Type**, choose **SSH**
    + For **Source**, choose **My IP**
    + Select **Add rule**
    + For **Type**, choose **Custom TCP Rule**
    + For **Port Range** choose **HTTPS**
    + For **Source**, choose **0.0.0.0/0**
    + Select **Add rule**
    + For **Type**, choose **Custom TCP Rule**
    + For **Port Range** choose **4172**
    + For **Source**, choose **0.0.0.0/0**
    + Select **Add rule**
    + For **Type**, choose **Custom UDP Rule**
    + For **Port Range** choose **4172**
    + For **Source**, choose **0.0.0.0/0**
    + Select **Add rule**
    + For **Type**, choose **Custom TCP Rule**
    + For **Port Range** choose **3389**
    + For **Source**, choose **My IP**
    
    Then, choose **Review** and **Launch**.
