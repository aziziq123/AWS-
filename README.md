Deploying MongoDB on an AWS EC2 Instance

This guide walks through provisioning an Ubuntu EC2 instance on AWS and installing MongoDB on it, including a known issue encountered during setup and how to work around it.

Prerequisites
An AWS account with permissions to launch EC2 instances
An existing key pair (.pem file) and security group, or create new ones during launch
Git Bash (or another SSH client) installed locally on your machine
1. Launch the EC2 instance
In the AWS EC2 console, click Launch instance.
Give the instance a descriptive name (e.g. mongodb-server).
Choose an Ubuntu AMI (e.g. Ubuntu 24.04 LTS).
Choose an instance type — t3.micro is free-tier eligible and sufficient for development/testing.
Under Key pair, either:
Select an existing key pair you already have, or
Create a new one and download the .pem file — store it somewhere safe, as it cannot be re-downloaded if lost.
Under Network settings, select or create a security group that allows:
SSH (port 22) — required to connect to the instance
HTTP (port 80) — only needed if also serving a website from this instance
Optionally, a custom rule opening port 27017 if MongoDB needs to be reached remotely (not required if only connecting locally on the instance)
Click Launch instance and wait for it to reach the Running state.
Note down the instance's Public IPv4 address from the instance summary page.
2. Connect to the instance

From a terminal with your .pem key file accessible, run:

bash
ssh -i /path/to/your-key.pem ubuntu@<instance-public-ip>

On first connection, you'll be asked to confirm the host's authenticity — type yes and press Enter.

Note: an instance's public IP can change if it is stopped and restarted, unless an Elastic IP is attached to it. Always check the instance's current IP in the EC2 console before trying to reconnect, rather than reusing an old one.

3. Update the system

Before installing anything, update the package lists and upgrade existing packages:

bash
sudo apt update
sudo apt upgrade -y

This ensures the instance's base software is current and reduces the risk of installing on top of outdated or insecure packages.

4. Install MongoDB

Import MongoDB's GPG signing key and add its official repository. This guide uses MongoDB 7.0 rather than the newest release — see the note in Step 6 for why.

bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

Update package lists again to pick up the new repository, then install:

bash
sudo apt update
sudo apt-get install -y mongodb-org
5. Start MongoDB
bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod

The status output should show active (running). Press q to exit the status view if it opens in a pager.

enable ensures MongoDB automatically starts again if the instance is rebooted.

6. Known issue: MongoDB 9.0 and newer Linux kernels

Symptom: mongod fails to start immediately, with systemctl status mongod showing:

Active: failed (Result: exit-code)

Diagnosis: check the actual error with:

bash
sudo journalctl -u mongod -n 20 --no-pager

If the kernel incompatibility is the cause, the log will show something like:

"msg":"MongoDB cannot start: Linux kernel versions 6.19 and newer has a known incompatibility with this version of MongoDB."

Cause: MongoDB's 9.0 release line has a known compatibility issue with newer Linux kernels (6.19+). This is a genuine bug in that MongoDB version, not a configuration mistake.

Fix: remove the 9.0 package and its repository, then install MongoDB 7.0 instead (an older, more widely tested release that does not have this issue):

bash
sudo apt-get remove -y mongodb-org*
sudo rm /etc/apt/sources.list.d/mongodb-org-*.list

Then repeat Step 4 above to install 7.0, and Step 5 to start it.

7. Verify MongoDB is working

Connect to the local MongoDB shell:

bash
mongosh
