# SSH into AWS EC2 instances on Windows

This section describes how to perform an SSH into an EC2 instance from a Windows machine.

#### Confirming & Enabling SSH

Go to Services > EC2 > Instances and select the running instance. Make sure that there is a key pair listed under "Key pair name". As the addition of a key pair can only happen at the time of cluster/instance creation, if it is not present, then it will not be possible to SSH into the instance. If it is present, go check under "Security groups", which should be a few headings above the "Key pair name" entry. Click on "view inbound rules" (or just ctrl+F it), and confirm whether port 22 is open or not. If not, then either click on the security group name or go EC2 > Network & Security > Security groups and select the group. Under the inbound tab, click "Edit" to enable the SSH port.

#### Setting up PuTTY
When creating a new key pair for the first time, a download prompt should pop up, which will allow you to save a ".pem" file. Open up PuTTYgen and select RSA as the generator. Load the .pem file, which will import it. Select "Save private key" and save it in the .ppk format for use with PuTTY.

Open up PuTTY and do the following:
1. Under Connection > SSH > Auth:
	1. Click "Browse" and select the .ppk file that was created
2. Under Session
	1. For the username, use: `ec2-user@<Public DNS (IPv4) Address>`
	1. Port: 22
	1. Connection type: SSH

This should allow access to the EC2 instance. There may be a certificate warning if it is the first time logging into the instance.

For information on retrieving, adding, and managing key pairs, go to [AWS' guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#replacing-key-pair).

---
Related links:

[PuTTY link](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html "PuTTY link").

AWS:[Authorizing access to an instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html)
[AWS PuTTY guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html)
