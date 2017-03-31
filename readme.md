# Azure DevOps Lab with .NET Core, Docker and VSTS

## Synopsis
From first principals and nothing but VS Code, an empty directory and a cmd prompt we will create a working web application running in the cloud, in containers, deployed via an automated DevOps CI/CD pipeline

The demo will cover in scope:
* Azure
* VSTS
* .NET Core (ASP MVC webapp)
* Docker & Docker Machine

You do not need to be an .NET or ASP expert for the coding part but you will need to be unafraid to make basic changes to a C# file, some HTML and project.json. You will need to be familiar with VSTS and Azure. You will also need to be fairly comfortable with the Docker command-line (but all commands will be supplied)

*TODO. Add waffle on why not using ACS or Swarm here xxxxx*

The basic overall flow is:
* Create .NET Core ASP app from template
* Git repo setup
* Minor modifications to code to make it more real
* Creation & modification of Dockerfile
* Creation of VSTS project and code repo
* Push of local git repo into VSTS
* Creation of Docker host in Azure
* Build definition in VSTS
* Deploy VSTS agent as container in new Docker host
* Run build in VSTS
* Create release definition in VSTS 
* Release to a running Docker container with our web app

## Pre-requisites 
Do not ignore this part :) You will need the following things set up and installed on your machine
* [Azure subscription](https://portal.azure.com/)
* [VSTS account](https://my.visualstudio.com/)
* [.NET Core 1.1 SDK](https://www.microsoft.com/net/download/core#/current)
* [VS Code](https://code.visualstudio.com/download)
  * Extension: Docker (Ctrl+P `ext install vscode-docker`)
  * Extension: C# (Ctrl+P `ext install csharp`)
  * Extension: Bootstrap 3 Snippets (Ctrl+P `ext install bootstrap-3-snippets`)
* Docker - Two options:
  * Install the complete [Docker for Windows package](https://docs.docker.com/docker-for-windows/install/)
  * You only need two CLI tools for this demo, so a lightweight option is to [download them from this repo](Docker 1.13 client only Win x64.zip?raw=true) and place the two executables in your path
* [Git for Windows](https://git-scm.com/download/win)
* Git credential manager (Should be installed with git on Windows)

**!IMPORTANT!** Run through the demo end to end on your machine at least once, to ensure you have everything setup. Things like VSTS tokens, git authentication, Azure subscription, blah blah
Besides, there's a **lot** to remember so you want to be familiar with the flow, commands and edits 


## One time initial setup steps
 * Create a VSTS agent pool: Settings -> Agent queues -> Manage pools -> New pool. 
Call it: *DockerAgents*
 * Make a note of your Azure subscription id (see below)
 * Make a note of your VSTS account name, `{acct_name}.visualstudio.com` (see below)
 * Create a VSTS PAT (personal access token) with a 1 year expiry [Details](https://www.visualstudio.com/en-us/docs/setup-admin/team-services/use-personal-access-tokens-to-authenticate)
 * If you've never run git before, run these commands (modifying as required):
 ```
  git config --global user.email "you@example.com"
  git config --global user.name "Your Name"
```

## Create a CMD batch script to store tokens & IDs
This is for convenience and means you can copy & paste the commands in this guide 'as is' without having IDs and tokens hard-coded. Save it somewhere and run it before your demo. For example create `demo-env.cmd` and populate it as follows:

File: `demo-env.cmd`
```bat
SET VSTS_PAT=1234567890_secret_token
SET AZURE_SUB=1234567890_subscription_id
SET VSTS_ACCT=1234567890_account_name
```

***
# Main Demo Flow
What follows is the full step by step guide to the demo 

## 1. Create .NET Core MVC webapp
Open a command prompt and run:
```
mkdir devopsdemo
cd devopsdemo
dotnet new -t web
git init
```
Now open your project folder in VS Code
```
code .
```
Choose 'Yes' to the add assets prompt, and 'Restore' for the dependencies prompt, however neither of these is critical


## 2. Modify code
Edit **Program.cs** and insert the following after the `UseKestrel()` line (note there is note semicolon).  
*Note. This is not important now, but critical later when the app is running inside a container so that the  
webserver is not bound to localhost*
```csharp
.UseUrls("http://*.5000")
```

*If you are feeling extra wimpy skip this step*  
Edit **Views/Home/Index.cshtml** and delete all the contents. If you feeling slighty wimpy put some plain text in here  
"Hello World" or "Demo for blah"  
If you are comfortable with HTML, then put in something like 
```
<div class="jumbotron">
    <div class="container">
        <h1>Hello demo audience!</h1>
        <p>Microsoft &hearts; DevOps and Docker</p>
        <p>
            <a class="btn btn-primary btn-lg">Pretty cool!</a>
        </p>
    </div>
</div>
```
Using the Bootstrap snippet extension for VS Code makes this simple, type `bs3-j` and insert a jumbotron which you can fiddle with and edit.

Lastly we need to make a small tweak to **project.json** (Which if your audience is new to .NET Core it's worth introducing anyhow)  
For reasons best explained by [this blog post](http://www.donovanbrown.com/post/Control-the-name-of-your-NET-Core-output), we need to specify the `outputName` for our build, otherwise the VSTS publish task will create a dll called "s.dll" 
```
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true,
    "preserveCompilationContext": true,
    "outputName": "webapp"
  },
```

## 3. Run the application
*This step is optional.* Demo has been fairly console/editor heavy until now, so let's show something running  
In CMD window
```
dotnet restore
dotnet run
```
Open browser and go to <http://localhost:5000>. Return to the command window and Ctrl-C when you are done


## 4. Add Docker support
We want to add Docker support to the project, the Docker extension for VSTS makes this easy. Return to VS Code:
* Hit *Ctrl+Shift+P*, type 'docker' then pick "Add docker files to workspace"
  * Choose '.NET Core'
  * Choose '5000' for the port

This will add a few of files to the project, the key one being `Dockerfile`, open it up, to take a look and make some small changes:
* Change .NET Core version to v1.1 
  * `FROM microsoft/aspnetcore:1.1.0`
* Change the dll entrypoint (note the dll name matches the outputName we put in `project.json`)
  * `ENTRYPOINT dotnet webapp.dll`
* Change app source to a folder 'pub':
  * `ARG source=.` -> `ARG source=pub`

Your resulting `Dockerfile` should look like this:
```docker
FROM microsoft/aspnetcore:1.1.0
LABEL Name=devopsapp Version=0.0.0 
ARG source=build
WORKDIR /app
EXPOSE 5000
COPY $source .
ENTRYPOINT dotnet demoapp.dll
```

The last two changes are subtle ones, which will only be apparent later when we set up the VSTS build task.  

***

*xxxxxxx TODO. add text on jump in point xxxxxxxxxxxx*

Now it is best to multitask, building the Docker host takes some time, so kick that off and then switch to VSTS to create the project and connect up our repo 


## 5. Create Docker host in Azure
Here we'll use Docker Machine to create a running Docker host, this is done with a single `docker-machine create`command. The host will be an Ubuntu 16.04 Linux VM which will have the Docker engine and daemon deployed on it
Most of the command parameters are self explanatory, the open ports are important for the last part of the demo when we want to connect to our app. There's a [tonne of other options](https://docs.docker.com/machine/drivers/azure/) you can add/modify should you wish. 
```
docker-machine create --driver azure --azure-subscription-id %AZURE_SUB% --azure-resource-group "Demo.Docker" --azure-location "West Europe" --azure-open-port 32768-32900 dockerhost
```
This will take about 5-8 minutes to complete, you can show it kicking off in the Azure portal but then switch over to VSTS


## 6. Create and VSTS project and load in code
Over in VSTS:
* Create a new project, call it what you like but choose git for version control 
* Now jump back to the command prompt and stage and your code to git locally
```
git add .
git commit -m "initial commit"
```
Then and push into the new repo in VSTS. You will get the correct URL & syntax for the 3rd command by expanding the  
*"or push an existing repository from command line"* section of the project getting started page
```
git remote add origin https://{vsts_acct}.visualstudio.com/_git/{project}
git push -u origin --all
```
Authentication **should** should pop up or 'just work', if you have git credential manager installed, if you have trouble you have the option of creating git credentials on the project screen


## 7. Install Docker Integration into VSTS 
* [Goto here to the Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.docker&targetId=c13c7b99-2463-4cd0-84a0-5260108a913e) - click "Install"


## 8. Point Docker client at remote host
Once the `docker-machine create` command (step 5) has completed (hopefully successfully) we need some way to interact with it. Docker machine makes this easy, by quickly setting a bunch of environment variables which "points" your docker client at the new host. There's no need to connect via SSH or anything messy, simply run:
```dos
docker-machine env dockerhost
```
This will spit out a bunch of stuff but the last line after the REM is what you need to copy & paste and run, it should look exactly like:
```
@FOR /f "tokens=*" %i IN ('docker-machine env dockerhost') DO @%i
```
What has this done? Well now if you issue any `docker` command on your machine it will be run on the remote Docker host running in Azure. Pretty cool. Run a quick `docker ps` or `docker run hello-world` to test everything is OK. 


## 9. Run VSTS agent as Docker container
We'll now spin up a VSTS agent, this will serve two purposes; to do our .NET code compile/publish, but also build our Docker image. We'll run it as a container which then gives us access to the parent Docker host.  
To run the agent as container in the Docker host you just created, run this command:
```bash 
docker run -e VSTS_WORK='/var/vsts/$VSTS_AGENT' -v /var/vsts:/var/vsts -v /var/run/docker.sock:/var/run/docker.sock -e VSTS_ACCOUNT=%VSTS_ACCT% -e VSTS_TOKEN=%VSTS_PAT% -e VSTS_POOL=DockerAgents -d microsoft/vsts-agent:latest
```
You need your VSTS PAT and VSTS account name. Note the `VSTS_POOL` should match the pool name you created pre-demo.

This might take about a minute to pull the image from Dockerhub and to fire up. You can check it has worked in the VSTS "Agent Queues" view, and check the *DockerAgents* pool/queue, the agent should eventually appear and turn green, the name will be gibberish BTW (if this annoys you can name it with `-e VSTS_AGENT=Foobar`)


## 10. Create build definition
We're nearly there, the last major step is to define the build job in VSTS. There's a few steps: 
* In your new VSTS project, go into 'Build & Release' and create new build definition
* Select "ASP.NET Core (PREVIEW)" as the template
* Give it a nice name, as the default is pretty ugly, e.g. "Dotnet CI build for Docker"
* Modify the build as follows:
  * Remove or disable the 'Publish Artifact' task  
    (The Docker image we create is in effect our release artifact, no need for VSTS server drops)
  * Remove or disable the 'Test' task
  * Change the 'Publish' task:
    * Untick both "Publish Web Projects" & "Zip Published Projects" options
    * Change the "Arguments" so the last parameter is `--output pub`
  * Add a new task (place it last in the sequence)
    * Search for "Docker"
    * Add the first task in the list (labeled Docker)
  * Change the new Docker task - in the 'Image Name' field and remove the first part of the image tag and hardcode it, something like `demoapp:$(Build.BuildId)` ALSO add `latest` to the additional tags
  * Click on the "Options" tab and set the default agent queue to *DockerAgents*

Click 'Save & Queue' and kick off a build, ensure the queue is set to *DockerAgents* then make a silent prayer to the Demo Gods(TM) and kick the build it off...  
When the build completes you should have a new Docker image called 'demoapp' ready for use, validate this with a quick `docker images` command

## 11. Release our app
Two choices at this point, quick a dirty is to run `docker run -d -p 5000 demoapp:latest` this starts a container running your compiled and built .NET core app.  


In order to connect to the app you will need to get the dynamic port number, to find this run `docker ps` and make a note of the port in the section looking like `0.0.0.0:xxxxx->5000/tcp`. If this is the first time you've started it the port is likely to be `32768` but it increases by one each time.

To connect we'll also need the public IP of the Docker host, we can get this from the Azure portal or by running `echo %DOCKER_HOST%`

# Appendix

## Cleanup tasks
 * Remove the docker-machine config and deployed VM in Azure with `docker-machine rm dockerhost`. Note. Annoyingly this leaves the storage account and resource group in Azure, so go into the portal and tidy up
 * Remove VSTS agent from *DockerAgents* queue
 * Remove the VSTS project
 * Delete local *devopsdemo* folder

## Run Docker management web UI
To make things a bit more visually exciting you can run a nice little web UI for your Docker host called Portainer, this runs as a container:
```
docker run -d -p 9000 -v "/var/run/docker.sock:/var/run/docker.sock" portainer/portainer 
```
After you've started it, get the dynamic port number as described above, and connect in your browser
