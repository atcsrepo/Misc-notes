### Basic AWS deployment

This section largely pertains to deploying a single Dockerized Node.js application to an AWS EC2 instance, which is relatively straight forward.

To start, make sure the security credentials for the IAM account being used is available and that the AWS Command-line Interface is installed.

#### Get the AWS-CLI

AWS CLI installation instructions for Windows can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/install-windows.html).

After installing the AWS-CLI, configure it in command prompt with:

`aws configure`

It will ask for security credentials, region, and output format. The output format option can be ignored, which will default to JSON. For more detailed instructions, visit AWS&#39; [guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).

#### Set up a repository

With the CLI configured, images can now be pushed into the ECS repository. To set one up, go to Services > ECS > Repository > Create repository. Once a repository has been created, select it and click on "View push commands". This will provide a 4 step guide to push an image into the repo. Just copy and paste the instructions into the command prompt. Note that if working with the Docker tool box, use the linux instructions.

----

### Setting up an EC2 instance
For a single container, setting up an instance is fairly straight forward, and will require 3 major steps - task set-up, service set-up, and cluster launch. When the cluster is launched, then the instance will be set-up and start running.

#### Task set-up:
The [task definition](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecs-taskdefinition.html) largely involves defining what images, volumes and resources will be required, and how everything will interact with each other.

To set one up for a single docker container, go to ECS > Task Definitions > Create new task. Select an EC2 instance (Fargate is not free tier eligible).  For a basic set-up, provide a name for the task, then skip down to "Task size" and, under "Container Definitions&#34;, select "Add container&#34;. 

Provide a name for the container and a link to the image repository. In the memory limits section, either a hard and/or soft limit for the task needs to be specified. The hard limit defines the maximum amount of memory a container can take up before it gets killed by AWS. In comparison, a soft limit allows for bursts past the limit, up to the point of a hard or instance limit. Additional detail can be found [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html). 

For port mapping, it will likely be the host&#39;s port 80 (http) mapped to a user-defined port for the docker container (8000 in our case).  After setting up the port mapping, just skip to the end and add the task.

#### Launching a cluster
We can now set up an instance for the task. Go to ECS > Clusters > Create cluster > EC2 Linux + Networking. Provide a name for the cluster. Select t2.micro as the instance type and initiate only 1 instance to stay within the AWS free tier limit - this assumes that the instance will be running for the entirety of the month. Enable SSH (create a key pair if needed via Services > EC2 > Security & Networks > Key Pairs). Note that this is the only time SSH can be enabled. After confirming the details, create the cluster, which will launch the instance.

#### Adding a service for the task
AWS has a section on creating services [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-service.html), which defines how your service behaves.

Go click on the newly created cluster. Click on the services tab and create a new service. Select "EC2" for launch type. Provide a name for the service and enter "1" for the number of tasks. Because there is only 1 task, for minimum and maximum health, enter "0" and "100" respectively. Elastic Load Balancing can be enabled if desired, otherwise ignore it, along with everything else up to the point of service creation.

After creating the service, the task should start running shortly. At that point, it should be possible to visit the application at either the instance address or, if there is a [domain name](https://github.com/atcsrepo/Misc-notes/blob/master/Domain-name.md) associated with it, a domain name address.

To find the instance address, go to Service > EC2 > Instances, and select the instance. It should list the Public DNS or an IP address that can be used to access the live instance.

----

### Updating a container
At some point, there may be a need to update the deployed container. Just push the new image into the repo. Afterwards, go to services and select update for the running service. If nothing other than the container has changed, select "Force  new deployment" and just skip everything until the end.

----

### Getting a static IP
Acquiring a static IP might be a good idea because the default instance IP assigned by AWS can change if the instance is stopped. This could disrupt things like DNS registration, where an IP is specified. To acquire an IP (or IPs) for the account, go to EC2 > Network & Security > Elastic IPs.

Allocate a new IP address and associate it with the running instance, which will update the IP address. Additional information from AWS can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html).

A point of note - if an Elastic IP is NOT associated with an active/running instance, then the user will accrue [pro rata charges](https://aws.amazon.com/premiumsupport/knowledge-center/elastic-ip-charges/) at a rate of a [few dollars per month](https://aws.amazon.com/ec2/pricing/on-demand/#Elastic_IP_Addresses) per unused IP address. As such, an Elastic IP should be released if there is no longer any use for it.
