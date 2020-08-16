# Assignment_P7_15Aug


## Deliverable

- Solution: 
	- Git repository of a working Cloudformation stack(s) of a:
	- AWS Client VPN Deployment backed by AWS Managed AD hosted in a new VPC following VPC best practises.
	
- Components would include:
	VPC
	Security Groups
	AWS Client VPN
	ACM Certificate
	AWS Managed AD
	EC2 instance to test connected VPN connectivity (this can be manually created in console)

## Manual Process

### 1. Create ACM public certificate:

To create a Client VPN endpoint, you must provision a server certificate in AWS Certificate Manager, regardless of the type of authentication you use. 

- Ideally you will use a registered domain to create certificate. 
- However, in case if you do not have any registered domain, you need to create a Self-signed certificate and upload it to ACM
	
	- Certificate Generation
		```
		$ git clone https://github.com/OpenVPN/easy-rsa.git
		$ cd easy-rsa/easyrsa3
		```

		- initialize a new PKI environment:

			$ ./easyrsa init-pki

		- build a new certification authority (CA):

			$ ./easyrsa build-ca nopass 

		- generate the server-side certificate and key:

			$ ./easyrsa build-server-full server nopass

				Using SSL: openssl LibreSSL 2.8.3
				Generating a 2048 bit RSA private key
				................+++
				...................................+++
				writing new private key to '/Users/anaudiya/personal/Others/Interview/PolarSeven/Assignment_15 Aug/workspace/easy-rsa/easyrsa3/pki/easy-rsa-93648.2tFsRn/tmp.iVL691'
				-----
				Using configuration from /Users/anaudiya/personal/Others/Interview/PolarSeven/Assignment_15 Aug/workspace/easy-rsa/easyrsa3/pki/easy-rsa-93648.2tFsRn/tmp.nuzabr
				Check that the request matches the signature
				Signature ok
				The Subject's Distinguished Name is as follows
				commonName            :ASN.1 12:'server'
				Certificate is to be certified until Nov 18 16:27:24 2022 GMT (825 days)

				Write out database with 1 new entries
				Data Base Updated

		- generate the client-side certificate and key: _(WILL BE PERFORMED FOR EACH USER: a_naudiyal, r_kumar & p_patel)_
			
			```
			$ ./easyrsa build-client-full a_naudiyal.adu.directory.com nopass			
			$ ./easyrsa build-client-full r_kumar.adu.directory.com nopass
			$ ./easyrsa build-client-full p_patel.adu.directory.com nopass
			```

				.....
				The Subject's Distinguished Name is as follows
				commonName            :ASN.1 12:'a_naudiyal.adu.directory.com'
				commonName            :ASN.1 12:'r_kumar.adu.directory.com'
				commonName            :ASN.1 12:'p_patel.adu.directory.com'
				Certificate is to be certified until Nov 18 16:30:09 2022 GMT (825 days)
				.....

	- Copy the server certificate and key & client certificate(s) and Key to a custom folder, for easy usage for next command.

			$ mkdir custom_folder/
			$ cp pki/ca.crt custom_folder/
			$ cp pki/issued/server.crt custom_folder/
			$ cp pki/private/server.key custom_folder/
			
			$ cp pki/issued/p_patel.adu.directory.com.crt custom_folder/
			$ cp pki/issued/r_kumar.adu.directory.com.crt custom_folder/
			$ cp pki/issued/a_naudiyal.adu.directory.com.crt custom_folder/

			$ cp pki/private/a_naudiyal.adu.directory.com.key custom_folder/
			$ cp pki/private/r_kumar.adu.directory.com.key custom_folder/
			$ cp pki/private/p_patel.adu.directory.com.key custom_folder/

			$ cd custom_folder/			

	- Upload the server certificate & key and the client certificate & key to ACM.

		```
		$ aws acm import-certificate --certificate fileb://server.crt --private-key fileb://server.key --certificate-chain fileb://ca.crt --region us-east-1

		$ aws acm import-certificate --certificate fileb://a_naudiyal.adu.directory.com.crt --private-key fileb://a_naudiyal.adu.directory.com.key --certificate-chain fileb://ca.crt --region us-east-1
		$ aws acm import-certificate --certificate fileb://r_kumar.adu.directory.com.crt --private-key fileb://r_kumar.adu.directory.com.key --certificate-chain fileb://ca.crt --region us-east-1
		$ aws acm import-certificate --certificate fileb://p_patel.adu.directory.com.crt --private-key fileb://p_patel.adu.directory.com.key --certificate-chain fileb://ca.crt --region us-east-1		
		```


### 2. Create AWS managed Microsoft Active Directory:

For this requierment, we will create one AWS Managed AD, with following settings:

Directory type: Microsoft AD (Standard edition)  
Directory DNS name: adu.directory.com  
	_(## FQDN what will resolve inside your VPC only. Does not need to be public resolvable.)_
Directory NetBIOS name: ADU  
Admin pass: ************  
	_(## Password for default administrator user named Admin)_

- VPC & Subnets : 
	Your directory adu.directory.com (d-90XXXXXXX9) is being created! **This can take up to 20-45 minutes.**
	Availability Zones: us-east-1a, us-east-1c (public/private subnets)

		DNS: 192.168.11.138, 192.168.1.183

	- 'Create' Application Access URL : adu.awsapps.com
		(## public endpoint URL where users in this directory can gain access to your AWS applications and to your AWS Management Console.)

	- Create users/Groups in MS-AD.

		- To create users/groups in an AD, we will use one Windows EC2 instance that has been joined to the AWS Directory Service directory, and be logged in as a user that has privileges to create users and groups. 
		- We will install the Active Directory Tools on this EC2 instance, using 'Active Directory Users and Computers' snap-in.

		- Process: 

			- **Creating DHCP options set**: 
				name: aws-ds-dhcp
				domain name: adu.directory.com
				domain name servers: 192.168.11.138, 192.168.1.183, 192.168.0.2

				dopt-045XXXXXXXXXd8d

			- Go to VPC : created for above Managed AD.
				Change its DHCP option set to created above

			- **Create IAM Role**: To join EC2 windows to domain, we will need SSMdirectoryService access permissions.

				Trust relationship: EC2
				Permission policy: AmazonSSMManagedInstanceCore, AmazonSSMDirectoryServiceAccess
				Role Name: EC2DomainJoin

			- **Create EC2 instance & automatically join the directory**:

				AMI: Microsoft Windows Server 2019 Base - ami
				type: t2.medium
				Network: Use same VPC where AD is created and choose public subnet to have public IP, so we can login. (AutoAssign public IP: true)
					Domain: adu.directory.com
					IAM Role: EC2DomainJoin

				create securityGroup: 
					name: ad-mgmt-instance-securityGroup
					3389 : my IP
				
				Public IP : 3.XXX.XXX.18  (AD managment instance)

			- **Install Active Directory Tools**

				- Verify instance is joined to AD by login using above Admin username/password created in MS-AD.

						username: adu.directory.com\Admin
						pass: *********

				- Login to the instance 
					> Start Menu > Server Manager
					> Add Roles & Features
						> Role-based or feature-based installation
						> Select destination server: Choose local server
						> Select server roles page > Next (default)
						> Select features
							> select Group policy management
							> Expand Remote Server Administration tools > Role Administration tools 
								> Select the 'AD DS and AD LDS Tools' check box. 
								> Select the 'DNS Server Tools' check box. 

					> Install

					- When the feature installation is finished, the following new tools or snap-ins will be available in the Windows Administrative Tools folder in the Start menu.

						    Active Directory Administrative Center
						    Active Directory Domains and Trusts
						    Active Directory Module for Windows PowerShell
						    Active Directory Sites and Services
						    Active Directory Users and Computers
						    ADSI Edit
						    DNS
						    Group Policy Management

				- Install AD tools

					- Start windows power shell, and type:
						
						Install-WindowsFeature -Name GPMC,RSAT-AD-PowerShell,RSAT-AD-AdminCenter,RSAT-ADDS-Tools,RSAT-DNS-Server

			- **Create Users**:

					- Login with Domain Admin:
							user: adu.directory.com\Admin
							pass: m1cr0s0FtLog1n

					- Start > Windows Administrative Tools > Active Directory Users & Computers

						adu.directory.com > adu > Users

							First/Last/UserLogon/pass: Amit / Naudiyal / a_naudiyal / Redhat123#    @adu.directory.com
							First/Last/UserLogon/pass: Ramesh / Kumar / r_kumar / Redhat123#		@adu.directory.com
							First/Last/UserLogon/pass: Parikshit / Patel / p_patel / Redhat123#  	@adu.directory.com

						adu.directory.com > adu > Groups

							Group name: Client VPN
							Group scope: Global
							Group type: Security

								Add a_naudiyal & r_kumar users into it.
								Keep p_patel out of it.


### 3. Create Client VPN Endpoint: https://console.aws.amazon.com/vpc/home?region=us-east-1#ClientVPNEndpoints:sort=clientVpnEndpointId

			Name: vpn-endpoint-amitInfraVPC-public
			Description: vpn-endpoint-amitInfraVPC-public
			Client IPv4 CIDR: 172.16.0.0/22
				(## Subnet range, from which Client IP address will be allocated. Must be different than the IP of the resources which will be connected via VPN)

			Authentication:
				Server Certificate ARN: arn:aws:acm:us-east-1:XXXXXXXXXXXX:certificate/6fb9fd21-5457-4c54-91b8-db9d028648e0
						(## Imported Server certificate)

				Authentication Option: Use user-based authentication > Active Directory authentication

						Directory ID: d-90XXXXXXX9  (from above)

			Connection logging: 
					Log Group: /adu.directory.com-logs/
					Stream: first-stream

			Other
				DNS Server 1 Ip address: 192.168.11.138  (from above)
				DNS Server 2 Ip address: 192.168.1.183   (from above)

				Transport: udp
				VPC ID:
					SecurityGroup: sg-03fea951c537fbcb3 (vpn-endpoint-security-group)
				VPN Port: 443

		> Once you create VPN endpoint, it will be in 'Pending-associate' status. 
			- This means we can now associate the VPN endpoint with one or more VPCs.

		- Associate Client VPN endpoint to a Target Network:
			- We choose a VPC and subnet to create the association with our Client VPN endpoint. 
			- you can associate client VPC endpoint to multiple subnets, provide it belongs to the same VPC and in a different AZ.

			- Create VPN Association to Target network:
				VPC: vpc-b4255cd2
				Subnet: subnet-d288c2b7  (amitInfra-public3)
					(## Association ID : cvpn-assoc-0d711a1045d6c2d7b)

				Subnet: subnet-301beb78  (amitInfra-pvt2)	
					(## Association ID : cvpn-assoc-03f7d4db5ef564684)				


		- Enable end-user access to VPC (Add authorization rule)
			Authorization rule controls which set of users can access to specified network through Client VPN endpoint.

			- Get the SID of the 'Client VPN' AD group that was created earlier.

				PS C:\Users\Admin> Get-adgroup -identity "Client VPN"

				DistinguishedName : CN=Client VPN,OU=Users,OU=adu,DC=adu,DC=directory,DC=com
				GroupCategory     : Security
				GroupScope        : Global
				Name              : Client VPN
				ObjectClass       : group
				ObjectGUID        : 85c51f1a-08fd-45fd-8506-87d468116002
				SamAccountName    : Client VPN
				SID               : S-1-5-21-3319784565-47933065-3989491047-2113  		<<=====

			- Client VPN Endpoints > Select your VPN endpoint > Authorization > Authorize Ingress

				Client VPN endpoint ID: cvpn-endpoint-095af903f2f1327c9
				Destination network to enable: 0.0.0.0/0   (IP address/range which can access this endpoint. You can restrict it to specific IP range/address)
				Grant access to: 
					Allow access to users in a specified access group

						Access Group ID: S-1-5-21-3319784565-47933065-3989491047-2113

				Description: Client VPN AD Group

				Note: Now users belonging to 'Client VPN' AD group are authorized to route all traffic through the VPN client endpoint.

		- Applying Security Group:

			These are used to limit access to applications. This securityGroup only controls the traffic egress from VPC associated ENIs.
			To limit traffic that can route through VPC associated ENIs, restrictive authorizations can be used.

			- WE will already have one securityGroup: sg-03fea951c537fbcb3 added there, which we selected earlier.
			- WE can leverage the securityGroup we have applied to our VPN endpoint, as the source for traffic in other securityGroups.

		- Add Routes:

			Routes for associated VPC/Subnets are automatically added to the Client VPN Route table.

			- If you want this VPC association to provide Internet connectivity to VPN clients (through NATGW / IGW), you need to add a default route of 0.0.0.0/0 to the route table. 

					Route destination: 0.0.0.0/0
					Target VPC subnet ID: subnet-d288c2b7
					Description: Internet access for vpn clients


				Destination CIDR 			TargetSubnet			Type 		Origin

				10.0.0.0/16 				subnet-d288c2b7 		Nat 		associate
				0.0.0.0/0 				    subnet-d288c2b7 		Nat 		add-route
				10.0.0.0/16 				subnet-301beb78 		Nat 		associate

			- If you now notice the Network interfaces in your account with Description "ClientVPN Endpoint resource", you will notice 3 different ENIs for each of above Routes, with public and private IP address.

					eni-014f5c761311d6786 : subnet-301beb78 : 3.234.159.254 : 10.0.12.104
					eni-03dd5c6c09067b0e5 : subnet-d288c2b7 : 3.215.53.31 : 10.0.3.100
					eni-07b306df5667e10bf : subnet-d288c2b7 : 54.161.153.42 : 10.0.3.53

				Note: If your associated VPC have access to on-Prem resources, you can add a route your on-Prem network (100.200)

	- Download Client Configuration

			downloaded-client-config.ovpn


### 4. Installed AWS VPN Client: https://aws.amazon.com/vpn/client-vpn-download/

	- Created a Profile:

			Name: vpn-endpoint-amitInfraVPC-public
			VPN file: downloaded-client-config.ovpn

	- Connect:

			Username: a_naudiyal
			password: Redhat123#

		CONNECTED

		Client IP address allocated to my mac: 

			utun2: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1500
			inet 172.16.0.162 --> 172.16.0.162 netmask 0xffffffe0 

