# Amazon MSK Custom Domain Name with IAM Authentication

This sample deploys an Amazon MSK Express broker cluster with a custom domain name, using IAM authentication and an internal Network Load Balancer (NLB) for TLS termination and broker routing.

## Architecture

The CloudFormation template provisions:

- A VPC with 2 public and 3 private subnets across 3 AZs
- A 3-node MSK Express broker cluster (`express.m7g.large`) with IAM + unauthenticated access
- An AWS Private CA (ACM PCA) root certificate authority
- An ACM certificate for `bootstrap.<domain>` with SANs for `b-1`, `b-2`, `b-3`
- An internal NLB with TLS listeners on ports 9000–9003, routing to individual brokers on port 9098
- A Route 53 private hosted zone with CNAME records pointing `bootstrap.<domain>` and `b-{1,2,3}.<domain>` to the NLB
- A KMS key for MSK encryption
- A bastion EC2 instance (Amazon Linux 2, t2.micro) accessible via SSM Session Manager, pre-installed with Kafka CLI tools and the MSK IAM auth library
- A Lambda-backed custom resource that retrieves broker IPs and bootstrap strings after cluster creation

## Port Mapping

| DNS Record | NLB Port | Target Broker | Broker Port |
|---|---|---|---|
| `bootstrap.<domain>` | 9000 | All brokers (round-robin) | 9098 |
| `b-1.<domain>` | 9001 | Broker 1 | 9098 |
| `b-2.<domain>` | 9002 | Broker 2 | 9098 |
| `b-3.<domain>` | 9003 | Broker 3 | 9098 |

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `LatestAmiId` | Amazon Linux 2 AMI (SSM) | AMI for the bastion EC2 instance |
| `MSKKafkaVersion` | `3.6.0` | Kafka version (must be 3.6.0 for Express brokers) |
| `CustomDomainName` | `example.com` | Custom domain for the cluster |

## Prerequisites

- An AWS account with permissions to create VPC, MSK, NLB, ACM PCA, Route 53, KMS, IAM, Lambda, and EC2 resources
- AWS CLI configured

## Deployment

```bash
aws cloudformation create-stack \
  --stack-name msk-custom-domain \
  --template-body file://msk-express-broker-custom-domain-name-iam-v3.yaml \
  --parameters ParameterKey=CustomDomainName,ParameterValue=kafka.example.com \
  --capabilities CAPABILITY_IAM
```

Stack creation takes approximately 30–45 minutes.

## Post-Deployment Setup

### 1. Update Advertised Listeners

After the stack is created, connect to the bastion via SSM Session Manager and run the helper script to update each broker's advertised listeners to use the custom domain:

```bash
# Get the cluster ARN from stack outputs
CLUSTER_ARN=$(aws cloudformation describe-stacks \
  --stack-name msk-custom-domain \
  --query "Stacks[0].Outputs[?OutputKey=='MSKClusterArn'].OutputValue" \
  --output text)

./update_advertised_listeners.sh --cluster-arn $CLUSTER_ARN --region <your-region>
```

### 2. Trust the Private CA

Since the ACM certificate is issued by a private CA, Kafka clients need to trust it. On the bastion, export the CA certificate into a truststore:

```bash
# Get the PCA ARN
PCA_ARN=$(aws cloudformation describe-stacks \
  --stack-name msk-custom-domain \
  --query "Stacks[0].Outputs[?OutputKey=='CertificateAuthorityARN'].OutputValue" \
  --output text)

# Download the CA cert and import into a Java truststore
aws acm-pca get-certificate-authority-certificate \
  --certificate-authority-arn $PCA_ARN \
  --query Certificate --output text > ca-cert.pem

cp $JAVA_HOME/jre/lib/security/cacerts ./cacerts
keytool -import -trustcacerts -alias rootca -file ca-cert.pem -keystore cacerts -storepass changeit -noprompt
```

### 3. Produce and Consume via Custom Domain

Use the `client-iam-custom-domain-name.properties` config file (pre-created on the bastion) which includes the custom truststore:

```bash
# Create a topic
bin/kafka-topics.sh --bootstrap-server bootstrap.kafka.example.com:9000 \
  --command-config client-iam-custom-domain-name.properties \
  --create --topic test-topic --partitions 3 --replication-factor 3

# Produce
bin/kafka-console-producer.sh --bootstrap-server bootstrap.kafka.example.com:9000 \
  --producer.config client-iam-custom-domain-name.properties \
  --topic test-topic

# Consume
bin/kafka-console-consumer.sh --bootstrap-server bootstrap.kafka.example.com:9000 \
  --consumer.config client-iam-custom-domain-name.properties \
  --topic test-topic --from-beginning
```

## Cleanup

```bash
aws cloudformation delete-stack --stack-name msk-custom-domain
```

> **Note:** The ACM PCA root CA has `deletion_protection` disabled in this sample for easy cleanup. In production, enable deletion protection and plan CA lifecycle accordingly.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
