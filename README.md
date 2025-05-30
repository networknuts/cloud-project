# Shopizer Java E-commerce App Deployment on AWS

This README provides detailed, step-by-step instructions to deploy a Java/Spring Boot e-commerce application (Shopizer) on AWS using Elastic Beanstalk, RDS, S3, and CloudFront.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Create an RDS Database](#step-1-create-an-rds-database)
4. [Step 2: Create an S3 Bucket](#step-2-create-an-s3-bucket)
5. [Step 3: Create an IAM Role for EB Instances](#step-3-create-an-iam-role-for-eb-instances)
6. [Step 4: Deploy Shopizer to Elastic Beanstalk](#step-4-deploy-shopizer-to-elastic-beanstalk)
7. [Step 5: Upload Media to S3](#step-5-upload-media-to-s3)
8. [Step 6: Create a CloudFront Distribution](#step-6-create-a-cloudfront-distribution)
9. [Final Validation](#final-validation)
10. [Architecture Diagram](#architecture-diagram)

---

## Overview

You will deploy the [Shopizer](https://github.com/shopizer-ecommerce/shopizer) demo e-commerce application (Spring Boot + Thymeleaf) on Elastic Beanstalk (Corretto 11) with:

* **Database**: Amazon RDS (PostgreSQL or MySQL)
* **Static/Media Storage**: Amazon S3
* **Content Delivery**: Amazon CloudFront
* **Application Platform**: AWS Elastic Beanstalk

---

## Prerequisites

Follow these steps to prepare your environment:

### 1. AWS Account and Permissions

1. **Sign up for AWS** (if you don’t have an account):

   * Visit [https://aws.amazon.com](https://aws.amazon.com) and click **Create an AWS Account**.
2. **Create an IAM user** for deployments:

   * Open the **IAM** console.
   * Go to **Users** → **Add users**.
   * Username: `deploy-user` (or your choice).
   * Select **Programmatic access** and **AWS Management Console access**.
   * Under **Permissions**, attach the **AdministratorAccess** policy (or a custom policy granting RDS, S3, IAM, Elastic Beanstalk, and CloudFront permissions).
   * Complete the wizard and download the **Access Key ID** and **Secret Access Key**.
3. **(Optional)** Create IAM groups for finer-grained control:

   * In **IAM**, create a group (e.g. `deployers`) with required policies.
   * Add your `deploy-user` to this group.

### 2. Package the Shopizer Application (on a temporary EC2 instance)

Assuming you’ll perform these steps on a fresh EC2 instance:

1. **Launch an EC2 instance**

   * AMI: **Amazon Linux 2 AMI (HVM), SSD Volume Type** (e.g. `ami-0abcdef1234567890` in us-east-1)
   * Instance type: **t3.small** (or larger)
   * Key pair: your SSH key
   * Network: same VPC/subnet as your RDS (or a public subnet with internet access)
   * Security group: allow **SSH (port 22)** from your IP

2. **Connect via SSH**:

   ```bash
   ssh -i /path/to/your-key.pem ec2-user@<EC2_PUBLIC_IP>
   ```

3. **Install prerequisites**:

   ```bash
   sudo yum update -y
   sudo yum install -y git unzip zip
   ```

   ### Install Java 11

   Amazon Linux 2 may not include the `corretto11` topic in Amazon Linux Extras. You have two options:

   **Option A: Install OpenJDK 11 via Amazon Linux Extras**

   ```bash
   sudo amazon-linux-extras enable java-openjdk11
   sudo yum install -y java-11-openjdk-devel
   ```

   **Option B: Install Amazon Corretto 11 manually**

   ```bash
   curl -Lo corretto11.rpm https://corretto.aws/downloads/latest/amazon-corretto-11-x64-linux-jdk.rpm
   sudo yum localinstall -y corretto11.rpm
   rm -f corretto11.rpm
   ```

   **Verify Java installation**:

   ```bash
   which java    # e.g. /usr/bin/java
   which javac   # e.g. /usr/bin/javac
   java -version # Amazon Corretto or OpenJDK 11
   javac -version# 11.x.x
   ```

   **If Java is still not found or JAVA\_HOME is not set**:

   1. Find the install location:

      ```bash
      sudo find /usr/lib/jvm -maxdepth 1 -type d
      ```
   2. Set JAVA\_HOME and update PATH:

      ```bash
      echo 'export JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto' >> ~/.bashrc
      echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
      source ~/.bashrc
      ```

4. **Clone the Shopizer repository**: **Clone the Shopizer repository**: **Clone the Shopizer repository**: **Clone the Shopizer repository**:

   ```bash
   git clone https://github.com/shopizer-ecommerce/shopizer.git
   cd shopizer
   ```

5. **Ensure the Maven wrapper is available and executable**:

   * The Maven wrapper script `mvnw` is located at the project root (`./mvnw`).
   * The wrapper JAR and properties live under `.mvn/wrapper/`:

     ```bash
     ls -R .mvn/wrapper
     # .mvn/wrapper:
     # gradle-wrapper.jar  gradle-wrapper.properties
     ```
   * Make the script executable:

     ```bash
     chmod +x mvnw
     ```

6. **Build the Spring Boot JAR**: **Build the Spring Boot JAR**:

   ```bash
   ./mvnw clean package -DskipTests
   ```

7. **Prepare the deployment ZIP**:

   ```bash
   mkdir deploy
   cp modules/sm-shop/target/shopizer-*.jar deploy/shopizer-boot.jar
   mkdir -p deploy/.ebextensions
   cp .ebextensions/postdeploy.config deploy/.ebextensions/
   cd deploy
   zip -r ../shopizer-deploy.zip .
   cd ..
   ```

### 3. Install and Configure the AWS CLI. Install and Configure the AWS CLI

1. **Install AWS CLI v2**:

   * **macOS**: `brew install awscli`
   * **Windows**: Download and run the MSI installer from AWS docs.
   * **Linux**: Follow instructions at [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. **Verify installation**:

   ```bash
   aws --version
   ```
3. **Configure AWS CLI**:

   ```bash
   aws configure
   # Enter your Access Key ID, Secret Access Key,
   # Default region name (e.g., us-east-1),
   # Default output format (e.g., json)
   ```

---

## Step 1: Create an RDS Database

1. Sign in to the AWS Management Console and open **Amazon RDS**.
2. Click **Create database**.
3. Engine: **PostgreSQL** (or **MySQL**).
4. Template: **Production**.
5. DB instance identifier: `shopizer-db`.
6. Master username/password: choose secure credentials.
7. Connectivity:

   * VPC: your application VPC.
   * Subnet group: private subnets.
   * Public access: **No**.
   * Security group: allow inbound on port 5432 from EB instances.
8. Additional configuration:

   * Database name: `shopizerdb`.
9. Click **Create database**.
10. Once **Available**, copy the **Endpoint** and **Port**.

---

## Step 2: Create an S3 Bucket

1. Open **Amazon S3** in the Console.
2. Click **Create bucket**.
3. Bucket name: `myjava-shop-static-12345`.
4. Region: same as RDS/EB (e.g., `us-east-1`).
5. Uncheck **Block all public access** if you plan public hosting (acknowledge warning).
6. Click **Create bucket**.
7. (Optional) Add a bucket policy granting public read for `/media/*` and `/static/*`.

---

## Step 3: Create an IAM Role for EB Instances

1. Open **IAM** → **Roles** → **Create role**.
2. Select **AWS service** → **Elastic Beanstalk EC2**.
3. Attach policies:

   * `AmazonS3FullAccess` (or scoped policy to your bucket).
   * `AmazonRDSFullAccess` (or scoped to your RDS instance).
4. Role name: `eb-shopizer-service-role`.
5. Create the role.

---

## Step 4: Deploy Shopizer to Elastic Beanstalk

1. Open **Elastic Beanstalk** in the Console.
2. Click **Create application**.

   * Application name: `shopizer-java-app`.
   * Platform: **Corretto 11 on Amazon Linux 2**.
3. Create environment:

   * Environment name: `shopizer-java-env`.
   * Domain: `shopizer-mycompany`.
   * Upload your code: select `shopizer-deploy.zip`.
4. Configure more options:

   * **Network**: select your VPC and private subnets.
   * **Security**:

     * Instance profile: `eb-shopizer-service-role`.
     * Security groups: allow HTTP/HTTPS.
   * **Software** → Environment properties:

     ```properties
     SPRING_DATASOURCE_URL=jdbc:postgresql://<RDS-ENDPOINT>:5432/shopizerdb
     SPRING_DATASOURCE_USERNAME=<db-user>
     SPRING_DATASOURCE_PASSWORD=<db-password>
     AWS_S3_BUCKET=myjava-shop-static-12345
     AWS_REGION=us-east-1
     ```
   * **Capacity**: t3.small (min 1, max 2).
5. Click **Create environment**.
6. Wait for deployment (\~5–10 minutes) and verify health **OK**.

---

## Step 5: Upload Media to S3

1. In the S3 Console, open `myjava-shop-static-12345`.
2. Click **Upload** → **Add files/folder** → select your `media/` directory.
3. Under **Permissions**, grant **Public read** (if desired).
4. Click **Upload**.

Alternatively, via AWS CLI:

```bash
aws s3 sync media/ s3://myjava-shop-static-12345/media/ --acl public-read
```

---

## Step 6: Create a CloudFront Distribution

1. Open **CloudFront** in the Console.
2. Click **Create distribution** → **Web**.
3. Add Origins:

   * **Origin 1**: S3 bucket (`myjava-shop-static-12345.s3.amazonaws.com`), Origin ID `S3-myjava-shop`.
   * **Origin 2**: EB URL (`shopizer-java-env.us-east-1.elasticbeanstalk.com`), Origin ID `EB-shopizer-java-env`.
4. Default Cache Behavior:

   * Origin: **EB**.
   * Forward all headers/cookies.
5. Add Cache Behavior:

   * Path pattern: `/media/*`.
   * Origin: **S3-myjava-shop**.
   * Viewer protocol: **Redirect HTTP to HTTPS**.
   * TTL: 86400 seconds.
6. (Optional) Add `/static/*` behavior similarly.
7. Distribution settings:

   * CNAMEs: `shop.mycompany.com`.
   * SSL: ACM certificate covering your domain.
8. Click **Create distribution** and wait for **Deployed** status.

---

## Final Validation

* **EB URL** serves the Shopizer storefront (dynamic pages).
* **CloudFront domain** serves media at `/media/...` and static assets.
* Point your custom domain (via Route 53) to the CloudFront distribution for production.

---

## Architecture Diagram

```
   ┌─────────────┐          ┌───────────────┐
   │ CloudFront  │───┐─────▶│   S3 Bucket   │
   └─────────────┘   │      └───────────────┘
         │           │
         │           ▼
         │     ┌──────────────┐
         └────▶│ Elastic      │
               │ Beanstalk    │
               └──────────────┘
                     │
                     ▼
               ┌──────────────┐
               │   RDS DB     │
               └──────────────┘
```

---

You now have a fully functional, Java-based e-commerce application deployed on AWS!
