---
title: Setup Sentiment Analysis with Amazon EKS
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Before you begin

You will need an [AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html). Create an account if needed. 

Multiple tools are required on your local computer. Follow the links to install each tool.

* [Kubectl](/install-guides/kubectl/)
* [AWS CLI](/install-guides/aws-cli/)
* [Docker](/install-guides/docker/)
* [Terraform](/install-guides/terraform/)
* [Java](/install-guides/java/)

To use the AWS CLI, you will need to generate AWS access keys and configure the CLI. Follow the [AWS Credentials](/install-guides/aws_access_keys/) install guide for instructions. 

## Setup sentiment analysis

Take a look at the [GitHub repository](https://github.com/koleini/spark-sentiment-analysis) then clone it on your local computer:

```console
git clone https://github.com/koleini/spark-sentiment-analysis.git
cd spark-sentiment-analysis/eks
```

Edit the file `variables.tf` if you want to change the default AWS region.

The default value is at the top of the file and is set to `us-east-1`.

```output
variable "AWS_region" {
  default     = "us-east-1"
  description = "AWS region"
}
```

Also, if you're using a profile other than `default` then update the following variable

```output
variable "AWS_profile" {
  default     = "Your_AWS_Profile"
  description = "AWS authorization profile"
}
```

Execute the following commands to create the Amazon EKS cluster:

```console
terraform init
terraform apply --auto-approve
```

Once the cluster gets created, verify it in AWS console.

Update the `kubeconfig` file to access the deployed EKS cluster with the following command:

If you want to use an AWS CLI profile with is not the default, change the profile name before running the command. 

```console
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name) --profile <Your_AWS_Profile>
```

Create a service account for Apache spark

```console
kubectl create serviceaccount spark
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

## Build the sentiment analysis JAR file

Navigate to the `sentiment_analysis` folder to create a JAR file for the sentiment analyzer.

You will need `sbt` installed. If you are running Ubuntu, you can install it with:

```console
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
echo "deb https://repo.scala-sbt.org/scalasbt/debian /" | sudo tee /etc/apt/sources.list.d/sbt_old.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt-get update
sudo apt-get install sbt
```

If you have another operating system, refer to [Installing sbt](https://www.scala-sbt.org/1.x/docs/Setup.html).

```console
cd ../sentiment_analysis
sbt assembly
```

A JAR file is created at the following location:

```console
sentiment_analysis/target/scala-2.13/bigdata-assembly-0.1.jar
```

## Create a Spark container image

Create a repository in Amazon ECR to store the docker images. You can also use Docker Hub.

The Spark repository contains a script to build the container image you need to run inside the Kubernetes cluster. 

Execute this script on your Arm-based computer to build the arm64 image.

In the current working directory, use the following commands to get the `apache spark` tar. Extract this repository prior to building the image

```console
wget https://archive.apache.org/dist/spark/spark-3.4.3/spark-3.4.3-bin-hadoop3-scala2.13.tgz
tar -xvzf spark-3.4.3-bin-hadoop3-scala2.13.tgz
cd spark-3.4.3-bin-hadoop3-scala2.13
```

Copy the JAR file generated in previous step to the following location
```console
cp ../sentiment_analysis/target/scala-2.13/bigdata-assembly-0.1.jar jars/
```

Build the docker container using the following commands. Substitute the name of your container repository before running the commands.

```console
bin/docker-image-tool.sh -r <your-docker-repository> -t sentiment-analysis build
bin/docker-image-tool.sh -r <your-docker-repository> -t sentiment-analysis push
```

## Run Spark computation on the cluster

Execute the `spark-submit` command within the Spark folder to deploy the application. The following commands will run the application with two executors, each with 12 cores, and allocate 24GB of memory for both the executors and driver pods.

Set the following variables before executing the `spark-submit` command:

```console
export K8S_API_SERVER_ADDRESS=<K8S_API_SERVER_ENDPOINT
export ES_ADDRESS=<IP_ADDRESS_OF_ELASTICS_SEARCH>
export CHECKPOINT_BUCKET=<S3_UCKET_NAME>
export ECR_ADDRESS=<ECR_REGISTERY_ADDRESS>
```

Execute the `spark-submit` command:

```console
bin/spark-submit \
      --class bigdata.SentimentAnalysis \
      --master k8s://K8S_API_SERVER_ADDRESS:443 \
      --deploy-mode cluster \
      --conf spark.executor.instances=2 \
      --conf spark.kubernetes.container.image=$ECR_ADDRESS \
      --conf spark.kubernetes.driver.pod.name="spark-twitter" \
      --conf spark.kubernetes.namespace=default \
      --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
      --conf spark.driver.extraJavaOptions="-DES_NODES=$ES_ADDRESS -DCHECKPOINT_LOCATION=s3a://$CHECKPOINT_BUCKET/checkpoints/" \
      --conf spark.executor.extraJavaOptions="-DES_NODES=$ES_ADDRESS -DCHECKPOINT_LOCATION=s3a://$CHECKPOINT_BUCKET/checkpoints/" \
      --conf spark.executor.cores=12 \
      --conf spark.driver.cores=12 \
      --conf spark.driver.memory=24g \
      --conf spark.executor.memory=24g \
      --conf spark.memory.fraction=0.8 \
      --name sparkTwitter \
      local:///opt/spark/jars/bigdata-assembly-0.1.jar
```

Use `kubectl get pods` to check the status of the pods in the cluster.

```output
NAME                                        READY   STATUS    RESTARTS   AGE
sentimentanalysis-346f22932b484903-exec-1   1/1     Running   0          10m
sentimentanalysis-346f22932b484903-exec-2   1/1     Running   0          10m
spark-twitter                               1/1     Running   0          12m
```

## Twitter sentiment analysis

Create a twitter(X) [developer account](https://developer.x.com/en/docs/x-api/getting-started/getting-access-to-the-x-api) and download the `bearer token`. 

Use the following commands to set the bearer token and fetch the Tweets:

```console
export BEARER_TOKEN=<BEARER_TOKEN_FROM_X>
python3 scripts/xapi_tweets.py
```

{{% notice Note %}}
You may need to install the following python packages, if you run into any dependency issues.
pip3 install requests
pip3 install boto3
{{% /notice %}}

You can modify the script `xapi_tweets.py` with your own keywords. 

Here is the code which includes the keywords: 

```output
query_params = {'query': "(#onArm OR @Arm OR #Arm OR #GenAI) -is:retweet lang:en",
                'tweet.fields': 'lang'}
```

Use the following command to send these processed tweets to Elasticsearch

```console
python3 csv_to_kinesis.py
```

Navigate to the Kibana dashboard using the following URL and analyze the tweets

```console
http://<IP_Address_of_ES_and_Kibana>:5601
```

## Environment Cleanup

Following this learning path will deploy a lot of artifacts in your cloud account. Remember to destroy the resources after you're done executing it. Use the following command to cleanup the resources

```console
terraform destroy
```