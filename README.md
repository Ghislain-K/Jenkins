# Jenkins: Automated Daily Backups of Jenkins Master to Amazon S3 Bucket

Automated Daily Backups of Jenkins Master to Amazon S3 Bucket


Owned by ................

Jan 12, 20225 min read
3 people viewed
 


Summary
In this article we are going to create a Jenkins job that backs up our Jenkins Master to an Amazon S3 Bucket.

Instead of taking a snapshot of the entire volume, you can choose to just back up the $JENKINS_HOME directory, which contains your Jenkins-specific configurations. When you restore, you simply launch a new Jenkins master and replace the $JENKINS_HOME directory with the contents of your backup. — AWS best practices

Prerequisites
Amazon EC2 Linux Instance with Jenkins installed

If you do not have an EC2 instance with Jenkins installed I recommend using bitnami’s Jenkins AMI which can be found in the AWS marketplace tab in the first step (Choose AMI) of launching an instance.


AWS Marketplace — Jenkins Certified by Bitnami AMI

Steps
Download AWS CLI to EC2 Instance

Create S3 Bucket and IAM User

Configure Jenkins Environment Variables and Job

1. Download AWS CLI to EC2 Instance
Once logged into your AWS account go to your EC2 dashboard, check the instance where your Jenkins is installed and click the ‘Connect’ button. Here we should see a popup modal with instructions to connect to the instance using ssh.


SSH into Instance

Follow the instructions provided by AWS in relation to your private key file, then in your bash shell SSH into the EC2 instance.


$:ssh -i "YOUR_PRIVATE_KEY.pem" ubuntu@ec2-1-23-456-78.us-east-2.compute.amazonaws.com
We rely on aws-cli to upload the Jenkins configuration backup to the Amazon S3 bucket. Once we successfully SSH into the instance we can download awscli. We will need to do this as root.


$:sudo apt install awscli
...
If successful the aws --version command should output the aws-cli version you have installed.


$:aws --version
aws-cli/1.11.13
2. Create S3 Bucket and IAM User
Next we need to create an S3 bucket to push our Jenkins backups to. In your AWS account navigate to S3 and create a new bucket.


Creating an S3 Bucket for Jenkins Backup Job

Remember to choose a unique name for your S3 bucket as specified by AWS.

Bucket names must be unique across all existing bucket names in Amazon S3. — AWS

Go with the default settings as we just need an S3 bucket, nothing fancy. We can always come back and add more restrictions if needs be. Review & Create bucket. You should now see a new bucket in your S3 dashboard.


Keep note of the bucket name as we will need to add this to our build script when creating our Jenkins job.

Creating an IAM user
For us to be able to upload to S3 we will need an IAM user. This can be found under Security, Identity and Compliance in your AWS console. Give it a username and check ‘Programmatic access’.


IAM user creation

In step 2 add the user to the default ‘admin’ group.


Add IAM user to admin default group

you can skip the following steps and go straight to Review & Create. Take note of your access key and secret access key as we will need these so that we can upload to S3 from Jenkins with authentication. We will also need the region of your S3 bucket. You can view the region in your bucket in the S3 dashboard then use regions and availability zones to get the relevant code. At this stage you should have the following;


# Jenkins environment variablesAWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID>
AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY_ID>
AWS_DEFAULT_REGION=<YOUR_S3_BUCKET_REGION>
3. Configure Jenkins Environment Variables and Job
Lets add our environment variables to our Global properties so that we can upload to our S3 bucket. In your Jenkins Master, navigate to;

Manage Jenkins > Configure System > Global Properties > Environment Variables.

Once there we can add our environment variables.


Adding environment variables to Jenkins Global Properties

Now that we have our IAM user added to our environment we will be able to successfully upload to S3, lets create a new Jenkins job. Click ‘New item’ from the left nav panel of Jenkins.

Lets call this job ‘jenkins-backup’ and choose ‘Freestyle project’ for project type.


Jenkins Backup Job Creation

We only have 2 configurations we need to add for the ‘jenkins-backup’ job;

Build Triggers > Build Periodically

Build > Execute Shell

First we want to setup our CRON job. Go to ‘Build Triggers’ tab, for this job I want it to run every day at midnight, so we can set it to ‘0 0 * * *’. Crontab.guru is a handy tool if you want to configure your job to run at a more specific time.


CRON job set to run at midnight every night

Next we want to add a script which will do the following;

tar gzip the ‘jenkins_home’ directory

push our tar file to S3 bucket.

remove all files after successful upload.


echo 'tar $JENKINS_HOME directory'
set +e 
tar -cvf jenkins_backup.tar -C $JENKINS_HOME .
exitcode=$?
if [ "$exitcode" != "1" ] && [ "$exitcode" != "0" ]; then
exit $exitcode
fi
set -eecho 'Upload jenkins_backup.tar to S3 bucket'
aws s3 cp jenkins_backup.tar s3://<YOUR_BUCKET_NAME>/echo 'Remove files after succesful upload to S3'
rm -rf *

Script to create .tar file and upload to S3

Save the job. Although we have it set to run at midnight every night, we want to make sure it definitely works, lets go ahead and build it to be sure. If all works as expected you should see a successful build.


Successful Jenkins Build

Check the ‘Console Output’ to get a better understanding of how the script ran. Next we want to visit our S3 bucket and make sure it has uploaded a ’jenkins_backup.tar’ as expected.


Add label
