# AWS Solution for a 2-company Website Using Reverse Proxy Technology


## Objective:
- Buuild a secure infrastructure in AWS VPC for a company that uses Wordpress CMS for its main company site and a tooling website for its DevOps team. 
- Reverse proxy technology with Nginx has been selected with the aim to improve security and performance.
- Cost, reliability and scalability are the major considerations for the company, to make the infrastructure resilient to server failures, accomodate increased traffic   and have reasonable cost.

Infrastructure:

![project-15-infrastructure IMG](https://user-images.githubusercontent.com/23315232/135461812-e94e31b1-526b-4950-b82a-e910ee53773c.png)
Credits: Darey.io

### Initial setup
- Create a subaccount in AWS to manage all resources for the company's AWS solution and assign any appropriate name e.g 'DevOps'
- From the root account, create an organization unit (OU) and move the subaccount into the OU. We will launch Dev resources in the subaccount

<img width="938" alt="creating_organization_unit_ou_and_Adding_devops_account" src="https://user-images.githubusercontent.com/23315232/135610014-f847f9ce-e04e-4ba6-be4e-a84f0b77c2c5.png">

- Create a domain name for the company website on domain name providers. You can obtain a free domain name from freenom website
- Create a hosted zone in AWS and map the hosted zone name servers to the domain name.

<img width="639" alt="creating_route_53_hosted_zone_in_aws" src="https://user-images.githubusercontent.com/23315232/135609723-01b7ba48-f7ac-4f63-9d34-dc98716a8938.png">

<img width="663" alt="mapping_hosted_zone_name_Servers_to_domain_name" src="https://user-images.githubusercontent.com/23315232/135609786-8664ade8-a286-4e42-b4a4-ec24d701dbeb.png">

### Setup a Virtual Private Cloud on AWS
- Create a VPC 
- Create subnets (public and private subnets) as shown in the architecture. The subnets Ips are CIDR IPs. We can use utility sites like IPinfo.io to see the range of IP addresses associated with each subnet.

<img width="772" alt="Public-subnet-in-two-AZs" src="https://user-images.githubusercontent.com/23315232/135610761-69a92e4b-af61-47f8-90fd-bc47c1928083.png">

- Create private and public route tables and associate it with with the private and public subnets respectively

<img width="938" alt="edit_route_in_public_routetable_to_allow_subnets_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612563-2b40aed3-5308-4d0a-94bc-a4d8f1dfa33d.png">

- Edit a route in public route table, and associate it with the Internet Gateway. This allows the public subnet to access the internet

<img width="938" alt="edit_route_in_public_routetable_to_allow_subnets_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612563-2b40aed3-5308-4d0a-94bc-a4d8f1dfa33d.png">

- Create a NAT gateway and assign an elastic IP to it. You can use a NAT gateway so that instances in a private subnet can connect to services outside your VPC but external services cannot initiate  a connection with those instances.

<img width="937" alt="edit_route_in_private_routetable_to_allow_nat_Gateway_access_the_internet" src="https://user-images.githubusercontent.com/23315232/135612545-4c9cf9c3-deb2-4d4e-9fb9-cb38feb6c9d6.png">

- Create security groups for:
  -  Nginx servers: To allow access to from external application load balancer to the Nginx server
  -  Bastion servers: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. 
  -  External Load Balancer: The external application load balancer will be accessible from the internet
  -  Internal load balancer: The internal load balancer will allow https and http access from the  Nginx server and SSH access from the bastion server
  -  Web servers: The webservers will allow https and http access from the internal load balancer and SSH access from the bastion server
  -  Data layer security group: The access to the data layer for the appliation (consisting of both the Amazon RDS storage and Amazon EFS as shown in the architecture), will consist of webserver access to the RDS storage and both webserver and Nginx access to the Amazon EFS file system.
   
  <img width="900" alt="security_grp_rule_for_bastion_host" src="https://user-images.githubusercontent.com/23315232/135623670-bbbd20e3-944e-45e9-9054-05beef0349d2.png">
  
  <img width="904" alt="security_grp_rule_for_External_ALB" src="https://user-images.githubusercontent.com/23315232/135623681-1b0f9961-f807-4d6b-85d5-9b7a069ed7ff.png">

### Create a SSL/TLS certificate using Amazon Certificate Manager (ACM) to be used by the external and internal Application Load balancers (ALB)
- Create a wild card SSL/TLS certificate to be used when creating the external ALB. We want to ensure connection to the external ALB is secured and data sent over the internet is encrypted. Since the external ALB will listen for client requests to both the tooling webserver and the wordpress server, we'll create a wild card TLS certificate. Select DNS validation 

<img width="927" alt="creating_public_wild-card-TLS_certificate_for_ALBs" src="https://user-images.githubusercontent.com/23315232/135927685-23d76672-0d20-4f98-8f00-2466294515cd.png">

### Create Amazon EFS
- Create Amazon Elastic file System (EFS) to be used by the web servers for files storage. The mount targets to be specified fot the elastic file system will be the subnets for the webservers. Specifying the mount targets makes the EFS storage available to the webservers

<img width="729" alt="creating-Amazon-EFS-for-the-wordpress-and-tooling-servers-to-access" src="https://user-images.githubusercontent.com/23315232/136053493-6a69cb91-1a77-42d1-8af7-fbbaa7d4e7bb.png">

- Also, we specify access points on the EFS we created for the web servers. Amazon EFS access points are application-specific entry points into a shared file system. In this project, we create two access points on the EFS one for each web servers, each with its own root directory path specified.  Set the POSIX user and user group ID to root user and the root directory path to ```/wordpress``` and `/tooling` respectively.
- The root directory creation permission is set to `0755` to allow read write permissions on the file system by the clients 


<img width="726" alt="tooling-access-point-created on-EFS-for-the-tooling-webserver" src="https://user-images.githubusercontent.com/23315232/136055726-9aea4cd5-2fe9-4001-aa93-39d2ec6cf6ea.png">

### Create KMS key to be used for RDS
- Next, navigate to AWS KMS page to create a cryptographic key to be used to secure the MySQL relational database for the project.
- Create a symmetric key
- Set the admin user for the key. You can leave the 'key usage permission' with the default settings

<img width="663" alt="creating-symmetric-key-for-encrypting-and-decrypting-the-DB" src="https://user-images.githubusercontent.com/23315232/136060813-6fcb5b53-6283-49ce-b20f-e7815bbfdac2.png">


<img width="916" alt="kms-key" src="https://user-images.githubusercontent.com/23315232/136060837-c38d6b07-5311-4ff2-aac0-e0e2677f0403.png">




