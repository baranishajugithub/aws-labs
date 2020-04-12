# Designing Network and Compute architecture for a Highly Available Web Application.

## Lab 01 – Part 01 of 03

Private networks in AWS are created by the service called Amazon Virtual Private Cloud (VPC). We can create a VPC within any AWS region of our choice and within that VPC we define subnets to manage related sets of servers or other AWS resources. VPC service lets us define rules for how network traffic from our subnets is routed. We can also decide whether our private network (VPC) should be connected to the Internet, to corporate networks or to keep the network completely private.

In this lab we are going to design the network for a highly available two tier web application. The web servers will be deployed in two public subnets across two availability zones having Internet connectivity and DB servers will be deployed in two private subnets across two availability zones. The DB servers will use network address translation (NAT) service for outbound Internet connectivity. The incoming and outgoing traffic will be filtered by virtual firewalls; by NACL at Subnet level and by Security Groups at instance/resource level.

The network after building, will look similer to the picture below

 ![](https://github.com/ashydv/aws-labs/blob/master/images/NetworkDiagram.png)

### Activity 01 – Creating a VPC

Login to your AWS account and find VPC under Networking & Content Delivery category.  
Click on Your VPCs in the side bar and then click on Create VPC.  

_Did you notice that a VPC (default VPC) was already created? Find out what other resources were automatically created for you in VPC and why._

Now you need to give a name to your VPC and select a CIDR notation.

* Name tag: MyVPC
* IPv4 CIDR block: 10.0.0.0/16
* IPv4 CIDR block: No IPv6 CIDR Block
* Tenancy: default
* Click on Yes Create.

Select MyVPC and click on Action drop-down.

* Ensure that Edit DNS Resolution is selected as yes/box is checked.
* Ensure that Edit DNS Hostnames is selected as yes/box is checked..

### Activity 02 - Creating Subnets

We would now create one public and one private subnet in both availability zones for high availability of our resources.
Click on Subnets in the sidebar of the VPC Dashboard, click on Create Subnet

* Name tag: MyPrivateSubnet01
  * VPC: MyVPC
  * Availability Zone: _choose the first one you see_
  * IPv4 CIDR block: 10.0.1.0/24
  
Click on Yes, Create. Your new subnet should have been created now and show up on the screen.  
Repeat the same steps to create 3 more Subnets with below configuration.

* Name tag: MyPrivateSubnet02
  * VPC: MyVPC
  * Availability Zone: _choose the second one you see_
  * IPv4 CIDR block: 10.0.2.0/24
* Name tag: MyPublicSubnet01
  * VPC: MyVPC
  * Availability Zone: _choose the first one you see_
  * IPv4 CIDR block: 10.0.3.0/24
* Name tag: MyPublicSubnet02
  * VPC: MyVPC
  * Availability Zone: _choose the second one you see_
  * IPv4 CIDR block: 10.0.4.0/24
  
Once all the subnets are created, select MyPublicSubnet01 and click on the Subnet Actions dropdown; go to Modify auto-assign IP settings and check Enable auto-assign public IPv4 address box.  
Click on Save. Repeat the same step for MyPublicsubnet02 as well. Do not enable this setting for Private Subnets.

_Why is the available number of IPs showing as 251, where are the rest 5 IPs used?_  
_Why have we created two private and public in different subnets? Should we not create both Public subnets in one AZ and both Private in another AZ?_

### Activity 03 - Create Internet gateway

As you might have noticed, there were similar steps taken in creating the Public and Private subnets, what differentiates them?
A public subnet is the one that has a route to Internet Gateway in its routing table whereas Private Subnets don't. So now let's create an Internet Gateway.
Click on Internet Gateways in the sidebar of VPC Dashboard and then click on Create Internet Gateway.

* Name tag: MyIGW
* Click on Yes, Create.

You will see the state of MyIGW as detached as it is not yet attached to a VPC.  
Select the MyIGW and click on Attach to VPC, select MyVPC from the drop-down in next pop up and click
on Yes, Attach.

_Why was the default VPC not showing in the dropdown._

### Activity 04 - Create Route table (public) and assign to relevant Subnets

Click on Route tables in the side bar, you should see a Route Table already created for you, assigned to MyVPC.

* Click on Create Route Table
  * Name tag: MyPublicRoute
  * VPC: MyVPC
* Click on Yes, Create.

A new route table would have come up now.

While the MyPublicRoute selected, click on Routes tab in the lower half of the screen. You would see that it already has an entry for local traffic. We now should add the route entry meant for non local traffic (Internet).
Click on Edit and then on Add route. Fill in the below details in the new blank route table entry.

* Destination: 0.0.0.0/0
* Target: Internet Gateway (you would see the Internet gateway name in the drop down)

Click on Save routes.

This way we have added an entry to Internet in our public route table, now is the time to assign the route table to our public subnets.

* Click on the Subnet Associations tab right next to the Routes tab.

_You would see that all four subnets that you created are associated with the main route table, why?_

* Click on Edit subnet associations and select the two Public Subnets that you created. Save.

Let us now create different 'Security Groups' for App server, database and load balancer. We would leverage them in coming steps.
In the navigation pane find and click on 'Security Groups'

* Click on 'Create Security Group'
  * Security group name: My-App-SG
  * Description: This SG is to be used for web application servers.
  * VPC: MyVPC
* Click on Create

Create two more security groups with following configurations --

* Click on 'Create Security Group'
  * Security group name: My-DB-SG
  * Description: This SG is to be used for database servers.
  * VPC: MyVPC
* Click on Create

Select either of the Security Group now and click on 'Inbound Rules' tab.
Click on 'Edit Rules' and add rules for incoming traffic on the security groups like mentioned below.

#### My-App-SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Custom | 0.0.0.0/0 |
| HTTPS |   TCP    |    443     | Custom | 0.0.0.0/0 |
|  SSH  |   TCP    |     22     | Custom | 0.0.0.0/0 |

#### My-DB-SG

|     Type     | Protocol | Port Range | Source |           |
| :----------: | :------: | :--------: | :----: | :-------: |
| MYSQL/Aurora |   TCP    |    3306    | Custom | My-App-SG |
|     SSH      |   TCP    |     22     | Custom | 0.0.0.0/0 |

#### My-ALB-SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Custom | 0.0.0.0/0 |
| HTTPS |   TCP    |    443     | Custom | 0.0.0.0/0 |

For now, our VPC configuration is complete. The instances launched in our public subnets should have outbound access to Internet and the instances in our private subnet should not. We would verify the same in the next section.

## Lab 01 – Part 02 of 03

### Activity 05 - Creating EC2 instances

We are now going to create 2 EC2 instances. One each as MyAppServer and MyDBServer. Let us switch to EC2 Dashboard now and click on Launch Instance.

Creating the App Server

* Amazon Machine Image: "Amazon Linux 2" (free tier eligible)
* Instance Type: t2.micro
* Configure Instance Details: Select the below mentioned points and leave everything else as default.
  * Network: MyVPC
  * Subnet: MyPublicSubnet01

**Expand the Advance Details section and paste the following script in the user data section.**

```bash
#!/bin/bash
yum install httpd mysql git -y
amazon-linux-extras install -y php7.2
cd /var/www/html
git clone https://github.com/ashydv/bikerhood.git
mv /var/www/html/bikerhood/*  /var/www/html/
service httpd start
chkconfig httpd on
```

This script will –

1 - Install an Apache web server and create a Hello World! page.  
2 - Activate the Web server and configure it to automatically start on reboots.

* Add Storage: Leave defaults
* Add Tags
  * Key: Name
  * Value: MyAppServer
* Configure Security Group: Select existing -> My-App-SG
* Click on Review and Launch.

On the next page check that your AMI is free tier eligible and Instance Type is showing as t2.micro.

* Click on Launch.

On the next window, create a new Key Pair, Key pair name: mykey and then Download Key pair.

Finally click on Launch Instance

We have just created one EC2 instance in our public subnet as MyAppServer, now we would create another EC2 instance in private subnet as MyDBServer following similar steps.  

Go back to EC2 Dashboard now and click on Launch Instance.

Creating a server in the private subnet for Database installations

* Amazon Machine Image: "Amazon Linux 2" (free tier eligible)
* Instance Type: t2.micro
* Configure Instance Details: select the below mentioned points and leave everything else as default.
  * Network: MyVPC
  * Subnet: MyPrivateSubnet01
* Add Storage: Leave defaults
* Add Tags
  * Key: Name
  * Value: MyDBServer
* Configure Security Group: Select existing -> My-DB-SG
* Click on Review and Launch.

On the next page check that your AMI is free tier eligible and Instance Type is showing as t2.micro.

* Click on Launch.

On the next window, select to use existing Key Pair 'mykey'.

* Click on Launch Instance

Go back to your EC2 instance page. You should see your two instances.

_Did you notice that your MyAppServer has got public IP and public DNS while MyDBServer has not, why?_  
_Why are all instances running in the same AZ?_

Try browsing the public DNS/IP of the web server, does it open? Yes, because you have opened the traffic on port 80 from anywhere. Ideally it should be open only to the traffic coming from the load balancer, we will do that in next lab.

Go through the various options under Action drop down. 

Since we have installed our application and did the configurtion already through user data, let us save it as an image for our future use so we dont have to start from scrach.

On the EC2 dashboard, select the instance, go to the Action dropdown, name your Image and create image. Note the AMI ID, we will use it later.


### Activity 06 - Verifying the connectivity

We now have created EC2 instances across public and private subnet. We would now verify whether our network configuration is working as desired.

Let us try to SSH to the MyAppServer.

We will use PUTTY, you can download the portable version from [here](https://the.earth.li/~sgtatham/putty/latest/w64/putty.zip) and extact the zip file which contains multiple small tools, we will use three of them.

* Step 01 - Converting the .pem key into a .ppk  
PUTTY does not support the private key format (.pem) generated by Amazon EC2. We will use  PUTTYgen to convert out mykey.pem to mykey.ppk
  * Start PUTTYgen from the PUTTY folder.
  * Select RSA under type of key to generate in the bottom left corner.
  * Click on Load. By default, PUTTYgen displays only files with the extension .ppk. To locate your .pem file, select the option to display files of all types. Select mykey.pem and then click Open. Click OK to dismiss the confirmation dialog box.
  * Click Save private key. PUTTYgen displays a warning about saving the key without a passphrase. Click Yes.
  * Specify the same name for the key that you used for the key pair, in our case mykey. PUTTY automatically adds the .ppk file extension.

Our private key is now ready, we will use this to connect to our instance in further steps.

* Step 02 - adding our key in PAGEANT for ssh key forwarding.  
Pageant is one of the tools present in the zip. It is an SSH authentication agent which holds your private keys in memory. We will add our key here in order to login to our EC2 instance in private subnet.
  * Open PAGEANT.EXE from the putty folder.
  * It will put an icon of a computer wearing a hat into the System tray. Right click and click on Add Key
  * In the window that opens, browse and select your mykey.ppk and click open. The key gets added without showing any confirmation.

* Step 03 - connecting to EC2 instance using putty.

  * Start PuTTY.
  * In the Category pane, click Session.
  * In the Host Name text box, type the public IP address of your MyBastionHost(Public IP of MyAppServer)
  * In the Category pane, click on Connection and type 30 in the keepalive box which by defalt has 0 written in it.
  * In the Category pane, expand Connection, expand SSH, and then click Auth.
  * Check the Allow agent forwarding box.
  * Click Browse and select mykey.ppk file you generated in previous steps and then click Open.
  * Click Open to start the PuTTY session. PuTTY will ask whether you wish to cache the server’s host key. Click Yes.

```bash Lobin user name: ec2-user ```

You are now connected to MyAppServer EC2 instance. From here you can jump on to MyDBServer.

Try pinging google.com from here. It will work because MyAppServer is a part of Public Subnet which has internet access through IGW.  

We will now login to MyDBServer and check the same.

```bash
ssh ec2-user@ <Private IP of My DB Server DNS end point of MyDBServer instance>
``` 

Once you are in the MyDBServer, if you try pinging google.com it will fail. That proves that we do not have out bound internet connectivity from the instances in Private Subnets. You can use either use a NAT instance or NAT Gateway service to enable out bound internet connectivity from the instances in Private Subnets, remember NAT is only for outbound and Jump Servers/Bastion hosts are used for inbound remote access to the instances in Private Subnets.

Terminate the instance, we are simulating a disaster now!

## Lab 01 – Part 03 of 03

You will now be launching this application using a Launch Configuration into an Auto Scaling Group, the ASG will automatically grow and shrink the number of your servers based on the user defined threshold. The requests to your application will be distributed by Application Load Balancer.

### Activity 07 - Creating an Auto Scaling Group

Go to the Auto Scaling section in your EC2 dashboard and click on Create Launch Configuration.

- AMI: Select the AMI from MyAMI section that you created in previous steps.
- Instance Type: t2.micro
- Name: MyAppServer_V01_LC

Go next.

- No additional storage, go next.
- On the security group page, choose My-App-SG
- Click on Create launch configuration
- Select the existing 'mykey' and Create launch configuration

Your Launch Configuration is created, let us now create the auto scaling group. Click on Create an Auto scaling group using this Launch configuration.

- Group name: MyApp_ASG
- Group size: Start with 2 instances
- Network: MyVPC
- Subnet: Select both the public subnets here.
- Configure scaling policies: Use scaling policies to adjust the capacity of this group
- Scale between 2 and 6 instances.
- Target value: 60
- Instances need: 10
- Next Configure Notification
- Add Notification: Create Topic
- Send a notification to: MyASG_Topic
- With these recipients: 'your email ID'
- Next Create a Tag with 'Key: Name' and 'Value: MyAppServer'
- Review: Create Auto Scaling group

Click on Close, you would be directed to the Auto Scaling Groups Dashboard. Explore the Activity History and other tabs.

You have just launched our highly available BikerHood application in an Auto Scaling Group. You can open the public IP addresses of both the instances in separate browser and see what happens. You should be seeing the webpage with same information but look at the bottom. You should see some instance related information. This way you can identify which server your request is being served from.

Also check if you received an email from SNS topic, you need to confirm the subscription.

But the end users would not have IP addresses of your server right? They should have a domain name to go to. Let us create a Load balancer that will divert the traffic to both these instances in round robin method.

### Activity 08 - Creating an Application Load Balancer

Go to the Load Balancing section of EC2 dashboard and click on Target Group

- Create Target Group
- Target group name: MyTG
- VPC: MyVPC
- Leave rest defaults and click Create.

Let us register our instances in ASG with the MyTG target group. Select your ASG and go to action dropdown and click on edit. You will find a field for target group in the lower section. Click on the empty field and assign MyTG. Save it (save button is towards the top right of lower section)

Click on Load Balancers: Create Load Balancers

From next screen, create an Application Load Balancer

- Name: MyALB
- Scroll down to the Availability Zones Section
- Select the VPC in which you have launched the ASG
- Select Public Subnets from both AZs. This is a critical step, reconfirm before going forward.
- Next - Configure Security Settings – Ignore the warning, it is recommending to have SSL certificate.
- Next - Configure Security Groups. Select My_ALB_SG from existing ones.
- Next Configure Routing –
- Target group – Existing Target Group
- Name: MyTG
- Leave rest defaults - Register Targets – Review – Create

Click on close and it will take you to the load balancer dashboard, you should see the DNS endpoint of your load balancer in Description Tab. ALB takes a little time to come up. Refresh till you see the state as active.

Open the DNS address of your ALB in a browser and notice what it shows. It is now diverting the traffic to both your instances. You can see the behavior of load balancer while you refresh the page and notice the instance ID.

So at this point of time your application can be reached by the load balancer endpoint as well as direct IPs of the instances. This is not an ideal behavior, we should not allow our app servers to accept traffic from anywhere else apart from the load balancer in order to ensure security.

Can you restrict it?

### Activity 09 – Modify the Security Groups to ensure security on incoming traffic

Update the **My_App_SG** security group settings as shown below.

#### My_App_SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Custom | My_ALB_SG |
| HTTPS |   TCP    |    443     | Custom | My_ALB_SG |
|  SSH  |   TCP    |     22     | Custom | 0.0.0.0/0 |


If your application is reachable only through the load balancer endpoint and not through visiting the IP addresses or EC2 instances in browser, you have done it well.

You can also now try deleting one/more server in order to verify whether the auto scaling feature is able to spin up instances in response.

### Clean up steps –

Delete the resources in the below order

 1: ALB  
 2: Auto Scaling Group (takes little time to delete)  
 3: Target Group  
 4: Launch Configuration  
 5: Deregister AMI  
 6: EBS Snapshot  

***All the services used in this lab are eligible and covered within the free tier account. There should not be any charge if you delete all the resources within a couple of hours of creation provided you have monthly limits left.***

### Lab Complete!
