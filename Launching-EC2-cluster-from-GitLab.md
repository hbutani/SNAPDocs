# Launch SNAP on EC2 from GitLab

You can launch an EC2 cluster directly from GitLab with any desired version of SNAP.

## Launch the desired cluster from a specified commit
1.  Go to the pipelines page https://gitlab.com/SparklineData/snap/pipelines (found under "CI / CD" -> Pipelines)
2.  Press the button in the "Status" column of the commit ID you want to launch.  
![pipelines](/uploads/c5fc25d663a60b74f244e24538131852/pipelines.png)
3.  Press the "Play" button for the type of deployment you want.  Currently the deployments are parameter by instance-type and number of slaves
![pipeline-details](/uploads/f30a11201b09debb31156e5a60c376c2/pipeline-details.png)
 
## What this will do:

1.  Start the cluster using flintrock
2.  Copy the installer to /tmp/snap_installer.sh
3.  Install snap in /home/ec2-user/snap
4.  Create /media/ephemeral<n>/snapcache directories on master and slaves
5.  Initialize SNAP using command: `snap-tool --cache /media/ephemeral0/snapcache,/media/ephemeral1/snapcache init` (all /media/ephemeral<n>/snapcache) will be used)
6.  Start SNAP Server with command `snap-tool start` (SNAP Server will connect to the master at port 7077 automatically)

## Access the cluster remotely
You will need to add your local IP to the flintrock security-group via the AWS console:in 
### 1.  Go to the AWS EC2 console: 
https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:sort=desc:tag:aws:ec2spot:fleet-request-id

### 2.  See the EC2 instances you just created:
![ec2-instances](/uploads/c77acd7b106441c69972cf6e371ec276/ec2-instances.png)
### 3.  Go to the `flintrock` security-group 
https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups:groupName=flintrock$;sort=groupId

### 4.  Add a new rule for your IP address
Under the "Inbound" tab, press "Edit" button.  Scroll to the the bottom of the pop-up window, press "Add Rule" to add rules for port 10000, 8080, and 22 for "My IP"
![add-rule](/uploads/a0d274f23845a15d7396c33af64d44a1/add-rule.png)

### 5.  You can now access the cluster!
Get the ` <instance-ip>` from the "Public DNS" column on the EC2 console (see screenshot in step 1)
*  SSH: `ssh -i /path/to/sri-1.pem ec2-user@<instance-ip>`
*  SparkUI: go to this URL: `http://<instance-ip>:8080/`
*  JDBC via notebooks: `%%sql hive://<instance-ip>:10000/default?auth=NOSASL`

### 6.  Destroy the cluster
On the "Pipeline" page, press the "Play" on the "destroy-ec2" stage to start destruction of the cluster


## Limitations and Issues:

Overall, this is a temporary setup to allow deployment to AWS without needing to download flintrock and getting hands-dirty with AWS.  
In the future, we will need more robust and flexible infrastructure automation

*  Only 1 deployment of a specific commit can be running at a time (because cluster-name is based on the commit-id)

*  The SNAP Server is started with the default parameters in the `sparkline.properties`.  You will want to change these by:
  1.  Shutting down snap: `/home/ec2-user/snap/bin/snap-tool stop`
  2.  Editing the `/home/ec2-user/snap/conf/sparkline.properties` file
  3.  Set any environment variables needed for S3 (i.e AWS_SECRET_ACCESS_KEY and AWS_ACCESS_KEY_ID)
  4.  Starting up snap: `/home/ec2-user/snap/bin/snap-tool start`

*  Launching from gitlab requires pressing just 1 button, but it is slow (about 5 minutes end-to-end) because:
  1.  GitLab is pulling the latest snap-docker container so it can run flintrock
  2.  Copying the installer to EC2 is slow sometimes
  3.  Its sending multiple commands to cluster over the network instead of running a single start-up script on each instance

## How this works:

All of the setup is contained in the `.gitlab-ci.yaml` file found [here](https://gitlab.com/SparklineData/snap/blob/master/.gitlab-ci.yml)

The steps for setup look like this:
```yaml
.launch_ec2_template: &launch_ec2
  before_script: *prepare_flintrock
  stage: deploy
  image: ${SNAP_DOCKER_IMAGE}
  when: manual
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: ""
  script:
    - git clone ${REPO} repo
    - echo "Deploying commit=${CI_COMMIT_SHA:0:8} cluster=snap-${CI_COMMIT_SHA:0:8} user=${GITLAB_USER_EMAIL} repo=${REPO} parameters=${LAUNCH_ARGS}"
    - flintrock --config repo/config.yaml launch snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem --ec2-key-name ${PEM_NAME} ${LAUNCH_ARGS}
    - echo "You can now connect to your cluster here (add your IP to the AWS flintrock security-group using the aws console"
    - echo http://$(flintrock describe | egrep "master:(.+)" | grep -Po "\S+$"):8080
    - echo "Copying snap_installer.sh to /tmp/snap_installer.sh on master"
    - flintrock --config repo/config.yaml copy-file snap-${CI_COMMIT_SHA:0:8} snap_installer.sh /tmp/snap_installer.sh --master-only --ec2-identity-file ${PEM_NAME}.pem
    - flintrock --config repo/config.yaml run-command snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem --master-only -- "chmod a+x /tmp/snap_installer.sh"
    - flintrock --config repo/config.yaml run-command snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem --master-only -- "SNAP_SECRET=${SNAP_SECRET} /tmp/snap_installer.sh --target /home/ec2-user/snap"
    - flintrock --config repo/config.yaml run-command snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem -- "find /media -name ephemeral* -mindepth 1 -maxdepth 1 -type d -exec mkdir {}/snapcache \;"
    - flintrock --config repo/config.yaml run-command snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem --master-only -- '/home/ec2-user/snap/bin/snap-tool --cache `find /media/  -mindepth 1 -maxdepth 2 -type d -name snapcache* | paste -s -d,` init;'
    - flintrock --config repo/config.yaml run-command snap-${CI_COMMIT_SHA:0:8} --ec2-identity-file ${PEM_NAME}.pem --master-only -- '/home/ec2-user/snap/bin/snap-tool start'
```

And the parameter deployment options look like this:

```
i3.8xlarge (Expensive) :
  <<: *launch_ec2
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: --ec2-instance-type i3.8xlarge

(2x) i3.8xlarge (Most Expensive):
  <<: *launch_ec2
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: --ec2-instance-type i3.8xlarge --num-slaves 2

(2x) i3.4xlarge (Commonly used):
  <<: *launch_ec2
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: --ec2-instance-type i3.4xlarge --num-slaves 2

i3.xlarge (Most Cheap):
  <<: *launch_ec2
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: --ec2-instance-type i3.xlarge

(2x) i3.xlarge (Cheep):
  <<: *launch_ec2
  variables:
    REPO:  https://gitlab.com/SparklineData/Public/SNAP-on-EC2.git
    LAUNCH_ARGS: --ec2-instance-type i3.xlarge --num-slaves 2
```
 

