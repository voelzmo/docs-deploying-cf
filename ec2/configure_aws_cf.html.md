---
title: Configuring AWS for Cloud Foundry
---

This topic describes how to configure Amazon Web Services (AWS) for Cloud Foundry.

## <a id="create-cf-manifest"></a>Step 1: Create a Deployment Manifest

The Cloud Foundry deployment manifest is a YAML file that defines the deployment components and properties. The [*cloudfoundry/cf-release*](https://github.com/cloudfoundry/cf-release) repo
contains template deployment manifests that you can edit with your deployment information. The `minimal-aws.yml` file contains the minimum information necessary to deploy Cloud Foundry to AWS.

Create a manifest for your deployment as follows:

1. Create a deployment directory to store your manifest.

    <pre class='terminal'>
    $ mkdir ~/CF-deployment
    </pre>

1. Clone the *cloudfoundry/cf-release* repo.

    <pre class='terminal'>
    $ git clone https://github.com/cloudfoundry/cf-release.git
    </pre>

1. Navigate to the *example_manifests* subdirectory to retrieve the `minimal-aws.yml` template. Copy and paste the template into a text editor and save the edited manifest to your deployment directory.

    In the template, you must replace the following properties:
    * `REPLACE_WITH_DIRECTOR_ID`
    * `REPLACE_WITH_PRIVATE_SUBNET_ID`
    * `REPLACE_WITH_PUBLIC_SUBNET_ID`
    * `REPLACE_WITH_ELASTIC_IP`
    * `REPLACE_WITH_PUBLIC_SECURITY_GROUP`
    * `REPLACE_WITH_SYSTEM_DOMAIN`
    * `REPLACE_WITH_SSL_CERT_AND_KEY`
    

1. Run `bosh status --uuid` to retrieve your BOSH Director ID. Update `REPLACE_WITH_DIRECTOR_ID` in the example manifest with this value.

We describe replacing these properties in [Step 2: Configure AWS for Your Cloud Foundry Deployment](#config-aws).

##<a id="config-aws"></a>Step 2: Configure AWS for Your Cloud Foundry Deployment

To configure your AWS account for Cloud Foundry:

* [Create a NAT VM](#create-nat-vm)
* [Update the MicroBOSH Security Group](#update-mibo-sec-group)
* [Create a Subnet for Cloud Foundry Deployment](#create-cf-subnet)
* [Configure your Cloud Foundry System Domain](#config-cf-dns)
* [Multi-AZ full deployment prep](#config-cf-for-multi-az-deploy)

<p class="note"><strong>Note</strong>: Ensure that "N. Virginia" is selected as the AWS Region.</p>

###<a id="create-nat-vm"></a>Create a NAT VM

1. On the EC2 Dashboard, click **Launch Instance**.
1. Click **Community AMIs**.
1. Search for and select "amzn-ami-vpc-nat-pv-2014.09.1.x86_64-ebs".
1. Select "m1.small".
1. Click **Next: Configure Instance Details** and complete as follows:
    * **Network**: Select your 'microbosh' VPC.
    * **Subnet**: Select your "Public subnet".
    * **Auto-assign Public IP**: "Enable"
1. Click **Next: Add Storage**.
1. Click **Next: Tag Instance**.
1. Enter "NAT" as the **Value** for the "Name" **Key**.
1. Click **Next: Configure Security Group**.
1. Click **Create a new security group** and complete as follows:
    * **Security group name**: "nat"
    * **Description**: "NAT Security Group"
    * **Type**: "All traffic""
    * **Protocol**: "All"
    * **Port Range**: "0 - 65535"
    * **Source**: "Custom IP / 10.0.16.0/24"
1. Click **Review and Launch**.
1. Click **Launch**.
1. Specify "Choose an existing key pair" and "bosh" from the dropdown menus.
1. Click **Launch Instances**.
1. Click **View Instances**.
1. Select the "NAT" instance in the **Instances** list.
1. Click **Actions**, then **Networking**, then **Change Source/Dest. Check**.
1. Click **Yes, Disable**.


###<a id="update-mibo-sec-group"></a> Update the MicroBOSH Security Group

1. On the VPC Dashboard, click **Security Groups**.
1. Select the "bosh" security group.
1. Click **Inbound Rules** at the bottom.
1. Click **Edit** and **Add another rule** as follows:
    * **Type**: "Custom TCP Rule"
    * **Protocol**: "TCP (6)"
    * **Port Range**: "4443"
    * **Source**: "0.0.0.0/0"
1. Click **Save**.

###<a id="create-cf-subnet"></a> Create a Subnet for Cloud Foundry Deployment 

1. Click **Subnets** from the VPC Dashboard.
1. Click **Create Subnet** and complete as follows:
    * **Name tag**: cf
    * **VPC**: microbosh
    * **Availability Zone**: Pick the same Availability Zone as the MicroBOSH Subnet.
    * **CIDR block**: 10.0.16.0/24
    * Click **Yes, Create**.
1. Replace the following in your manifest:
    * `REPLACE_WITH_AZ` with the Availability Zone you chose.
    * `REPLACE_WITH_PRIVATE_SUBNET_ID` with the Subnet ID for the cf Subnet.
    * `REPLACE_WITH_PUBLIC_SUBNET_ID` with the Subnet ID for the MicroBOSH Subnet.
1. Select the "cf Subnet" from the **Subnet** list.
1. Click the **Route table** tab in the bottom window to view the route tables.
1. Click the route table ID link in the **Route Table** field.
1. Click the **Routes** tab in the bottom window.
1. Click **Edit** and complete as follows:
    * **Destination**: 0.0.0.0/0
    * **Target**: Select the NAT instance from the list.
    * Click **Save**.
1. Click **Elastic IPs** from the VPC Dashboard.
1. Click **Allocate New Address** and click **Yes, Allocate**.
1. Update `REPLACE_WITH_ELASTIC_IP` in your manifest with the new IP address.
1. Click **Security Groups** from the VPC Dashboard.
1. Click **Create Security Group**.
    * **Name tag**: cf-public
    * **Group name**: cf-public
    * **Description**: cf Public Security Group
    * **VPC**: Select the bosh VPC.
    * Click **Create**.
1. In **Inbound Rules** tab in the bottom window, click **Edit** and add the following inbound rules:
<table border="1" class="nice">
	<tr>
		<th>Type</th>
		<th>Port Range</th>
		<th>Source</th>
		<th>Purpose</th>
	</tr>
	<tr><td>HTTP</td><td>TCP</td><td>80</td><td>0.0.0.0/0</td></tr>
	<tr><td>HTTPS</td><td>TCP</td><td>443</td><td>0.0.0.0/0</td></tr>
	<tr><td>TCP</td><td>TCP</td><td>4443</td><td>0.0.0.0/0</td></tr>
</table>

1. Update `REPLACE_WITH_PUBLIC_SECURITY_GROUP` in your manifest with the new security group.

###<a id="config-cf-dns"></a> Configure your Cloud Foundry System Domain

If you have a domain you plan to use for your Cloud Foundry System Domain, set up your DNS as follows:

1. Create a wildcard DNS entry for your root System Domain in the form `*.your-cf-domain.com` to point at the Elastic IP address you created in the [Create a Subnet for Cloud Foundry Deployment](#create-cf-subnet) section.
1. Click on **Route 53** from the Amazon Web Services Dashboard.
1. Click on **Hosted Zones**.
1. Select your zone.
1. Click **Go to Record Sets**.
1. Click **Create Record Set** and complete as follows:
    * **Name**: *
    * **Type**: A - IPv4 address
    * **Value**: Enter the Elastic IP created above.
    * Click **Create**.

If you do not have a domain, you can use 0.0.0.0.xip.io for your System Domain and replace the zeroes with your Elastic IP.

1. Update `REPLACE_WITH_SYSTEM_DOMAIN` with your system domain value.

1. Run the following series of commands to generate an SSL certificate for your system domain:

    <pre class="terminal">
    $ openssl genrsa -out cf.key 1024
    </pre>

    <p class="note"><strong>Note</strong>: The following command displays multiple prompts for certificate request information. For the <code>Common Name</code> prompt, enter <code>*.your-system-domain</code>.</p>

    <pre class="terminal">
    $ openssl req -new -key cf.key -out cf.csr
    </pre>

    <pre class="terminal">
    $ openssl x509 -req -in cf.csr -signkey cf.key -out cf.crt
    </pre>

    <pre class="terminal">
    $ cat cf.crt && cat cf.key
    </pre>

1. Update `REPLACE_WITH_SSL_CERT_AND_KEY` in your manifest with the value from the above command.


##<a id="config-cf-for-multi-az-deploy"></a> Configure AWS for multi AZ deploy, with RDS, Load Balancer

### Create additonal Subnets
Select _VPC_ and then "Subnets" can create the subnets in the screenshot below. The number of AZs you want to deploy to will decide how many subnets you need.

![subnets](http://content.screencast.com/users/geapi/folders/Jing/media/f6cf13e8-b3ac-4474-9c13-6211356b3c9c/00000018.png)


For a 2 zone deployment you need:

- cf1/cf2
- cf_elb1/cf_elb2
- rds_az1/rds_az2

The elb subnets need to be governed by a public route table so that they can talk to the blobstore, more on that [here](#multi-az-route-tables).
The cf subnets need to be on the NAT route table.
The CIDR blocks needed for each of these subnets can be seen in the screenshot.

###<a id="internet-gateway"></a> Create Internet gateway
You need an _Internet Gateway_ in order for your cf installation to talk to the internet, i.e. talking to the blobstores when deploying CF.
Once created it should look like this.

![Internet Gateway](http://content.screencast.com/users/geapi/folders/Jing/media/59f28af5-5c7b-4229-936c-20ff5abf1d8f/00000017.png)

### Create additional security group
From the dashborad click on _VPC_ and then on _Security Groups_. Once done it will looks like this:

![security groups](http://content.screencast.com/users/geapi/folders/Jing/media/1acc99e1-872c-4c96-a99e-ff9acd2afce4/00000021.png)

All security groups need to be within the VPC. The _cf_ng_ group can have the name _cf_  in your case. There should always be a default security group in the VPC.

The _cf_ security group needs the inbound rules as seen in the screenshot below. All UDP and TCP needs to be allowed back to itself.

![cf security group inbound rules](http://content.screencast.com/users/geapi/folders/Jing/media/c8f64d80-cc8b-451a-bc64-b16888a82f94/00000022.png)

The _nat_ security group's inbound and outbound rules.

![nat inbound](http://content.screencast.com/users/geapi/folders/Jing/media/2c80c2fb-b82c-464c-8867-43fe1e399e0f/00000023.png)

![nat outbound](http://content.screencast.com/users/geapi/folders/Jing/media/73eabe63-6719-41d7-a9f0-d38d58c46fff/00000024.png)

You might need to update the _bosh_ security group with some inbound rules see below. The outbound rules are the same as for the _nat_ security group.

![bosh inbound](http://content.screencast.com/users/geapi/folders/Jing/media/0b32a70e-4705-44c1-8de1-c6ca83d4e041/00000025.png)

The _web_ group needs to allow all traffic on a few ports. The outbound rules are the same as for the _nat_ security group.

![web inbound](http://content.screencast.com/users/geapi/folders/Jing/media/c47e49de-dc1f-4550-92e3-66ccde1548d1/00000026.png)


The _cf_ group needs to allow all traffic from the _nat_ group. The outbound rules are the same as for the _nat_ security group.

![cf inbound](http://content.screencast.com/users/geapi/folders/Jing/media/304d7c12-df4b-45a9-a88b-e1961b2053d5/00000027.png)

### Set up routes tables

Click on _VPC_ and then _Route Tables_.

**Default Route Table**

There should always be a default route table, it is associated with the original security group, usually named _microbosh_ or _bosh_. Picture below.
The route table's routes need to point to the internet gateway your created [here](#internet-gateway).

![default route table routes](http://content.screencast.com/users/geapi/folders/Jing/media/2392c16d-da14-441c-8cc2-c6fc40fc168d/00000028.png)

![default route table subnet associations](http://content.screencast.com/users/geapi/folders/Jing/media/306b5665-2b87-4a99-8c81-21dc02775607/00000029.png)


**Public route table**

![public route table routes](http://content.screencast.com/users/geapi/folders/Jing/media/608d4971-cd45-4910-96f9-b371fa1e254c/00000030.png)

![public route table subnet associations](http://content.screencast.com/users/geapi/folders/Jing/media/02f5dfdd-7af8-47b3-9af5-0e0acf72fede/00000031.png)

**Nat route table**

Needs to be associated with the Nat box you created [earlier](#nat-box). No need for services association unless you have that subnet from a services deployment.

![nat route table routes](http://content.screencast.com/users/geapi/folders/Jing/media/7ccae183-32e3-45e4-8a3e-e76ad17c899c/00000032.png)

![nat route table subnet associations](http://content.screencast.com/users/geapi/folders/Jing/media/992d22f1-2f1f-4185-ade7-503820eaad60/00000033.png)


### Create RDS DBs
Make sure to create the RDS subnets mentioned above first.
Go to _RDS_ from the dashboard, then first create a "subnet group" that uses the subnets _rds_az1/rds_az2_ above.

![rds subnet](http://content.screencast.com/users/geapi/folders/Jing/media/94a0985b-071c-4ac0-9240-e9e672ea8790/00000019.png)

Then create 3 databases (boshdb, ccdb, uaadb) in the AZ, with the rds subnet groups, you'll need to copy the db url and username/password for the manifest under the cc, uaa and bosh sections

![rds_instances](http://content.screencast.com/users/geapi/folders/Jing/media/572176f2-7108-45e9-9f95-877f55354f69/00000020.png)

### Create blobstores/s3 buckets
The following blosbstores need to be created via _S3_ -> _Create Bucket_, replacing 'arborglen' with your environment name.
(Make sure the region is set to the region you cf instance is running in)

![s3 buckets](http://content.screencast.com/users/geapi/folders/Jing/media/00a07190-010c-4065-8fbf-bc5ef13de696/00000015.png)

### Create load balancer
From dashboard go to _EC2_ click on _Load Balancers_ and then set one up, if you don't have a certificate, follow the steps [here](#create-cert) to create a cert first.

**Overview:**

![overview](http://content.screencast.com/users/geapi/folders/Jing/media/10cd06e6-bbda-4abb-9883-deff4990aeca/00000038.png)

1. ![details](http://content.screencast.com/users/geapi/folders/Jing/media/549f35fa-9b62-40f3-a554-4684880b4f74/00000037.png)

2. ![cert](http://content.screencast.com/users/geapi/folders/Jing/media/b305eb58-bdb1-4d35-b4d8-97e2a721a77e/00000039.png)

3. ![cipher](http://content.screencast.com/users/geapi/folders/Jing/media/7ad0dbb3-ef60-4566-bfbe-ef04065c98e9/00000034.png)

4. ![healthcheck](http://content.screencast.com/users/geapi/folders/Jing/media/4350f9a2-9a6d-408b-8c40-649db0035f63/00000035.png)

5. Select the two subnets you created for the ELB.
![subnets](http://content.screencast.com/users/geapi/folders/Jing/media/226feb5d-6358-4285-8a85-b1a39bf98736/00000040.png)

6. ![security group](http://content.screencast.com/users/geapi/folders/Jing/media/a8ea3cd4-9b5d-43ef-a967-5775d9dd80d6/00000041.png)

7. Nothing to be selected here, will happen automatically when deploying cf with bosh
![ec2 instances](http://content.screencast.com/users/geapi/folders/Jing/media/ff5fb68f-1397-4c58-a6cb-3778d77cc1b1/00000042.png)

8. Nothing to be added here.
![add tags](http://content.screencast.com/users/geapi/folders/Jing/media/71e9591b-b6fd-48af-a0ed-eba0eb7e179d/00000036.png)

9. Check that everything looks good.
![review](http://content.screencast.com/users/geapi/folders/Jing/media/e34a637c-906b-4207-a63d-8646c8afdd6a/00000043.png)

10. ![sucees dialog](http://content.screencast.com/users/geapi/folders/Jing/media/6bdf4c24-3f44-4884-ac8e-f2d5e8c23561/00000044.png)


####<a id="create-cert"></a>  Create self signed cert for load balancer
Make sure openssl is installed. Then.

```
openssl genrsa 2048  -out MY_CERT_cert.pem > MY_CERT_cert.pem
openssl req -new -key MY_CERT_cert.pem -out csr.pem
openssl x509 -inform PEM -in csr.pem
```

```
openssl x509 -req -days 365 -in csr.pem -signkey MY_CERT_cert.pem -out server_MY_CERT.crt

```

- `MY_CERT_cert.pem` contains the private key.
- `server_MY_CERT.crt` contains the public key certificate.

Back to [Deploying to AWS](aws_steps.html)

Next: [Deploying Cloud Foundry on AWS](deploy_aws_cf.html)