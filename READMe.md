# Continuous Integration and Continuous Delivery/Deployment (CICD) Pipeline
![image](https://user-images.githubusercontent.com/88166874/132583482-6d8060cb-54ac-43d7-8d7c-0ba62a9549a9.png)
## Prerequisites
- GitHub repo with all of the required directories for the app (clone this repo, then make a new one, connect to it through ssh, and push the code to the new repo) -> [see this README](https://github.com/am93596/SRE_Github_SSH/blob/main/README.md)  
- Jenkins server  
- AWS account with EC2 instances for the app and db -> [see this repo](https://github.com/am93596/SRE_intro_cloud_computing_two_tier_arch)  
## Jenkins
![image](https://user-images.githubusercontent.com/88166874/132583118-1a674c3b-0a95-420f-96bd-ea7c3a95ab29.png)
### Setting Up A New Job
- Click `New Item`
- Give the item a name, then click `Freestyle project`
- Click `OK`
### Configuration For CI job
- This job will run the automated tests on the new code  
Step 1) Give the job an accurate description (e.g. `Building a CI job with GitHub and localhost`)  
Step 2) Click `Discard old builds` and set `Max # of builds` to `3`  
Step 3) Click `GitHub project` and put in the https clone link for your repo  
Step 4) In `Office 365 Connector`, tick `Restrict where this project can be run`; `Label Expression`: `sparta-ubuntu-node`  
Step 5) `Source Code Management`: `Git`. Paste in the ssh clone link for the repo, and add the private key for your repo  
  Step 5a) Click `Add` (the one with a key on it), then `Jenkins`, and set `Kind` to `SSH Username with private key`  
  Step 5b) Give it a description so you can identify it  
  Step 5c) Beside `Private Key`, select `Enter directly`, and paste in the whole contents of the private key file  
  Step 5d) Click `Add`  
Step 6) `Branch Specifier`: `*/dev`  
Step 7) Tick `GitHub hook trigger for GITScm polling`  
Step 8) Tick `Provide Node & npm ...`; `NodeJS Installation`: `Sparta-Node-JS`  
Step 9) In `Build`, select `Add build step`, then `Execute shell`  
Step 10) Paste this code into the `Command` box:  
```bash
cd app
npm install
npm test
```  
Step 11) Click `Apply`, then `Save` -> We will come back to `Post-build Actions` later  
### Configuration for CD Job
- This job will merge the newly-tested code with the main branch if the tests were successful  
Step 1) Make another job for `Merging the dev branch with main after successfully passing tests`  
Step 2) Repeat steps 1, 2, 3, 5, and 6 from `Configuration For CI job`; don't do Step 4, 7, 8 or 9  
Step 3) Click `Git Publisher`, then tick `Push Only If Build Succeeds`, and `Merge Results`  
Step 4) Click `Apply`, then `Save` -> We will come back to `Post-build Actions` later  
## Configuration for CDE Job
- This job will deploy the newly-merged code into the EC2 instances  
Step 1) Make another job for `Taking the code from the GitHub repo and deploys it in an EC2 instance`  
Step 2) Repeat steps 1, 2, 3, and 5 from `Configuration For CI job`; don't do Step 4, 7, 8 or 9  
Step 3) In `Branch Specifier`, enter `*/main`  
Step 4) In `Build Environment`, tick `SSH Agent`, and fill in the details for the key required to log in to the EC2 instances (e.g. from `sre_key.pem` file)  
Step 5) In `Build`, add `Execute shell`, and paste in the following - REDO THE EC2 IDS AND IP ADDRESSES  
```bash
ssh -A -o "StrictHostKeyChecking=no" ubuntu@ec2-34-254-188-238.eu-west-1.compute.amazonaws.com  << EOF
#sudo dpkg --configure -a
rm -rf sre_jenkins_working_app
git clone https://github.com/am93596/sre_jenkins_working_app.git
cd sre_jenkins_working_app/environment/db
sudo chmod +x provision.sh
./provision.sh
EOF
ssh -A -o "StrictHostKeyChecking=no" ubuntu@ec2-176-34-155-161.eu-west-1.compute.amazonaws.com  << EOF
#sudo dpkg --configure -a
killall node
rm -rf sre_jenkins_working_app
git clone https://github.com/am93596/sre_jenkins_working_app.git
cd sre_jenkins_working_app/environment/app
sudo chmod +x provision.sh
./provision.sh
export DB_HOST=34.254.188.238:27017/posts/
cd ~/sre_jenkins_working_app/app
npm install
node seeds/seed.js
nohup node app.js > /dev/null 2>&1 &
EOF
```  
Step 6) Click `Apply`, then `Save`  
### Connecting The CI, CD, and CDE Jobs
Step 1) In your first job, click `Configure`, then `Add post-build action` -> `Build other projects`  
Step 2) Enter the name of the second job we created, and select `Trigger only if build is stable`  
Step 3) In your second job, click `Configure`, then `Add post-build action` -> `Build other projects`  
Step 4) Enter the name of the third job we created, and select `Trigger only if build is stable`  
### GitHub Webhook
Step 1) In your GitHub repo, click `Settings` -> `Webhooks` -> `Create webhook`, and type in your password if prompted  
Step 2) `Payload URL`: `http://18.170.212.241:8080/github-webhook/` - REPLACE `18.170.212.241` WITH THE CURRENT JENKINS IP  
Step 3) Click `Send me everything`  
Step 4) Click `Add webhook`  

Now make a new dev branch in your local repo, make a change, and push it -> then give Jenkins some time, before navigating to the IP address for the app EC2 instance, then IP/posts. Both should appear.
