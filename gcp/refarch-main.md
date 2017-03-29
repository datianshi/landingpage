#Pivotal Customer0 _PCF on GCP_ Reference Architecture

*__Goal__*:  Customer0 Reference Architectures are utilized by Pivotal's Customer0 group to _simulate_ a base deployment of our products that is common to as many customer use cases as possible.  These architectures are then automated via concourse pipelines and _'validated'_ thru various customer0 validation scenarios to simulate typical customer use cases.

######* Customer0[Typical Customer] * = _A secured but publicly accessible PCF Foundation, capable of hosting ~100 Application instances with PCF managed Services: "Mysql, RabbitMQ, Pivotal Spring Cloud Services"_

*__Non-Goals__*:

- This PCF on GCP reference architecture is published as is with no warranty or support expressed or implied!
- This document is NOT intended to replace the basic installation documentation located @ [http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp.html](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp.html), but rather to demonstrate how those instructions should be related to a typical/recommended Pivotal Cloud Foundry Installation on GCP.

*__Validation Key Info__*: (STATUS=Validation In Progress)

| PCF Products Validated        | Version                  | Known Issues              |
| -----------------------------:|:-------------------------|:-------------------------|
| PCF Ops Manager               | 1.8.10.0 (Nov 2 2016)    | PASS |
| Elastic Runtime               | 1.8.13 (Nov 6 2016       | SmokeTests=PASS; CATs=6 Failures |
| Rabbit                        | 1.7.6 (Nov 2 2016)       | PASS |
| Metrics                       | 1.1.# (Latest)           | NOT YET VALIDATED |
| Mysql                         | 1.7.# (Latest)           | NOT YET VALIDATED |
| Spring Cloud Services         | 1.2.# (Latest)           | NOT YET VALIDATED |

## Reference Architecture IaaS Overview

PCF on GCP Reference Architecture Overview (*c0-gcp-base*):![c0-gcp-base v1.0.1](../static/gcp/images/PCF-GCP-RefArch-Overview/overview-arch.png)

- [Pipeline Repo Link](https://github.com/pivotal-cf/gcp-concourse) : Customer0 Concourse Pipelines


###__PCF on GCP requires the following GCP Components__:

##### Project/Region/Zones

A Cloud Foundry Deployment will exist within a single Project.  It will be located in a single GCP region and should distribute Cloud Foundry Jobs/Instances across 3 GCP zones to ensure a high degree of availability.  Each GCP Zone will map to a Cloud Foundry Availability Zone

##### Google Cloud API

Pivotal Ops Manager & BOSH will utilize the Google Compute Engine API, it must be enabled on a new Google Cloud subscription as it is not enabled by default


#####Quotas

Default quotas on a new GCP subscription will not have enough quota for a typical PCF deployment.  This Reference Architecture has been sized to host ~100 typical Cloud Foundry AIs.   You should request a Quota increase for the following Objects:

| Resource        | Suggested Min Limit                   |
| ----------------------------- |:-------------------------:|
| CPUs *Region Specific*					|150|
| Firewall Rules				   				|15|
| Forwarding Rules				   			|5|
| Global Forwarding Rules					|5|
| Global Backend Services					|5|
| Health Checks				   				|10|
| Images				   						|10|
| Static IP Addresses *Region Specific* & Assuming SNAT topology|5|
| IP Addresses Global				   		|5|
| IP Addresses *Region Specific*	& Assuming SNAT topology|5|
| Networks 									|5|
| Subnetworks									|5|
| Routes										|20|
| Target Pools								|10|
| Target HTTP Proxies Global				|5|
| Target HTTPS Proxies Global				|5|
| Total persistent disk reserved (GB) *Region Specific* | 15,000 |

#####Service Accounts

Best practice PCF on GCP deployments requires 2 "Service Accounts"

1. Operator Account -> "For Terraforming" 

   - Prior to deploying PCF on GCP, you will need a GCP account with proper permissions to create GCP objects via terraform.  Minimum permissions for this account can be found [here](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp.html#gcp).

   
2. IAM Service Account -> "For OpsMan/BOSH"


   - Ops Manager will require you to select 1 of 2 approaches for a service account for BOSH & bosh-init to use for deployments.

		- `AuthJSON`

			This option requires that the operator create an IAM svc account with appropriate permissions documented [here](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp-prepare-env.html#iam_account).  This is the preferred method of Customer0 Engineers & Reference architecture.

		- `The Ops Manager VM Service Account`

			This option polls the GCP api to get the service account `email|id` of the service account & scopes that were assigned when the OpsManager VM was created and leverages that account.  This method requires the service account have the _Project > Service Account Actor_ role

		*Note*: C0 pipeline does not currently utilize 2 service accounts.  This will be modified soon to allow the pipeline to be fully portable and simulate a customer deployment with limited creds on service accounts.
		
#####Networks

Review Pipeline Network objects here: [C0 GCP Pipeline Terraform Network Objects](https://github.com/pivotal-cf/gcp-concourse/blob/master/terraform/networks.tf)

GCP Network objects allow multiple subnets with multiple CIDRs , so a typical deployment of PCF will likely only ever require 1 GCP Network object as well as 1 or more of the following:

- **Subnets** 

	A minimum of 3 subnets are recommended:
	
	1. *"infrastructure"* (/26) <->  This network will host: 
		- _["OpsManager","Director(aka BOSH)","Jumpbox"]_
	2. *"ert"* (/22) <-> This network will host the core instances of cloud foundry:
	   - _["GoRouters","Diego Cells","Cloud Controllers", "etc..."]_
	3. *"services-#"* (/22) <->  This network ,as well as additional service networks, will host PCF managed service tiles:
		- _["Rabbit","Mysql","Spring Cloud Services", "etc..."]_

- **Routes**

	Routes are typically created by GCP dynamically when subnets are created, but C0 terraform scripts create additional routes to force outbound communication to dedicated SNAT nodes.  These objects are required to deploy PCF without public IP addresses.  (see Network Topology image)

- **External IPs**

	

#####FireWall Rules

Review Pipeline Rules here:[C0 GCP Pipeline Terraform FW Rules](https://raw.githubusercontent.com/pivotal-cf/gcp-concourse/master/terraform/c0-gcp-base/3_firewalls.tf)

![rules_and_tags v1.0.0](../static/gcp/images/PCF-GCP-RefArch-Overview/firewall-rules.png)

GCP Firewall rules are bound to a Network object and can be created to use any of the following objects as a match for source & destination fields in a rule:

- *[IP range/Any]*, basically a specific cidr or 0.0.0.0/0 in the case of any
- *[Subnetworks]*, selecting a subnet that a GCP instance is attached to will apply as a match
- *[Instance Tags]*, All PCF jobs get a default_deployment tag as well as additional tags, like job specific tag ["router", "diego-brain", "cell", "etc..."]

Instance tags are the preferred method of applying Firewall rules/ACLs in the Customer0 Reference Architecture. In the image above, irrespective of the subnet ranges applied, traffic can be controlled with the use of tags. Traffic that does not match the source & destination of a rule that is explicitly 'allowed' will be dropped. For example, in the image above, ssh attempts to connect to TCP:22 routed to the 'Diego Brain' job will fail as only TCP:2222 has been allowed via a rule and a matching tag. Since the 'Diego Brain' has no tag matching an 'allow' rule for TCP:22, traffic  will be dropped.


#####Load Balancing

PCF on GCP requires multiple load balancing services types.  The first type is the **_"HTTP"_** load balancer which is leveraged for API/Application access to Cloud Foundry.  It is a very scalable and globally capable resource which does support SSL termination as well as required header sets to forward HTTP/HTTPS traffic direct to the Cloud Foundry  'GoRouter' jobs.   This is why 'ha_proxy' jobs are not necessary in a GCP deployment.  Currently, **HTTP load balancers DO NOT SUPPORT WEBSOCKETS!!!**, this is important in that doppler/loggregator endpoints require websockets, so improperly configure load banacing topologies will not allow 'cf logs' to function. The GCP refarch splits log traffic to another GCP router type (TCP) as shown in the "Network Topology" diagram in this refarch documentation.  Additionally, this is why the GoRouters perform SSL termination as well as the HTTP load balancers, so that log traffic can be encrypted.

The second key type of GCP load balancer PCF will us is the **_"TCP"_** variant.  It cannot perform advanced URL functions, nor can it terminate SSL,  but is a very capable TCP load balancer for Cloud Foundry to leverage as proxy services for:

1. Websockets based 'cf logs' traffic (Required)
2. Diego-Brain ssh-proxy services for 'cf ssh' traffic (Optional)
3. Cloud Foundry TCP Load Balancing Services (Optional)

GCP load balancing services are comprised of multiple components you will see deployed as part of the Customer0 Terraform Scripts:

- Forwarding Rules:
  - These rules can be either 'Global' for use with HTTP load balancing or local for use with TCP load balancing.
  - 'Global' forwarding rules (Used by HTTP LB) map a front end IP address & TCP port to a 'target proxy'.
  - Local forwarding rules (Used by TCP LB) map a front end IP address & TCP port to a 'target pool'

- Target Proxy(s) (Used by HTTP LB)
  - Maps a URL linked to the Front End IP to a 'Backend Service'

- Backend Service(s) (Used by HTTP LB)
  - Bind a 'Health Check' as well as an 'Instance Group' to accept forwarded traffic from the Global-Forwarding-Rule(s)->Target-Proxy(s)->Url_Map(s).  The 'Instance Group' is typically comprised of the Cloud Foundry 'GoRouter' jobs.

- Certificate(s) (Used by HTTP LB)
  - Certs for Client SSL termination when accessing the Cloud Foundry API or Cloud Foundry hosted applications.

- Target Pool(s) (Used by TCP LB)
  - Bind a 'Health Check' as well as 1 or more instances/jobs to accept traffic from a Local-Forwarding-Rule.  These can be various jobs depending on the Forwarding rule,  for example, the 'ssh-proxy' forward rule will map to a target pool consisting of 'Diego-Brain' jobs and a health check for TCP:2222.

- Health Checks
  - Define how HTTP & TCP load balancing services determine is a node is healthy or not to accept forwarded traffic.
 
#####Instance Groups
#####Images
#####Google Cloud Storage Bucket(s) & Token(s)

## Network Topology

PCF on GCP Base Network Topology (*c0-gcp-base*):![v1.0](../static/gcp/images/PCF-GCP-RefArch-Overview/net-topology-base.png)

Explain Base Network Architecture & GCP Objects

  - Recommended Firewall Topology with Tags
  - GCP Project SSH Topology
  - Application Security Groups
  - Load Balancer Topology
    -  TCP
    -  HTTPS

## Variants to Base Reference Architecture

Will Insert Variant Image(s) here:

Customer0 will 'validate' a limited number of variant scenarios from the base topology.

| Variant *{{gcp_pcf_terraform_template}}*| Varient Description                   |
| -----------------------------:|:-------------------------|
|c0-gcp-nonat|Base template (+) No dedicated SNAT, default GCP instances each with Public IPs for SNAT| 
|c0-gcp-private|Base template (-) All External IPs, no public IPs at all||
|c0-gcp-ipsec|Base template (+) IPSEC add on||
|c0-gcp-gslb|2 x Base templates deployments Globally load balanced||

  

##Pivotal Customer0 PCF on GCP Deployment Pipeline

Describe what Customer0 Uses the pipeline for (Solution Validation)
[Insert Link to pipeline repo]

Min-Reqs to run the Pipleine ...

1. Appropriate GCP Quotas
2. [GCP Service Account](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp-prepare-env.html#iam_account)
3. [Enable GCP APIs](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp-prepare-env.html#enable_compute_resource_api)
4. Google Cloud Storage Access Token
5. A resolvable/registered DNS domain for Cloud Foundry `system` & `apps` domains
6. A [Concourse](https://concourse.ci/) instance with workers that have public access.

(ToDo) Document How to use the pipeline in a POC scenario ...

(ToDo) Document How to ref the pipeline for manual deployment steps ...

(ToDo) Document What Customer0 Validates * (Future html link to validation repos) ...


##PCF on GCP Helpful Links

- [https://cloud.google.com/solutions/cloud-foundry-on-gcp](https://cloud.google.com/solutions/cloud-foundry-on-gcp)
- [http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp.html](http://docs.pivotal.io/pivotalcf/1-8/customizing/gcp.html)
- [https://github.com/cloudfoundry-incubator/bosh-google-cpi-release](https://github.com/cloudfoundry-incubator/bosh-google-cpi-release)
- [http://bosh.io/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent](http://bosh.io/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent)
- [http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-google-cpi-release?all=1](http://bosh.io/releases/github.com/cloudfoundry-incubator/bosh-google-cpi-release?all=1)
