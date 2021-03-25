# Curity CloudFormation Template
## Introduction
This CloudFormation template creates a Curity Identity server cluster in AWS.
The cluster is installed in a [Standalone Admin setup](https://developer.curity.io/docs/latest/system-admin-guide/deployment/clustering.html#standalone-admin-setup) and the cluster configuration is generated during installation of the template.

For more information on Curity and its capabilities, click [here](https://curity.io).

## Installing the CloudFormation Template

In order to install the CloudFormation template, an AWS account is required that has the correct IAM capabilities to create all the resources listed below. 

Installed Resources:
- An EC2 Instance for the Admin node
- A LaunchConfiguration for the Runtime node(s)
- An Application Load Balancer (ALB), and all related resources (listeners and target groups)
- IAM Roles and Profiles which will be attached to the nodes
- SecurityGroups for both node types and the ALB
- A S3 Bucket where the cluster configuration will be stored
- LogGroups, if logging into CloudWatch is enabled
- Scaling Policies and Alarms, used for autoscaling

## Configuration

When importing this template into CloudFormation, it is possible to configure or ommit using several key resources, such as logging and CloudWatch metrics. The parameters are explained in the following table
Parameter | Description | Default
--- | --- | ---
`AdminInstanceType` | The EC2 Instance type of the Admin node | `t3.small`
`RuntimeInstanceType` | The EC2 Instance type of the Runtime node(s) | `t3.small`
`RuntimeMinNodeCount` | The minimum number of Runtime node(s) | `2`
`RuntimeMinNodeCount` | The maximum number of Runtime node(s) | `4`
`KeyName` | The EC2 Key Pair to allow SSH access to the instances * | `null`
`VpcId` | VpcId of an existing Virtual Private Cloud (VPC) | `null`
`Subnets` | The list of SubnetIds in the Virtual Private Cloud (VPC), at least two must be selected | `null`
`TrustedIpRange` | The IP address range that can be used to SSH to the EC2 instances and access the Curity Admin UI | `0.0.0.0/0`
`LoadBalancerIpRange` | The IP address range that can be used to access Curity Runtime service through the load balancer | `0.0.0.0/0`
`CertificateArn` | The ARN of the certificate to be used by the load balancer * | `null` (optional)
`EFSDNS` | The EFS DNS for the file system containing configuration, plugins and template/translation overrides. * | `null` (optional)
`CloudWatchNamespace` | The namespace for the metrics pushed to CloudWatch. If not set, the metrics will not be pushed to CloudWatch | `null` (optional)
`EnableCloudWatchLogs` | Send application logs to cloudwatch | `no`
`MetricsScrapeInterval` | How often to scrape data from Curity's metrics endpoint (in seconds) | `30`
`MaxRequestsPerRuntimeNode` | The max threshold for the number of requests per runtime node. Exceeding this for 2 times in 5 minutes will scale up the number of runtime nodes by 1 | `400`
`MinRequestsPerRuntimeNode` | The min threshold for the number of requests per runtime node. Staying under this limit for 5 consective minutes will scale down the number of runtime nodes by 1 | `200`
`AdminUserPassword` | Password for the admin user of the Curity configuration service | `null`
`RuntimeServiceRole` | The Runtime service roles | `default`
`RuntimePort` | The internal port of the Curity runtime service | `8443`
`ConfigEncryptionKey` | The key to encrypt the Curity Configuration | `null` (optional)

\* This resource has to be created beforehand.

## Examples

### Understanding the IP Ranges

There are two IP ranges configurable in the parameters, the `TrustedIpRange` and `LoadBalancerIpRange`.

The `TrustedIpRange` is used by the SecurityGroups assigned in the Admin and Runtime nodes to allow SSH access only from this IP range. It is also used for port 6749 so the administrators can have access to the Curity Admin UI.
The Admin UI is exposed directly in the EC2 Instance, so in order to reach it you need to use the IP or DNS name of the EC2 Instance that runs the Admin Service. When a `CertificateArn` is configured, the Admin UI is instead exposed through the Load Balancer in the port 6749, while access to that port is still constraint to the `TrustedIpRange`

The `LoadBalancerIpRange` is the IP range that can access the Runtime Services through the Load Balancer. 

Note: The default for both IP ranges allows access from anywhere, so be cautious when you install the CloudFormation template, especially with the `TrustedIpRange`.

### Using a validated certificate

In AWS the certificates are managed in the Certificate Manager, which you can read more about [here](https://aws.amazon.com/certificate-manager/). When a verified certificate `arn` is configured in the parameter `CertificateArn`, the load balancer will listen to HTTPS traffic in port 443 and serve that certificate.
The same certificate will be used on the port 6749 which is only available through the Load Balancer if the `arn` is configured.

### Logs and Metrics

There are two different parameters to control if logs and metrics are sent to CloudWatch, `EnableCloudWatchLogs` and `CloudWatchNamespace` respectively. 

When `EnableCloudWatchLogs` is set to yes, two `LogGroups` are created by the CloudFormation template. The `AdminNodeLogGroup` and the `RuntimeNodeLogGroup`. The logs are sent to CloudWatch Logs by a cloudwatch-agent that runs in the EC2 Instances.

When a `CloudWatchNamespace` is configured, the [metrics](https://developer.curity.io/docs/latest/system-admin-guide/monitoring/index.html#prometheus-compliant-metrics) from the Curity Identity server are sent to CloudWatch and are grouped for each EC2 Instance in the form `CloudWatchNamespace-InstanceId`. The setting `MetricsScrapeInterval` in the parameters defines what is the interval of the scraping of the metrics, in seconds.

### Using EFS for template or message overrides and plugins

AWS EFS is a shared, elastic file storage system and it was chosen because it follows the traditional file system paradigm and can be mounted into multiple EC2 Instances. 
In order to enable this, an EFS Storage has to be created beforehand. In that storage, you can use the following file structure to install template and message overrides, as well as plugins in your cluster.
```
├── messages
│.. ├── overrides
├── templates
│.. ├── overrides
│.. ├── template-areas
├── plugins
│.. ├── pluging-group1
│.. ├── plugin-group2
```

The EFS storage, when configured, is mounted into the `/data` folder in the all the EC2 Instances that run both the Admin and Rumtine nodes. The Curity Identity Server is installed under the folder `/opt/idsvr` and there are already symlinks that point from the folders of the server to the mounted EFS storage. 

```
/opt/idsvr/usr/share/messages/overrides -> /data/messages/overrides
/opt/idsvr/usr/share/templates/overrides -> /data/templates/overrides
/opt/idsvr/usr/share/templates/template-areas -> /data/templates/template-areas
/opt/idsvr/usr/share/plugins -> /data/plugins
```

Also, the first-run script of the Curity Identity Server will copy all files under `<EFS_MOUNT_POINT>/config/` into `/opt/idsvr/etc/init/`. This way the cluster can be initialized with configuration.

> **_NOTE:_** Do not include cluster configuration, or a file called cluster.xml in the `config` folder, as it will be overridden by the one generated during startup. 


## More Information

Please visit [curity.io](https://curity.io/)  for more information about the Curity Identity Server.

Copyright (C) 2020 Curity AB.

