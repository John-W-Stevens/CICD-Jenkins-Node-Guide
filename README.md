### Create a CI/CD Pipeline in Jenkins for a Node.js App deployed on a DigitalOcean NodeJS droplet
###### Credit where credit is due:
This guide is based off a guide written by [Moshe Ezderman](https://medium.com/@mosheezderman/how-to-set-up-ci-cd-pipeline-for-a-node-js-app-with-jenkins-c51581cc783c). Despite my slightly different approach (I used different pre-configured DigitalOcean droplets, didn't create additional users, had to install Java for Jenkins to work, etc.) I would like to publicly thank (and give credit too) Mr. Ezderman for this very helpful guide.

#### Overview
A user walking through this guide will get hands-on experience with the following:
* Creating and running two seperate virtual machines (called droplets) hosted on DigitalOcean (a cloud platform like AWS or Azure)
    * Machine 1 will run a small Node.js/Express application
    * Machine 2 will run a Jenkins server
* Creating a small Node.js application using Express, Mocha, & Supertest 
    * Express is a popular Node.js backend framework
    * Mocha and Supertest are used for testing
* Installing, configuring, and using Jenkins (one of the most popular build tools out there) to setup a CI/CD pipeline
    * Creating/configuring a Jenkins "job" (in this case, a process you want run every time your app's master branch is updated on GitHub)
* Creating webhooks on GitHub (connecting source control with the Jenkins build process)
* Configuring auto-deployment (the CD portion) upon successful test completion.
* Establishing an automated SSH connection between two virtual machines 

#### Step 1 - Create GitHub repo
* Create a new public repositiory called "node-app"
* Choose "Initialize this repository with a README" checkbox
* Select "Node" in the "Add .gitignore" drop-down menu
* Click "Create Repository" button

#### Step 2 - Clone node-app repo to your computer
* git clone ```git@github.com:your-github-usernam/node-app.git```
* cd node-app

#### Step 3 - Create Node.js app
* Create package.json file in node-app root directory (which you should already be)
* Copy and paste this code into it:
   ```
    {
        "name": "node-app",
        "description": "Simple Node.js/Express app for CI/CD Jenkins demo",
        "version": "0.0.1",
        "private": "true",
        "dependencies": {
            "express": "3.12.0"
        },
        "devDependencies": {
            "mocha": "1.20.1",
            "supertest": "0.13.0"
        }
    }
    ```
* Run ``` "npm install" ``` in node-app root directory
* Create a new file, "node-app/index.js" and past this code into it:
    ``` 
    var express = require("express");
    var app = express();

    app.get("/", function(req, res){
        res.send("Hello World. Jenkins is great!");
    });

    app.listen(process.env.PORT || 3001);
    module.exports = app;
* Test your app, run ``` "node index.js" ```
* Navigate to ```http://localhost:3001``` and you should see "Hello World. Jenkins is great!"

#### Step 4 - Add a test to Node.js app
* In node-app root directory create "/test/" directory (run: ``` mkdir test ```)
* Inside node-app/test create "test.js"
* Paste the following code in node-app/test/test.js:
    ``` 
    var request = require('supertest');
    var app = require('../index.js');

    describe('GET /', function(){
        it('respond with hello world', function(done){
            request(app).get('/').expect('Hello World. Jenkins is great!', done);
        });
    });
    ```
* Make sure your test works, run this command from node-app root directory:
    ```
    ./node_modules/.bin/_mocha ./test/test.js
    ```
* ###### You should see "1 passing" in your terminal. If not, review the code above.

#### Step 5 - Add a script to Node.js app
* In node-app root directory create "/script/" directory (run: ``` mkdir script ```)
* Create a file named "test", without a file extenstion, inside node-app/script/ and paste in this code:
    ```
    #!/bin/sh
    
    ./node_modules/.bin/_mocha ./test/test.js
    ```
* Grant executable permission, run this command in the node-app root directory:
    ```
    chmod +x script/test
    ```
* Test your script by running the following command fron node-app root directory:
    ```
    ./script/test
    ```
* ###### You should see "1 passing" in your terminal. If not, review the code above.

#### Step 6 - Push code to GitHub
* Run the following command in the node-app root directory:
    ```
    git add .
    git commit -m "initial commit of simple Node.js app with test"
    git push origin master 
    ```

#### Step 7 - Serve Node App
* Sign in to your DigitalOcean Account (sign up for an account if you don't already have one: https://www.digitalocean.com/ - you will get a $100 free credit which is more then enough for this demo.)
* Click "Create" and select "Droplets" from the drop-down menu
* Under "Choose an image" tab select "Marketplace"
* Click "See all Marketplace Apps"
* Search for "NodeJS"
* Select "NodeJS 12.18.0 on Ubuntu 20.04"
* On NodeJS page, click "Create NodeJS Droplet"
* Under "choose a plan" select the cheapest option ($5/mo)
* Under "choose a datacenter region" choose a region close to you
* Under authentication add your SSH key. If you don't have one, follow the instructions [here](https://docs.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).
* Use this command to copy your SSH public key and paste it into the textfield:
    ```
    pbcopy < ~/.ssh/id_rsa.pub
    ```
* Under "Choose a hostname" add "nodejs-app"
* Click the "Create" button
* ###### Your droplet will be ready in a few seconds.

#### Step 8 - Configure your Node Droplet
* SSH into your nodejs-app droplet (run: ``` ssh root@NODE.SERVER.IP ```)
* Install git (run: ```sudo apt-get install git```)
* Clone node-app repo:(run ```git clone https://github.com/github_username/node-app.git ```)
* Navigate into node-app directory (run: ```cd node-app ```)
* Install app dependencies(run: ```npm install - production ```)
* Unblock port 3001 (run: ``` sudo ufw allow 3001 ```)
* Fire up your app (run: ```node index.js```)
* Navigate to ```http://NODE.SERVER.IP:3001``` in your browser
* Install pm2 (Like supervisor or gunicorn, this will run your app continously)
    ```
    sudo npm install pm2@latest -g
    pm2 start index.js
    ```
#### Step 10 - Setup Jenkins Server
* Follow the same instructions in Step 7 above, choosing "jenkins-app" as your hostname (create a second droplet that will run our Jenkins server)

#### Step 11 - Configure Jenkins server
* SSH into new droplet as root user (run: ```ssh root@JENKINS.SERVER.IP ```)
* Install git (run: ```sudo apt-get install git```)
* Update, Install Java, Install Jenkins:
    ```
    sudo apt-get update
    sudo apt install openjdk-8-jre
    sudo update-alternatives --config java
    wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
    sudo apt-add-repository "deb https://pkg.jenkins.io/debian-stable binary/"
    sudo apt install jenkins
    ```
* Start Jenkins and check status (if it says failed, go over the above steps)
    ```
    sudo systemctl status jenkins
    ```
* Remember DigitalOcean's firewall? Open up port 8080:
    ```
    sudo ufw allow 8080
    ```
* Navigate to ```http://JENKINS.SERVER.IP:8080``` (you should see Jenkins homepage saying something about unlocking Jenkins)

#### Step 12 - Unlock Jenkins
* Open the Jenkins admin password in VIM by running this command:
    ```
    sudo vim /var/lib/jenkins/secrets/initialAdminPassword
    ```
* Copy the password to clipboard and then exit VIM (esc, :wq, enter)
* Use this password to unlock Jenkins
* Select "Install Suggested Plugins"
* After Jenkins finishes configuring, follow the "Create First Admin User" instructions
* On the "Instance Configuration" page, click "Save and Finish"
* Click "Start using Jenkins"


#### Step 13 - Setup Jenkins job
* Click "New Item" button
* Name the item "node-app"
* Select "Freestyle project"
* Click "ok"
* In the "Source Code Management" tab, select the git radio button and enter the github https link to the node-app repo
* Under "Build Triggers" select "Github hook trigger for GITScm polling".
* Click on "Add Build Step" button and select "Execute Shell" option.
* Enter the following commands in the text area:
    ```
    npm install
    ./script/test
    ```
#### Step 14 - Add Git Webhook
* Go to the node-app repo on GitHub
* Click on "settings"
* Select "Webhooks" from the left menu 
* Click the "Add Webhooks" button (on the right)
* Under "Payload URL" enter:
    ```
    http://JENKINS.SERVER.IP:8080/github-webhook/
    ```
* Ensure the "Just the Push Event" option is checked (it should be by default).
* Click "Add webhook" button
* Test the webhook by going to your node-app on your local machine, changing the version in package.json to "0.0.2", committing these changes, and then pushing them to GitHub. If you look at your Jenkins console you should the job being processed. 

#### Step 15 - Establish SSH Authentication between servers
* SSH into Jenkins server (run: ```ssh root@JENKINS.SERVER.IP```):
* Generate root user key
    1. Get key (run: ```ssh-keygen -t rsa```)
    2. Save this key to /var/lib/jenkins/.ssh/id_rsa
    3. Hit enter through the optional passphrase configuration
    4. Print this key (run: ```cat ~/.ssh/id_rsa.pub```)
    5. Copy this key to an empty .txt file
* Generate jenkins user key
    1. Switch to jenkins user (run: ```su - jenkins ```)
    2. Get key (run: ```ssh-keygen -t rsa```)
    3. Save this key to /var/lib/jenkins/.ssh/id_rsa
    4. Hit enter through the optional passphrase configuration
    5. Print this key (run: ```cat ~/.ssh/id_rsa.pub```)
    6. Copy this key to the same .txt file as the root user key (there should be 2 keys there)
* Add both root/jenkins public keys to nodejs-app server:
    1. Exit Jenkins server (run: ```exit```)
    2. SSH into Nodejs-app server (run: ```ssh root@NODE.SERVER.IP```)
    3. Open the file where authorized keys are stored (run: ```sudo vim ~/.ssh/authorized_keys```)
    4. Paste both root/jenkins public keys into this file (press 'i' to insert), save and exit (esc, :wq, enter)
    5. Set correct permissions for the .ssh folder:
        ```
        chmod 700 ~/.ssh
        chmod 600 ~/.ssh/*
        ```
    6. Exit Nodejs-app server (run: ```exit```)


#### Step 16 - Verify SSH Connection
* SSH into Jenkins server (run: ``` ssh root@JENKINS.SERVER.IP```)
* Test root user SSH connection 
    1. SSH into Nodejs-app (run: ```ssh root@NODE.SERVER.IP```)
        * You should see your terminal change, reflecting that you are now logged into the nodejs-app server.
    2. Exit (run: ```exit```)
* Test jenkins user SSH connection
    1. Switch to jenkins user (run: ``` su - jenkins ```)
    2. SSH into Nodejs-app (run: ```ssh root@NODE.SERVER.IP```)
        * You should see your terminal change, reflecting that you are now logged into the nodejs-app server.
    3. Exit (run: ```exit```)

#### Step 17 - Setup Auto-Deployment when test passes
* On your local machine inside node-app/script directory, where the 'test' (without extension) file is located, add a file, without and extension, called 'deploy'
* Paste the following into this node-app/script/deploy
    ```
    #!/bin/sh

    ssh -tt root@NODE.SERVER.IP <<EOF
    sudo ufw allow 3001
    cd node-app
    git fetch --all
    git reset --hard origin/master
    npm install - production
    pm2 restart all
    exit
    exit
    EOF
    ```
* Make this script executable (run: ```chmod +x script/deploy``` from the node-app root directory):

* Before commiting changes, go back to your Jenkins console
* Under the job we created, navigate back to the build section's execute shell. Add ```./script/deploy```.
    * The execute shell should now look like this:
        ```
        npm install
        ./script/test
        ./script/deploy
        ```
* Click "Save".
* Add, commit, and push changes from local node-app repo to GitHub repo

#### Now, try it out! 
You should be able to make changes to your node-app, and see them reflected at the node-app public ip address, only if the changes pass your tests. To test this out, change index.js and test.js (so that they are matching). When you you commit and push changes, Jenkins will run your test and then re-deploy node-app if tests are successful. Read through the jobs/error reports in the Jenkins console.
