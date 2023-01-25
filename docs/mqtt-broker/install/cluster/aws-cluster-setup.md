---
layout: docwithnav-mqtt-broker
title: Cluster setup using AWS infrastructure
description: ThingsBoard MQTT Broker microservices setup with Kubernetes in AWS EKS

---

* TOC
{:toc}

This guide will help you to set up ThingsBoard MQTT Broker in AWS EKS.

## Prerequisites

{% include templates/install/aws/eks-prerequisites.md %}

### Pull ThingsBoard MQTT Broker image from docker hub

Run the following command to verify that you can pull the image from the Docker hub.

```bash
docker pull thingsboard/tb-mqtt-broker:{{ site.release.broker_full_ver }}
```
{: .copy-code}

## Step 1. Open ThingsBoard MQTT Broker K8S scripts repository

```bash
git clone https://github.com/thingsboard/thingsboard-mqtt-broker.git
cd thingsboard-mqtt-broker/k8s/aws
```
{: .copy-code}

## Step 2. Configure and create EKS cluster

In the `cluster.yml` file you can find suggested cluster configuration.
Here are the fields you can change depending on your needs:
- `region` - should be the AWS region where you want your cluster to be located (the default value is `us-east-1`)
- `availabilityZones` - should specify the exact IDs of the region's availability zones
  (the default value is `[us-east-1a,us-east-1b,us-east-1c]`)
- `instanceType` - the type of the instance with TB node (the default value is `m5.large`)

**Note**: if you don't make any changes to `instanceType` and `desiredCapacity` fields, the EKS will deploy **1** node of type **m5.large**.

{% capture aws-eks-security %}
In case you want to secure access to the PostgreSQL and MSK, you'll need to configure the existing VPC or create a new one,
set it as the VPC for the ThingsBoard MQTT Broker cluster, create security groups for PostgreSQL and MSK,
set them for `main-node-group` node-group in the ThingsBoard MQTT Broker cluster and configure the access from the ThingsBoard MQTT Broker cluster nodes to PostgreSQL/MSK using another security group.

You can find more information about configuring VPC for `eksctl` [here](https://eksctl.io/usage/vpc-networking/).
{% endcapture %}
{% include templates/info-banner.md content=aws-eks-security %}

Command to create AWS cluster:

```
eksctl create cluster -f cluster.yml
```
{: .copy-code}

## Step 3. Create AWS load-balancer controller

Once the cluster is ready you'll need to create AWS load-balancer controller.
You can do it by following [this](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html) guide.

Provisioning of the AWS load-balancer controller is a **very important step** that is required for those load balancers to work properly.

## Step 4. Amazon PostgreSQL DB Configuration

You'll need to set up PostgreSQL on Amazon RDS.
One of the ways to do it is by following [this](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_SettingUp.html) guide.

**Note**: Make sure your database is accessible from the cluster, one of the way to achieve this is to create
the database in the same VPC and subnets as ThingsBoard MQTT Broker cluster and use 
‘eksctl-thingsboard-mqtt-broker-cluster-ClusterSharedNodeSecurityGroup-*’ security group. See screenshots below.

Here you should choose VPC with the name of your cluster:

![image](/images/mqtt-broker/install/aws-rds-vpc.png)

Here you should choose security group corresponding to the one on the screen:

![image](/images/mqtt-broker/install/aws-rds-vpc-sg.png)

**Note**, some recommendations:

* Make sure your PostgreSQL version is latest 12.x, not 13.x yet;
* Use ‘Production’ template for high availability. It enables a lot of useful settings by default;
* Consider creation of custom parameters group for your RDS instance. It will make change of DB parameters easier;
* Consider deployment of the RDS instance into private subnets. This way it will be nearly impossible to accidentally expose it to the internet.

Make sure that `thingsboard_mqtt_broker` database is created along with PostgreSQL instance (or create it afterwards).

![image](/images/mqtt-broker/install/aws-rds-default-database.png)

**Note:** You may also change `username` field and set or auto-generate `password` field (keep your postgresql password in a safe place).

## Step 5. Amazon MSK Configuration

You'll need to set up Amazon MSK.
To do so you need to open AWS console, MSK submenu, press `Create cluster` button and choose `Custom create` mode.
You should see the similar image:

![image](/images/mqtt-broker/install/aws-msk-creation.png)

**Note**: Make sure your MSK instance is accessible from the ThingsBoard MQTT Broker cluster.
The easiest way to achieve this is to deploy the MSK instance in the same VPC.
We also recommend to use private subnets. This way it will be nearly impossible to accidentally expose it to the internet;

Now you should choose the ThingsBoard MQTT Broker cluster's VPC for the Kafka cluster:

![image](/images/mqtt-broker/install/aws-msk-vpc.png)

After that you need to browse the security groups and choose group corresponding to the one on the screen:

![image](/images/mqtt-broker/install/aws-msk-vpc-sg.png)

Also, you should enable `Plaintext` communication between clients and brokers:

![image](/images/mqtt-broker/install/aws-msk-security.png)

**Note**, some recommendations:

* Make sure your Apache Kafka version is 2.8.x;
* Use m5.large or similar instance types;
* Use default 'Monitoring' settings or enable 'Enhenced topic level monitoring'.

## Step 6. Configure links to the Kafka (Amazon MSK)/Postgres

### Amazon RDS PostgreSQL

Once the database switch to the ‘Available’ state, on AWS Console get the `Endpoint` of the RDS PostgreSQL and paste it to 
`SPRING_DATASOURCE_URL` in the `tb-broker-configmap.yml`.

![image](/images/mqtt-broker/install/aws-rds-endpoint.png)

Also, you'll need to set `SPRING_DATASOURCE_USERNAME` and `SPRING_DATASOURCE_PASSWORD` with PostgreSQL `username` and `password` corresponding.

### Amazon MSK

Once the MSK cluster switch to the ‘Active’ state, to get the list of brokers execute the next command:
```
aws kafka get-bootstrap-brokers --region us-east-1 --cluster-arn $CLUSTER_ARN
```
{: .copy-code}
Where **$CLUSTER_ARN** is the Amazon Resource Name (ARN) of the MSK cluster:

![image](/images/mqtt-broker/install/aws-msk-arn.png)

You'll need to paste data from the `BootstrapBrokerString` to the `TB_KAFKA_SERVERS` environment variable in the `tb-broker.yml` file.

Otherwise, click ‘View client information’. Copy bootstrap server information in plaintext.

## Step 7. Installation

Execute the following command to run installation:
```
./k8s-install-tb-mqtt-broker.sh
```
{: .copy-code}

After this command finish you should see the next line in the console:

```
INFO  o.t.m.b.i.ThingsboardMqttBrokerInstallService - Installation finished successfully!
```

Otherwise, please check if you set the PostgreSQL URL in the `tb-broker-configmap.yml` correctly.

## Step 8. Starting

Execute the following command to run installation:

```bash
./k8s-deploy-tb-broker.sh
```
{: .copy-code}

After few minutes you may call `kubectl get pods`. If everything went fine, you should be able to see `tb-broker-0` pod.
Every pod should be in the `READY` state.

## Step 9. Validate the setup

Now you can open ThingsBoard MQTT Broker web interface in your browser using DNS name of the load balancer.

You can see DNS name of the load-balancers using command:

```
kubectl get service
```
{: .copy-code}

You should see the similar picture:

![image](/images/mqtt-broker/install/aws-loadbalancer.png)

Use `EXTERNAL-IP` field of the `tb-broker-loadbalancer-external` to connect to the cluster.

Use the following default credentials:

- **System Administrator**: sysadmin@thingsboard.org / sysadmin

#### Troubleshooting

In case of any issues you can examine service logs for errors. For example to see ThingsBoard MQTT broker logs execute the following command:

```
kubectl logs -f tb-broker-0
```
{: .copy-code}

Or use `kubectl get pods` to see the state of the pods.
Or use `kubectl get services` to see the state of all the services.
Or use `kubectl get statefulsets` to see the state of all the deployments.
See [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) command reference for details.

## Cluster deletion

Execute the following command to delete ThingsBoard MQTT Broker nodes:

```
./k8s-delete-tb-broker.sh
```
{: .copy-code}

Execute the following command to delete all ThingsBoard MQTT Broker nodes and configmaps:

```
./k8s-delete-all.sh
```
{: .copy-code}

Execute the following command to delete EKS cluster (you should change the name of the cluster and zone):
```
eksctl delete cluster -r us-east-1 -n thingsboard-mqtt-broker-cluster -w
```
{: .copy-code}

## Next steps

{% assign currentGuide = "InstallationGuides" %}{% include templates/mqtt-broker-guides-banner.md %}