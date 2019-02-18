# Launching a Multi-Container Application on an EC2 Instance

This piece will largely serve as an additional supplement to the [AWS guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-ec2.html) for launching a multi-container application on an EC2 instance. While it is possible to combine all container services into one container using supervisord or runit, it is [not recommended by Docker](https://docs.docker.com/config/containers/multi-service_container/). As such, this calls for using docker-compose or an equivalent.

There are two genral approaches to launching a multi-container application on an EC2 instance - either use AWS&#39; docker-compose solution or SSH into the EC2 instance and install Docker compose. The only problem with the latter solution is that the application can not really be associated with an AWS task/service, if that can be considered a problem. On the other hand, going the AWS route places some limitations on what can and cannot be done. That being said, we will focus on the AWS route first. 

---
### Setting up AWS tools

For the AWS route, the [ECS-CLI](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html), and maybe AWS-CLI, will need to be installed. The instructions for configuring the ECS-CLI can be found [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_Configuration.html), and largely boils down to two steps:

1) Set-up credentials:

`ecs-cli configure profile --profile-name profile_name --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY`

2) Define cluster configurations:

`ecs-cli configure --cluster cluster_name --default-launch-type launch_type --region region_name --config-name config_name`

With this, we can now launch an EC2 instance using:

`ecs-cli up --keypair id_rsa --capability-iam --size 1 --instance-type t2.micro --cluster-config config_name`

Additional paramters can be found [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-up.html), but the above can be used to launch a single t2.micro cluster with id_rsa used as the keypair for SSH&#39;ing into the instance. The name of the keypair should be found in `EC2 > Network & Security > Key Pairs`. With this, an instance should now be up and running and ready for application deployment. Note that the above can be performed in the AWS console as well.

---
### Deployment through ECS-CLI

To deploy a multi-container application, a Docker compose YAML file will be required. When creating the YAML file, note that there are a number of commands unavailable when deploying with ECS-CLI, which is disclosed in the [AWS ECS-CLI tutorial](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-ec2.html). Most notably, "build" is missing. Likewise, if using the Docker compose version 3 format, "network" is also missing, so users will have to default to using links.

It will be apparent if a command is not supported, as warnings will be everywhere:

```
←[33mWARN←[0m[0000] Skipping unsupported YAML option for service...  ←[33moption name←[0m=networks ←[33mservice name←[0m=app
←[33mWARN←[0m[0000] Skipping unsupported YAML option for service...  ←[33moption name←[0m=restart ←[33mservice name←[0m=app
←[33mWARN←[0m[0000] Skipping unsupported YAML option for service...  ←[33moption name←[0m=depends_on ←[33mservice name←[0m=nginx
```

Quite hard to miss.

There are also two other points worth noting. If using persistent volumes (e.g. for storing SSL certificates), it may be better to bind a volume than to mount a named volume (e.g. provide a source path as opposed to name). The problem is that each time a different YAML file is tested (e.g. if an update is made), the ECS-CLI creates a new task. As a result of the task name changing, the volume name within the instance also changes. This leads to the previously mounted volume being left unused. This is largely circumvented by binding volumes. 

Note that when binding volumes, the provided path resolves to a file or directory in the remote location and not locally. Additionally, it is possible to try writing to EBS, EFS, or some other storage system, as outlined in this [SO answer](https://stackoverflow.com/questions/45788214/docker-volumes-in-aws-write-to-ebs-efs-s3). This will allow the data to persist beyond the instance.

The second point, which may be easier to miss, is that when starting up a cluster of containers on AWS, all containers, by default are considered essential. As a result, if one container terminates early, everything comes to a halt and all containers exit. This can be seen in the followng abridged exerpt:

```
2019-02-04T09:29:41.734383817Z 2019-02-04T09:29:41Z [INFO] Task [arn:aws:ecs:us-east-2:298398473745:task/8b067047-b641-460c-94ee-84c0d95d651a]: recording execution stopped time. Essential container [certbot2] stopped at: 2019-02-04 09:29:41.730416685 +0000 UTC m=+26298.873802262
2019-02-04T09:29:41.734402095Z 2019-02-04T09:29:41Z [INFO] Managed task [arn:aws:ecs:us-east-2:298398473745:task/8b067047-b641-460c-94ee-84c0d95d651a]: sending container change event [certbot2]: 
2019-02-04T09:29:41.734413047Z 2019-02-04T09:29:41Z [INFO] Task engine [arn:aws:ecs:us-east-2:298398473745:task/8b067047-b641-460c-94ee-84c0d95d651a]: stopping container [app]
2019-02-04T09:29:41.741458274Z 2019-02-04T09:29:41Z [INFO] Task engine [arn:aws:ecs:us-east-2:298398473745:task/8b067047-b641-460c-94ee-84c0d95d651a]: stopping container [revproxy]
```

Everything gets exited and removed after the first container stops. Lessons learnt? Read the log files - it may not be a problem with the containers per se, but an AWS feature.

Aside from the above, everything else is fairly standard. Because "build" is not supported, users will need to build images locally and push to a repo using either AWS-CLI, GitHub, or some other alternative. The repository link can then be used for the image.

If using the Docker compose version 3 format for the YAML file, AWS will also require users to include an [ecs-params.yml file](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-ecsparams.html) which will specify arguments for memory limits and cpu usage limitations that would typically be provided when setting up tasks.

When the YAML file is ready, the task can be launched via:

`ecs-cli compose up --create-log-groups --cluster-config config_name`

A quick checking using:

`ecs-cli ps` or `ecs-cli ps --cluster-config config_name`

should let you know if everything looks fine. If needed, it is always possible to SSH into the instance or visit the instance&#39;s public address. If everything looks fine, remember to take down the task using 

`ecs-cli compose --file hello-world.yml down --cluster-config config_name`

before initiating a service with:

`ecs-cli compose --file docker-compose.yml scale 1 --cluster-config config_name`

Additional service setting parameters can be found [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cmd-ecs-cli-compose-service-up.html).

#### Is the ECS-CLI necessary?
No. The "ecs-cli compose up" command fills out the task definitions, which can be seen on the AWS console under `ECS > Task Definitions`. That being said, it is quicker than filling it out by hand. In comparison, setting up a service on the AWS console is super quick. For one or two services, it may be quicker to just fill it out on the console itself.

----
### General steps to launching an application without using ECS-CLI
If, for some reason, using ECS-CLI is not possible, then everything can be done within the instance itself. To start, launch an instance by going through `Services > EC2 > Instances > Launch instance`. Alternatively, if ECS-CLI is already installed, an instance can be launched as descibed above. 

After launching an instance, [SSH into the instance](https://github.com/atcsrepo/Misc-notes/blob/master/EC2-SSH.md) and check that:
1. There is access to a repo (Git, AWS, etc). If not then set one up.
	1. Instructions for installing pip and the AWS CLI can be found [here](https://docs.aws.amazon.com/cli/latest/userguide/install-linux.html).
	1. If using Git, use `sudo yum update` and `sudo yum install git`
1. Confirm that Docker is available (it should be).
1. Install Docker compose.
	1. Follow the instructions [here](https://docs.docker.com/compose/install/), which comes down to two commands:
		1. `sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
		1. `sudo chmod +x /usr/local/bin/docker-compose`

If using the AWS repo, the IAM settings may need to be re-configured if the following error shows up:

```
An error occurred (AccessDeniedException) when calling the DescribeRepositories operation: User: arn:aws:sts::298398473745:assumed-role/amazon-ecs-cli-setup-portfolio-EcsInstanceRole-CDO71NCWWT8H/i-0102760df7837c852 is not authorized to perform: ecr:DescribeRepositories on resource: *
```

To do so, under an Admin account on the AWS console, go to `IAM > Roles`. Select the role in question and attach the required ECR policy to provide access to the repo. Otherwise, just pull in the YAML file, and any other required files, and use docker-compose up.

#### Mimicking AWS tasks and services
While it will not be exactly the same, it is possible to mimic some of the functionalities from AWS task definitions and services; however, this will depend to some extent on whether Docker compose version 2 or 3 format is being used.

For example, it is possible to declare reserved memory and memory limits in version 2, if using Docker compose; however, if using version 3, memory restrictions are ignored by Docker compose and are only applicable to swarms. The same applies to other paramters, such as restart requirements and cpu limits. Being that a single node application is being deployed, if there is a need to set limits, then version 2 format may be a better option.
