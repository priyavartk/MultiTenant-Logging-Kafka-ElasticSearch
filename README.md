## Multi-tenant logging using EKS , Amazon MSK and  Amazon OpenSearch

You can use terraform code and following instructions to use EKS's namespace for tenant's isolation and forward logs AMazon Managed Service for Kafka to store and Finally to OpeneSearch. To achieve this, we will deploy Fluent Bit as a DaemonSet to tail /var/log/containers/*.log on the EKS cluster and use fluent-bit annotations to configure desired parser for deployments(pods) in each tenant's namespace.It will create one topic for each tenant in KAFKA and a MSK connector for OpenSearch to send these logs to OpenSearch such that there will be one index per namespace configured to log to OpenSearch. In the end there is a link to OpenSearch multi-tenancy configuration using RBAC .

Terraform code will help you to create an EKS cluster, MSK cluster, Kafka custom pluging ,Kafka Connector  and OpenSearch domain in one VPC.

#### Prerequisites

* . A route 53 private hosted zone to ensure we get our kafka brokers to a fixed domain name

* . A S3 bucket for terraform backend

* . A Cloud9 IDE or an EC2 instances with IAM role attached with Administrator permissions.

#### Instructions

* To get started, edit **0-proivder.tf** to update backend S3 bucket , region and key prefix.

* Terraform will install fluent-bit daemonset in 'logging' namespace and also create a 'example' namespace.

* Refer to **3-variables.tf** to create/edit more namespaces and enable logging on them. 'enable_logs_to_es' allows you configure namespaces to log or not.
```
default = [
    {
      "name" : "logging",
      "enable_logs_to_es" = false,
    },
    {
      "name" : "example",
      "enable_logs_to_es" = true,
```
* Note. Terraform code will create VPC and all required components. But your OpenSearch dashboard will not be accessible over internet, so you might consider using a AWS client VPN ( or any connectivity method to allow you access to dashboard). you can also launch use a Microsoft windows instance in same VPC and access it via RDP and then access your OpenSearch dashboard 
 
```cd terraform
   run terraform init  
   terraform apply 
 ```
Wait for terraform  apply to complete. It can take approximatly 30-35 mins to provison MSK and Kafka connector to become "Running" 

2. Download KUBECONFIG to connect to your EKS cluster ( run aws eks update-kubeconfig --name <<name of your EKS cluster >>

3. Deploy a sample app in example namespace. This deployment will use 'nginx' parser

```
kubectl config set-context --current --namespace=logging
kubectl apply -f example.yaml
```
3. Login to EC2 instance which have KAFKA client binary are installed and   list KAFKA topics to verify logs_example and logs_logging topics are created and logs are sent to them.
 
./bin/kafka-topics.sh --bootstrap-server=<<list of your brokers>>  --list
./bin/kafka-console-consumer.sh --bootstrap-server <<list of your brokers>. --topic logs_example    

4. Login to your OpenSearch Dashboard as admin and verify the indexes are created for each of namespace enabled to log to OpenSearch. 


* Fluent-bit allows you to choose your parser. Annotate your pods with following to choose your parser.
   '''
      fluentbit.io/parser: apache
   '''
* If you want to completely opt out of logging for any of your pods. Use

	```fluentbit.io/exclude: "true"
```
* To configure and use RBAC with OpenSearch , you can follow instructions from https://aws.amazon.com/blogs/apn/storing-multi-tenant-saas-data-with-amazon-opensearch-service/

