---
title: 'Establish Site-to-Site VPN Connection'
menuTitle: 'Establish VPN Connection'
disableToc: true
weight: 40
---

{{% comment %}}
Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
SPDX-License-Identifier: CC-BY-SA-4.0
{{% /comment %}}

This section provides detailed step-by-step instructions for using AWS Site-to-Site VPN and AWS Transit Gateway as a means to quickly and securely establish network connectivity between your on-premises and AWS environments. 

By following these instructions, you will enable network connectivity between your on-premises environment and the centrally managed development VPC you established earlier in this guide.

Later in this guide, when you set up your test and production VPCs, the steps required to enable those VPCs to reuse your site-to-site VPN connection will be addressed.  The process in your AWS environment will be largely a repeat of the steps in this section that are used to connect your development VPC to your on-premises network.

{{% notice tip %}}
**Set up a virtual on-premises environment:** If you don't have access to an actual on-premises data center environment and you'd like to either evaluate or demonstrate the AWS Site-to-Site VPN capabilities, see [Setting Up a Virtual On-Premises Environment]({{< relref "05-virtual-on-premises" >}}) for instructions on how to set up a test on-premises environment in AWS.  You can deploy either open source or commercial VPN/router appliances in your virtual on-premises environment to act as the customer gateway device.
{{% /notice %}}

{{< toc >}}

## 1. Ensure Pre-requisites Are Satisfied

First, ensure that the following pre-requisites are satisfied:

### Engage Your On-Premises Network Team

In order to effect the necessary on-premises network configuration changes required by your AWS Site-to-Site VPN connection, you'll need to engage your Network team.  It's recommended that you review the overall requirements and solution design with them before proceeding with the configuration work.

### Use Non-overlapping IP Addresses

When you initially established your common development VPC, it was recommended that you use an IP range for CIDR block that does not overlap with other CIDR blocks in use by your organization. If you were not able to obtain a non-overlapping CIDR block or blocks for your AWS environment, it's recommended that you attempt to do so before proceeding further.  Otherwise, you'll need to prepare to perform some extent of Network Address Translation (NAT) in your on-premises environment to manage the use of overlapping IP address ranges.

See [Obtaining Non-Overlapping IP Address Range]({{< relref "04-address-prerequisites#ip-address-range" >}}) for more background and guidance.

### Obtain Static Public IP Address for Your Customer Gateway

In your on-premises environment, you will need to identify a static public IP address that will be associated with your Customer Gateway that will act as the on-premises side of the VPN site-to-site connection.  You'll use this IP address in the subsequent steps when you register your Customer Gateway in your AWS environment.

### Determine Dynamic or Static Routing

You'll need to work with your Network team to determine whether dynamic or static routing will be used with your Site-to-Site VPN connection. You can review [Site-to-Site VPN routing options](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPNRoutingTypes.html) for more details.

#### Dynamic Routing and Route Based VPNs

When you choose dynamic routing, you'll configure the Site-to-Site VPN connection to use Border Gateway Protocol (BGP) to automatically advertise routes across the connection.  In this scenario, you'll be using what is referred to as a "route based VPN".

It's recommended that you use BGP-capable customer gateway devices, when available, because the BGP protocol offers robust liveness detection checks that can assist failover to the second VPN tunnel if the first tunnel goes down.

In the dynamic routing and route based VPN configuration, both tunnels can be up at the same time. The BGP keep alive feature will bring up and keep the tunnel UP and active all the time. 

#### Static Routing and Policy Based VPNs

If your customer gateway device does not support BGP, then you will need to use static routing.

In this configuration only one tunnel will be up at a given time.  Depending on the features of your customer gateway device, you may be able to configure it to:
* Periodically send keep alive traffic to keep the tunnel active.
* Force a failover to the other tunnel in case of issues with the current tunnel.

## 2. Review Resources to Configure

In this section you'll be configuring resources in the **`network-prod`** AWS account that you established earlier.  

In later sections, you'll be creating test and production VPCs in other AWS accounts and taking steps to attach those VPCs to the transit gateway that you're establishing in the **`network-prod`** AWS account.

The resources you'll be configuring in this section within the **`network-prod`** AWS account include:
* Customer gateway
* Transit gateway
* VPN transit gateway attachment
* VPC transit gateway attachment
* VPC route table entries

The VPC transit gateway attachment will attach your centrally managed development VPC with the transit gateway so that network traffic can be exchanged between the VPC and your on-premises network.

You'll need to update the VPC's route tables so that traffic from the development VPC and destined for your on-premises environment is router to the transit gateway.

An optional step is to remove the public subnets and NAT gateways from the development VPC and route all Internet egress traffic from the development VPC to your on-premises network. You might choose this option as a short-term solution to direct the egress traffic through your existing network security filtering services in your on-premises environment. Longer term, you would likely host those filtering services in your AWS environment.

## 3. Create Customer Gateway

Register your on-premises customer gateway device in AWS.

1. As a Cloud Administrator, use your personal user to log into AWS SSO.
2. Select the **`network-prod`** AWS account
3. Select **`Management console`** associated with the **`AWSAdministratorAccess`** role.
4. Select the appropriate AWS region.
5. Navigate to **`VPC`**
6. Select **`Customer Gateways`**
7. Select **`Create Customer Gateway`**
8. Provide the Customer Gateway details:

|Field|Recommendation|Notes|
|-----|---------------|----|
|**`Name`**|`infra-on-prem-dc1-gw1`|Accounting for the possibility of multiple customer gateway devices in each of multiple data centers.|
|**`Routing`**|`Dynamic Routing`||
|**`BGP ASN`**|`6500`||
|**`IP Address`**|Public IP Address of your on-premises customer gateway||   
|**`Certificate ARN`**|Leave empty||
|**`Device`**|Leave empty||

{{% notice info %}}
**Pre-shared Key and Certificate Options:** These example set up instructions use the pre-shared key option to authenticate your Site-to-Site VPN tunnel endpoints.  If you don't want to use pre-shared keys, you can use a private certificate from AWS Certificate Manager Private Certificate Authority to authenticate your VPN endpoints. See [Site-to-Site VPN tunnel authentication options](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-tunnel-authentication-options.html).
{{% /notice %}}

## 4. Create a Transit Gateway

1. Select **`Transit Gateways`**
2. Select **`Create Transit Gateway`**
3. Provide the Transit Gateway details:

|Field|Recommendation|Notes|
|-----|---------------|----|
|**`Name tag`**|infra-main|You'll be able to use a single transit gateway for both on-premises integration and VPC-to-VPC routing if necessary.|
|**`Description`**|||
|**`Amazon side ASN`**|Consult your Network team|One consideration is to use a unique private ASN per transit gateway to provide more flexible routing options.|
|**`DNS support`**|checked||
|**`VPN ECMP support`**|checked|Enables you to use multiple VPN connections to aggregate VPN throughput.|
|**`Default route table association`**|**Unchecked**|Uncheck this option so you can specify which routing is allowed between accounts and environments|
|**`Default route table propagation`**|**Unchecked**|Uncheck this option so that you don't enable any-to-any connectivity between attachments by default.|
|**`Auto accept shared attachments`**|Unchecked||

4. Select **`Create Transit Gateway`**

{{% notice info %}}
**Default Table Association and Propagation:** Refer to the following VPC guides in regards to isolated VPCs and shared services. See [Isolated VPCs](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-isolated.html) and [Isolated VPCs with shared services](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-isolated-shared.html).  In cases where you want development environments to be able to communicate to one another (or like for like environments), but do not want dev to talk to test and/or prod, you will want to manage the route table associations manually.  This will take some additional effort, however, will ensure that you proper environment communication isolation in place.
{{% /notice %}}


## 5. Create VPN Transit Gateway Attachment

1. Select **`Transit Gateway Attachments`**
2. Select **`Create Transit Gateway Attachment`**
3. Provide the Transit Gateway Attachment form details:

|Field|Recommendation|
|-----|---------------|
|**`Transit Gateway ID`**|Select your transit gateway|
|**`Attachment Type`**|VPN|
|**`Customer Gateway`**|Existing|
|**`Customer Gateway ID`**|Select value from the list that matches the ID of the Customer Gateway you just created.|
|**`Routing options`**|Dynamic (requires BGP)|
|**`Enable acceleration`**|Leave unchecked|
|**`Tunnel Options`**|Leave unchecked|

4. Select **`Create attachment`**.
5. Once you're returned to the list of attachments, select the **`Name`** cell of the newly created attachment and assign a name to the attachment. For example, **`infra-on-prem-dc1-gw1`**, the same name as your customer gateway resource. 

{{% notice info %}}
**Site-to-Site VPN Connection:** As a result of the VPN attachment being provisioned, you will notice in the Site-to-Site VPN Connections area of the console that a new connection resource has been created. If you review the **`Tunnel Details`** of the connection, it will show both tunnels in the **`DOWN`** state because you have not yet configured the on-premises side of the connection.
{{% /notice %}}

## 6. Configure On-premises Side of the VPN Connection

Your next step is to obtain configuration data from the newly created site-to-site VPN connection and use it to configure your on-premises customer gateway device.

### Download VPN Configuration File

1. Select **`Site-to-Site VPN Connections`**
2. Select the connection that was just created
3. Download the VPN configuration information by clicking **`Download Configuration`**. 
4. Select your Vendor, Platform, and Software of your customer gateway device from the menus.  If your specific device is not available, select **`Generic`**.

### Configure Your On-Premises Customer Gateway

Use the configuration data to configure your on-premises customer gateway. See [Your customer gateway device](https://docs.aws.amazon.com/vpn/latest/s2svpn/your-cgw.html) in the [AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn) documentation for details.

### Configure Routing in Your On-Premises Environment

Ensure that your on-premises router configuration has been updated to route network traffic destined for the CIDR ranges allocated to your AWS environment to your customer gateway.

## 7. Confirm Status of the VPN Connection

After your on-premises customer gateway has been configured, check the status of your VPN connection.

1. Select **`Site-to-Site VPN Connections`**
2. Select the connection that was just created
3. Select **`Tunnel Details`**.
4. Monitor the status of the tunnels.  After several minutes, at least one of the two tunnels should transition to the **`UP`** state.

If at least one of the tunnels does not come up, then see [Troubleshooting your customer gateway device](https://docs.aws.amazon.com/vpn/latest/s2svpn/Troubleshooting.html)

## 8. Create Transit Gateway Route Tables

If you configured your site-to-site VPN connection to use BGP, then your customer gateway and the transit gateway will propagate routes to each other. However, your on-premises routes will not be automatically propagated from the transit gateway to you VPC route tables of the attached VPCs. You need to manually configure these route entries.

Review the transit gateway route table to verify that routing information for the on-premises CIDR is included.

1. Select **`Transit Gateway Route Tables`**
2. Click **`Create Transit Gateway Route Table`**
3. Provide the Transit Gateway Route Table details:

|Field|Recommendation|
|-----|---------------|
|**`Name`**|tgw-vpn-rt|
|**`Transit Gateway ID`**|Select your transit gateway from the list|

Click **`Create Transit Gateway Route Table`**

Once the Transit Gateway Route Table is **`available`**, click on the **`Associations`** table at the bottom of the screen.
1. Click **`Create association`**
2. Choose the attachment to associate

|Field|Recommendation|
|-----|---------------|
|**`Choose attachment to associate`**|Choose the transit Gateway attachment id for the VPN attachment|

Click **`Create association`**

Once the Association is **`associated`**, click on the **`Propagations`** tab
1. Click **`Create propagation`**
2. Choose the attachment to propagate

|Field|Recommendation|
|-----|---------------|
|**`Choose attachment to propogate`**|Choose the transit Gateway attachment id for the VPN attachment|

Click **`Create propagation`**

Click on the **`Routes`** tab and you should see that there is a route entry for your on-prem/VPN CIDR.
(When using BGP, the CIDR block of the attached VPC(s) should appear in the routing tables in your on-premises customer gateway.)

## 9. Enable resource sharing in your master account

You will need to enable sharing resources within your AWS Organizations from your master account.  This will allow you to access Transit Gateway from each of your accounts created and maintained within Control Tower.

1. As a Cloud Administrator, use your personal user to log into AWS SSO.
2. Select the **`master`** AWS account
3. Select **`Management console`** associated with the **`AWSAdministratorAccess`** role.
4. Select the appropriate AWS region.
5. Navigate to **`Resource Access Manager`**
6. Click the **`Settings`** in the left navigation panel
7. Check the **`Enable sharing within your AWS Organizations`** checkbox, then click the **`Save settings`** button

{{% notice info %}}
**Find your Organization ID:** Navigate to AWS Organizations and click on any of the accounts to open the information panel on the right.  Within the **`ARN`**, after the word account, you will see your Organization ID (starts with **`o-`**).  Note this down for the next set of steps.
{{% /notice %}}

## 10. Sharing your Transit Gateway to your other accounts

You will use AWS Resource Access Manager (RAM) to share your Transit Gateway for VPC attachments across your accounts and/or your organizations in AWS Organizations.

1. Navigate to **`Resource Access Manager`**
2. Click the **`Create a resource share`** button
3. Fill out the resource share form details.  Some suggested fields are below:


|Field|Recommendation|
|-----|---------------|
|**`Name`**|infra-network-tgw-share-01|
|**`Select resource type`**|Transit Gateways (and select your Transit Gateway)|
|**`Allow external accounts`**|(checked)|
|**`Add AWS account number, OU or organization`**|Enter the Organization ID from your master account (captured in the above steps)|



## 11. Create VPC attachments to your Transit Gateway

You will create VPC attachments to your Transit Gateway within each individual account.  THis is available now that you have shared the Transit Gateway via Resource Access Manager to your root organization.

1. As a Cloud Administrator, use your personal user to log into AWS SSO.
2. Select the **`dev`** AWS account
3. Select **`Management console`** associated with the **`AWSAdministratorAccess`** role.
4. Select the appropriate AWS region.
5. Navigate to **`VPC`**
6. Click the **`Transit Gateway Attachments`** in the left navigation panel
7. Click the **`Create Transit Gateway Attachment`** button
8. Fill out the resource share form details.  Some suggested fields are below:


|Field|Recommendation|
|-----|---------------|
|**`Transit Gateway ID`**|Select your transit gateway|
|**`Attachment type`**|VPC|
|**`Attachment tag name`**|tgw-dev|
|**`DNS support`**|checked|
|**`IP v6 support`**|only check if you are using IP v6 - otherwise, leave unchecked|
|**`VPC ID`**|Choose the VPC within your shared dev account that you want to attach|
|**`Subnet IDs`**|check any private subnets in the VPC that you want as part of the attachment|

{{% notice info %}}
**Repeat for other VPCs:** Repeat this step for any other accounts and VPCs that you want attached to the Transit Gateway.
{{% /notice %}}

## 12. Accept the Transit Gateway attachment requests

1. As a Cloud Administrator, use your personal user to log into AWS SSO.
2. Select the **`infrastructure`** AWS account
3. Select **`Management console`** associated with the **`AWSAdministratorAccess`** role.
4. Select the appropriate AWS region.
5. Navigate to **`VPC`**
6. Click the **`Transit Gateway Attachments`** in the left navigation panel

Within the table, you will see the list of your new **`pending acceptance`** transit gateway attachments.

1. Click the checkbox next to a row where the status is **`pending acceptance`** 
2. Click on the **`Actions`** button and select **`Accept`**
3. A prompt will popup - click **`Accept`** to confirm acceptance of the attachment


## 13. Create your VPC Transit Gateway Route tables for your Shared Dev environment

1. Select **`Transit Gateway Route Tables`**
2. Click **`Create Transit Gateway Route Table`**
3. Provide the Transit Gateway Route Table details:

|Field|Recommendation|
|-----|---------------|
|**`Name`**|tgw-ssdev-rt|
|**`Transit Gateway ID`**|Select your transit gateway from the list|

Click **`Create Transit Gateway Route Table`**

Once the Transit Gateway Route Table is **`available`**, click on the **`Associations`** table at the bottom of the screen.
1. Click **`Create association`**
2. Choose the attachment to associate

|Field|Recommendation|
|-----|---------------|
|**`Choose attachment to associate`**|Choose the transit Gateway attachment id for the VPC attachment|

Click **`Create association`**

Once the Association is **`associated`**, click on the **`Propagations`** tab
1. Click **`Create propagation`**
2. Choose the attachment to propagate (your VPC attachments setup in Step 11 - Repeat for each association to be added)

Repeat for each association to be added

|Field|Recommendation|
|-----|---------------|
|**`Choose attachment to propogate`**|Choose the transit Gateway attachment id for the VPC attachment|

Click **`Create propagation`**

Click on the **`Routes`** tab and you should see that there is a route entry for your VPC CIDR.


## 14. Update VPC Route Table

In this step, you'll update the route table for each of the private subnets in the **`dev-infra-shared`** VPC to add an entry to route traffic destined to your on-premises network to the transit gateway.

1. Select **`Route Tables`**
2. For each of the private subnet, modify the route table:
  1. Select route table entry in the list
  2. Select the **`Routes`** tab
  3. Select **`Edit routes`**
  4. Add a new route:

|Field|Recommendation|
|-----|---------------|
|**`Destination`**|Specify the CIDR range of your on-premises network|
|**`Target`**|Select your transit gateway. For example, **`infra-main`**|

## 15. Test Connectivity

One of the easiest means to test basic connectivity is to deploy either a Linux or Windows EC2 instance to one of the development network's private subnets and attempt to ping between that instance and one or more OS instances hosted in your on-premises network.

See [Testing the Site-to-Site VPN connection](https://docs.aws.amazon.com/vpn/latest/s2svpn/HowToTestEndToEnd_Linux.html).

## 16. Monitor Your VPN Connection

See [Monitoring Your Site-to-Site VPN connection](https://docs.aws.amazon.com/vpn/latest/s2svpn/monitoring-overview-vpn.html).
