# Overview

This project will describe high level steps to setup a **CI/CD pipeline using OCI DevOps** to deploy a **sample HRMS App (php, nginx, ATP)** in a Kubernetes cluster (**OKE**).

- Cloud Native HRMS built using php,nginx and ATP database

## Steps

### 1. Info Capture

- **Tenancy/Users/Auth-Tokens**

```
Tenancy Name: <tenancy-name>
Tenancy Namespace: <tenancy-namespace>	# object storage namespace, eg: ansh81vru1zp

OCI Username: <oci-username>	#eg: oracleidentitycloudservice/user01_idcs
OCI Auth Token: <oci-auth-token>
```

- **Docker/Git CLI Credentials**

```
Docker CLI credentials:
Username: <tenancy-namespace>/<oci-username> #eg: ansh81vru1zp/oracleidentitycloudservice/user01
Password: <oci-auth-token>

OCI Code Repo CLI credentials:
Username: <tenancy-name>/<oci-username>
Password: <oci-auth-token>
```

### 2. Infra Resource For HRMS

#### a. Spin Up Kubernetes Cluster

```
Name: <k8s-cluster>
Region:
Compartment:
```

#### b. Provision ADB Instance 

```
Display Name: 
Username: 
Password:
Region:
Compartment:
```

### 3. Setting up ADB For HRMS | PHP-ADB Integration

#### a. Create DB Objects

```
# Create new table in Autonomous Database using Oracle Database Action or any database client of your choice. you may create an app user for HRMS or use ADMIN

CREATE TABLE employee_data (
 id NUMBER GENERATED BY DEFAULT AS IDENTITY,
 first_name VARCHAR2(50) NOT NULL,
 last_name VARCHAR2(50) NOT NULL,
 gender VARCHAR2(50) NOT NULL,
 email VARCHAR2(50) NOT NULL,
 hire_date VARCHAR2(50) NOT NULL,
 department VARCHAR2(50) NOT NULL,
 job VARCHAR2(50) NOT NULL,
 salary NUMBER(10,2) NOT NULL,
 PRIMARY KEY(id)
);
```

#### b. Download Wallet |Copy Into PHP-ADB Folder| Update Docker File

```
i. Download Instance Wallet (eg: Wallet_HRMS.zip) 
ii. Copy the wallet file in php-adb folder
iv. Modify line 13 and 14 of the Dockerfile in php-adb folder to the name of your wallet file
    
    ADD <wallet file name> /opt/oracle/<wallet file name> 
    RUN cd /opt/oracle && unzip /opt/oracle/<wallet file name>
```

### 4. Create K8s Objects

- **Create a namespace for HRMS**

```
i. Login to OKE using cloud shell
ii. create a namespcae to deploy HRMS
$ kubectl create namespace <k8s-namespace>
```

- **Create docker registry secret**

```
$ kubectl create secret docker-registry <secret-name> --docker-server=<region-key>.ocir.io --docker-username='<tenancy-namespace>/<oci-username>' --docker-password='<oci-auth-token>' --docker-email='<email-address>' -n <k8s-namespace>
```

- **Create K8s secret resource for ADB credentials**

```
$ cat > <adb-secret-name>.yml
apiVersion: v1
kind: Secret
metadata:
  name: <adb-secret-name>
type: Opaque
stringData:
  ADB_USER: <adb-username>	# eg: ADMIN
  ADB_PASSWORD: <adb-password>
  ADB_TNSNAME: <adb-tnsname>	# eg: hrms_medium
  TNS_ADMIN: /opt/oracle/
  
$ kubectl apply -f <adb-secret-name>.yml -n <k8s-namespace>
```

***<u>Note- Kubernetes Objects:</u>***

| K8s Cluster    | Type                   | Namespace        | Name             | Notes |
| :------------- | ---------------------- | ---------------- | ---------------- | ----- |
| \<k8s-cluster> | Namespace              | NA               | \<k8s-namespace> |       |
| \<k8s-cluster> | Docker Registry Secret | \<k8s-namespace> | \<secret-name>   |       |
| \<k8s-cluster> | ADB Secret             | \<k8s-namespace> | \<secret-name>   |       |

### 5. Create OCIR Repository

**Using OCI Console** 

- Create OCIR repo for nginx image and note details in the following table
- Create OCIR repo for php image and note details in the following table

***<u>Note- OCIR Objects:</u>***

| Region \| Compartment    | Name of Repo  | Purpose       | Notes |
| :----------------------- | ------------- | ------------- | ----- |
| *eg: London \| Jahangir* | *hrms/nginx1* | *nginx image* |       |
|                          |               |               |       |
|                          |               |               |       |

### 6. Create Vault, Master Encryption Key, Secret For OCI Auth Token

```
Vault Secret OCID (oci-auth-token): <vault-secret-ocid>
```

### 7. Setup OCI DevOps (CI/CD)
#### a) Create DevOps Project

```
i. Create a notification topic
ii. Create DevOps project
iii. Enable logging
```
#### b) Create/Mirror Repository
```
## Create new repo
i. Create & Clone the empty repo to client machine
ii. Copy application codes
iii. Push to OCI Code Repo

OR

## Mirror Repo
i. Get the github PAT (personal access token)
ii. Create a secret using PAT
iii. Create External Connection (github/gitlab)
ii. Mirror Repository, select connection, provide mirror schedule (once/default-30mins/custom)
```
#### c) Get & Edit the build_spec.yml file, make sure the changes are pushed to repo

```
## Edit build_spec.yml especially add value for OCIRCRED # ocid of vault secret
## Use parameter whereever possible
```

**Example parameter (build pipeline):**

| Name                     | Value | Description                           |
| ------------------------ | ----- | ------------------------------------- |
| P_OCIR_REGION_KEY        |       | eg: iad                               |
| P_TENANCY_NAMESPACE      |       | object storage namespace              |
| P_OCIR_REPO_NAME_NGINX   |       | OCIR repo name for nginx              |
| P_OCIR_REPO_NAME_PHP_ADB |       | OCIR repo name for php-adb            |
| P_OCI_USERNAME           |       | eg: oracleidentitycloudservice/user01 |
|                          |       |                                       |

#### d) Create Build Pipeline

```
i. create build pipeline
ii. create stage->Manage Build-> [Select Primary Code Repository]
iii. add value for parameters from above table to build pipeline Parameter section
iii. start a manual run to test # by selecting the latest commit hash
iv. check if build was successfull by checking the docker image
```

#### e) Create DevOps Environment Selecting OKE Cluster

#### f) Edit the deploy.yaml File | Create DevOps Artifact

```
i. Edit k8s_deployment.yaml file as required
ii. Create devops artifact | Kubernetes Manifest File | Inline
```

**Example parameter (deployment pipeline):**

| Name                     | Value | Description                           |
| ------------------------ | ----- | ------------------------------------- |
| P_OCIR_REGION_KEY        |       | eg: iad                               |
| P_TENANCY_NAMESPACE      |       |                                       |
| P_OCIR_REPO_NAME_NGINX   |       | OCIR repo name for nginx              |
| P_OCIR_REPO_NAME_PHP_ADB |       | OCIR repo name for php-adb            |
| P_OCI_USERNAME           |       | eg: oracleidentitycloudservice/user01 |
| P_K8S_NAMESPACE          |       |                                       |
| P_ADB_SECRET_NAME        |       |                                       |
| P_IMAGE_PULL_SECRET      |       |                                       |
| BUILDRUN_HASH            |       |                                       |

#### g) Create Deployment Pipeline

```
i. create deployment pipeline
ii. add stage-> (select Apply manifest to K8s cluster)
iii. select artifact
ii. add parameter from above table
iii. start a manual run and check
iv: Test URL: http://lb_ip:5000/index.php
```

#### h) Update Build Pipeline | Add Stage | Trigger Deployment

```
Steps here
```



#### i) Create Trigger To Automate CI/CD

```
Steps here
```



## Reference



## Author

[S M Jahangir Alam@Linkedin](https://www.linkedin.com/in/jahangir2526/)

Special Thanks to [**Howie Owi**@github](https://github.com/howowi)



