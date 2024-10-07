---
tags: 
 - blog
 - cloud9
layout: post
title: Cloud9 is dead, long live VSCode Remote!
excerpt: |
  After the deprecation of Cloud9, VSCode remote seems a good alternative. Here is how we can quicly set it up on AWS. 

img_url: /images/2024-10-04-k7.png
img_credits: Photo by <a href="https://unsplash.com/@chalkpitcassetteclub?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Silas Gregory</a> on <a href="https://unsplash.com/photos/a-person-holding-a-blue-cassette-in-front-of-a-metal-machine-ve3Z4ukE3TY?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
  
---
## Context

As an AWS partner, I often support some AWS workshop, helping customers master the craft of using the correct AWS services. Those workshops are prepared by AWS and very often use Amazon Cloud9, a service that provides an IDE in the cloud. It is really convenient as you don't have to provide any credentials: as long as you're logged in the AWS console you inherits the permission of the current profile.

On July 25th 2024, [AWS announced](https://aws.amazon.com/blogs/devops/how-to-migrate-from-aws-cloud9-to-aws-ide-toolkits-or-aws-cloudshell/) the retirement of the Amazon Cloud9 service. Of course the thousands of workshops using Cloud9 have not been updated to adapt to that change. When running the workshop at an AWS event, they can start new Cloud9 instances, but on your own account, it's not possible to start a new instance anymore. In this post, I will present how quickly create a development environment that provides the same features than Cloud9.  

*Disclaimer:* the installation is for POSIX like systems, whereas it should be feasible to user the same technique with Windows, I can't test it thoroughly.

## Prerequisites

There are only a few prerequisites. Let's start by CDK that can be installed with the following command:

```shell
$ npm install -g aws-cdk
```

Just in case you don't have `node` installed on your computer, you can have a look at the [installation page](https://nodejs.org/en/download/package-manager).

The next one is VisualStudio Code that you download [here](https://code.visualstudio.com/download) with the `Remote Development` extension that you need to install.

![RemoteDevelopment extension](/images/2024-10-04-remote-extension.png)

Finally you need some AWS credentials to connect to your account. The creation and configuration of those credentials is outside the scope of this blog post. You can refer to the [AWS documentation](https://docs.aws.amazon.com/cli/v1/userguide/cli-configure-files.html) for that.

## Launching the compute environment

Then we need to launch an EC2 instance that we can reach from our development environment. To secure the connection we will place the EC2 instance in a public subnet attached to a security group that only allows connections from the development environment IP on the SSH port (22). 

![Architecture](/images/2024-10-04-RemoteVSCode.png)

Hopefully, everything here has been automated with the help of CDK.

```shell
$ git clone https://github.com/dmetzler/aws-workshop-ide.git
$ cd aws-workshop-ide
$ cdk deploy
```

at the end of the deployment you should have the following message that gives you the public adress of the EC2 instance. That EC2 instance is only accessible from your current public IP as it is protected by a security group.

![Architecture](/images/2024-10-04-cdkdeploy.png)

The following step have been run by the CDK automation:

- Deployment of an SSH keypair using your own `~/.ssh/id_rsa.pub`
- Creation of a security group allowing only your public IP to connect through SSH
- Deployment of an EC2 instance in a public subnet, attached to that security group


## Remote Connection with VisualStudio Code

Now it's time to start VisualStudio and connect to a remote host using the button in the lower left corner of the VisualStudio Code window.

![Remote Connect](/images/2024-10-04-start-remote.png)

The select the `Connect to host` entry in the drop down list and enter the following:

```
ec2-user@ec2-XXXXXXXXXXXXX.compute.amazonaws.com
```
by replacing the part after the `@` with the public address given in the CDK output. A new window opens where you can validate the fingerprint of the connection. That's it: you're on a remote connection where you can open remote folders:

![Remote Folder](/images/2024-10-04-open-folder.png)

Select and open the `environment` folder. You can now create some files and code directly on the remote instance. The beauty of the setup is that VisualStudio can automatically forward the ports of the EC2 instance to your local instance. That allows for instance to code a small API in Python and being able to browse it locally. Let's do it!

Create a file named `api.py` in the `environment` folder and fill it with this content:

```python
from fastapi import FastAPI
import boto3

client = boto3.client('sts')


app = FastAPI()

@app.get("/")
async def root():
  identity = client.get_caller_identity()
  return {
    "message":f"Hello {identity['UserId']}!",
    "roleArn":identity['Arn']
  }
```

In the `Terminal` menu, click on the `New Terminal` item: it opens a terminal in the bottom section of the window. Now run the following commands:

```shell
$ pip3 install boto3 fastapi[standard]
$ fastapi dev
```

VSCode automatically detects that some ports can be forwarded and offers to Preview the result either in your own local browser, or in a preview window in the editor. Here is a sample screenshot showing the resulting setup, and yes THIS IS ALL REMOTE:

![Final VSCode](/images/2024-10-04-final-vscode.png)




## Cleaning up

When you're finished, you probably want to cleanup your instance, at least to avoid incurring charges ! Thanks to CDK it is as easy as launching the following command in the local `aws-workshop-ide` folder:

```shell
$ cdk destroy
```

## Conclusion

Cloud9 is dead, probably for the better: it was becoming old and I encoutered a few problems in the past to just provision the correct instance type.
By using a remote VSCode setup, we can accomplish exactly the same kind of configuration, allowing to code remotely and inherit from the remote instance role permissions.
The CDK automation can of course be customized to fit your preferences like:

- Choosing a custom VPC to launch the EC2 instance
- Specifying a different AMI
- Increasing the default EBS size.

Feel free to comment here, add issues to the repository or even send PR to contribute to it! 
