
[TOC]



# 1.Objective
The aim of this article is to provide guidance to migrate an ASP.NET MVC 4.6 or older application in to AWS ECS using containers. This will also cover the step-by-step instructions, cloud formation template and ECS task definition

# 2.	Why ASP.NET MVC 4.6 and Windows Containers?
The ASP.NET MVC 4.6 and older versions of ASP.net occupy a significant footprint in the enterprise web application space. As enterprises  move towards microservices for new or existing applications, containers are one of the stepping stones for migrating from monolithic to microservices architectures. Additionally, support for Windows Containers in Windows 10, Windows Server 2016, and Visual Studio Tooling support for Docker, simplifies the containerization of ASP.NET MVC apps.

# 3.	Pre-requisites
Ensure your development environment has the following setup as per this [Microsoft article](https://docs.microsoft.com/en-us/aspnet/mvc/overview/deployment/docker-aspnetmvc):

a) Visual Studio 2017 with latest updates

b) Docker for windows – version stable 1.13.0 or 1.12 Beta 26 (or newer versions)

c) Windows 10 Anniversary Update (or higher) or Windows Server 2016 (or higher)

If the web application was developed in earlier version of Visual Studio, it needs to be opened in Visual Studio 2017 which migrates all the settings to IDE.


The following is the sample ASP.NET MVC 4.6 application that we’ll be migrating.
![](/screenshots/pic1.jpg)

When the application is run in the development environment, it renders the following output.

![](/screenshots/pic2.jpg)

# 4.	Containerization of ASP.NET MVC 4.6 application
These are the steps we will follow to containerize the ASP.NET MVC 4.6 application:
- Creation of Docker file
- Building a Docker image that will run ASP.NET MVC web app
- Run image locally
- Test in browser

## 4a.	Creation of Docker file
The Visual Studio 2017 support for Docker Tooling is leveraged to create Docker file.
Right click on Project and Add Docker Support

![](/screenshots/pic3.jpg)
 
This adds a Docker file to the current project and a Docker Compose project to the Solution.

The Docker file added by Visual Studio 2017 should look like this:

			
```
FROM microsoft/aspnet
ARG source
WORKDIR /inetpub/wwwroot
COPY ${source:-obj/Docker/publish} .

```

This file pulls the ASPNET Windows container image from the public DockerHub repository and pushes the binaries of the current project onto containers. This is enough for running locally in the development environment. To be able to run on Amazon ECS and for users to access the web service through Application Load Balancer (ALB), port 80 needs to be explicity exposed on the container. Update the Dockerfile to look like this:
```
FROM microsoft/aspnet
ARG source
WORKDIR /inetpub/wwwroot
COPY ${source:-obj/Docker/publish} .
EXPOSE 80

```
Right Click on the ASP.NET MVC Project -> Build

This completes compiling and building the ASP.NET MVC project.

## 4b.	Building a Docker image that will run ASP.NET MVC web app
The Docker Compose project in the Visual Studio Solution will be leveraged to build a Docker image that will run the ASP.NET MVC app. The Docker compose project added by Visual Studio 2017 should look like this.

 ![](/screenshots/pic4.jpg)

The Docker compose.yml definition added by Visual Studio 2017 should look like this.
version: '2.1'


```
==services:
  awsecssample:
    image: awsecssample
    build:
      context: .\AWSECSSample
      dockerfile: Dockerfile==

```

The default docker-compose.yml will be leveraged for building the container image. 

Right click on docker-compose project -> Build.

A container image is built as per the Docker file definition. It can be verified by invoking ‘Docker Images’ command in terminal.

![](/screenshots/pic5.jpg)

The step of building windows container image for ASP.NET 4.6 web application is complete.

## 4c.	Starting container that runs the image 
The ‘docker run’ command is executed from the terminal to start the container that runs the image.


```
docker run -d --name aspnetcontainer awsecssample
```

![](/screenshots/pic6.jpg)
 
## 4d.	Test in browser

When the http://localhost is opened in the browser, it will not render the expected ASP.NET MVC app. With the current windows container release, we can’t browse to http://localhost.  This is a known behavior in WinNat. Please refer this article for more details

https://docs.microsoft.com/en-us/aspnet/mvc/overview/deployment/docker-aspnetmvc

The IP address of the container needs to be figured out by running this command.

```
docker inspect -f "{{ .NetworkSettings.Networks.nat.IPAddress }}" aspnetcontainer

``` 
![](/screenshots/pic7.jpg)

When the container IP is accessed, it renders the ASP.NET MVC 4.6 app running inside windows container.
![](/screenshots/pic8.jpg) 


#  5.	Amazon EC2 Container Registry
OK! So you’ve created and ran the Docker image locally – now, how do we run it in the cloud?

Amazon ECS can access container images stored in docker hub (private / public), your organization’s container repository, or Amazon EC2 Container Registry (Amazon ECR).

In this post, we’ll use Amazon ECR because it’s fast, secure, and low cost. The Amazon EC2 container registry is a fully-managed Docker container registry that makes it easy for developers to securely store, manage, and deploy Docker Container images. It is deeply integrated with Amazon ECS, simplifying development to production workflow.

## 5.1 Create an Amazon ECR repository

Each container image should be stored in its own repository on Amazon ECR. Use the AWS console to create a new Amazon ECR repository for your image:

 
![](/screenshots/pic9.jpg)

After you create the repository, the AWS console will show you pre-filled code to authenticate to the repository with Docker, tag the image, and push the image to Amazon ECR.

## 5.2		  Log in to ECR 

The docker log in command needs to be retrieved to authenticate the Docker client into the registry.You will need to modify this code to reflect the region you created your Amazon ECR repository in.

aws ecr get-login --no-include-email --region ap-southeast-2

![](/screenshots/pic10.jpg)

Enter the Docker log in command from the last step:
![](/screenshots/pic11.jpg) 

You should see a similar input when the login is successful.


## 5.3 Tag the container image
The Container image that was built in the local development environment needs to be tagged with the ECR repository. For this example, we’ll use `:latest` if you are pushing many versions of an image, consider using a numerical tag structure.

```
docker tag awsecssample:latest awsaccountnumber.dkr.ecr.awsregion.amazonaws.com/containerimagename:latest

``` 
![](/screenshots/pic12.jpg)

## 5.4 Push the container image
Run the docker push command to push the newly created image to the ECR repository.

```
docker push awsaccountnumber.dkr.ecr.awsregion.amazonaws.com/awsecssample:latest
```
![](/screenshots/pic13.jpg)

 

The container image is encrypted and compressed in the Amazon ECR repository. The actual size of this container image is around 11GB in the local development environment. In the ECR repository it’s size is around 465 MB, which is a lot of space savings! 

![](/screenshots/pic14.jpg)



The required permissions need to be provided to the container image so that ECS instances can start leveraging that.

# 6. Instantiate the Amazon ECS Cluster
Amazon ECS manages clusters of Amazon EC2 compute instances that it deploys containers on to it. Creating the cluster to run your containers is a key step in scheduling and orchestrating containers in AWS. There are two options for creating ECS Windows Cluster. The first one is to leverage the cloud formation template provided in this link http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html, which provisions ecs cluster and other related resources end-to-end. The second one is to manually create a cluster, set up container instances, ecs agent and other dependencies mentioned in this link http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Windows_getting_started.html. The first option of leveraging Cloud formation template for ecs windows cluster is leveraged in this article.


## **6.1	Custom Cloud Formation template**
The ECS Task definition, ECS Cluster definition and IAM roles are modified in the default cloud formation template mentioned in section 6 is modified to create a custom stack. The customized template is attached below.![](/screenshots/pic15.jpg)
 

## **6.2	ECS Task Definition**
The task definition for running windows container image (ASP.NET MVC) should look like this.

```
"taskdefinition": {
            "Type": "AWS::ECS::TaskDefinition",
            "Properties": {
                "ContainerDefinitions": [{
                    "Name": "aws_ecs_sample",
                    "Cpu": "100",
                    "Essential": "true",
                    "Image": "yourawsaccountnumber.dkr.ecr.ap-southeast-1.amazonaws.com/sundardocker:latest",
                    "Memory": "500",
                    "LogConfiguration": {
                        "LogDriver": "awslogs",
                        "Options": {
                            "awslogs-group": {
                                "Ref": "CloudwatchLogsGroup"
                            },
                            "awslogs-region": {
                                "Ref": "AWS::Region"
                            },
                            "awslogs-stream-prefix": "aws_ecs_sample"
                        }
                    },
                    "PortMappings": [{
                        "ContainerPort": 80
                    }]
                }]
            }
        }


```
## **6.3 ECS Cluster creation and configuration**

The custom cloud formation template created in section 6.1 is validated in the cloud formation designer. The stack creation is initiated by referring to the custom template stored in S3.

![](/screenshots/pic16.jpg)

The desired capacity, instance type, key name, max size, subnetid and vpc id are provided. 

![](/screenshots/pic17.jpg)

The IAM role for executing cloud formation stack is left with default. Now the stack creation is initiated.

![](/screenshots/pic18.jpg)


The windows container images are generally larger in size. In our case it is around 11 GB. So it takes few minutes to create and configure the entire cluster. The Cloud Formation stack creation is successful. 

![](/screenshots/pic19.jpg)

It creates all the resources mentioned in the section 6.1.

## **6.4 Verification**
Once the windows container is up and running it can be verified  in the console.

![](/screenshots/pic20.jpg)

Hit the application load balancer url.

![](/screenshots/pic21.jpg)

The ASP.NET MVC 4.6 application is rendered in the browser.
 
 ![](/screenshots/pic22.jpg)

In this demo we walked through how you can use standard Windows developer tools, Docker, Amazon ECR, and Amazon ECS to deploy a legacy .NET web server to AWS using containers. These steps can be used as a template for containerizing and running other applciations written in legacy .net code.

For more information on using Amazon ECS, please visit the official product page or documentation.

If you found this article useful, please share. If you have questions, please use the comments below.

-- Sundar Narasiman,Thomas Fuller and Arun Kannan

