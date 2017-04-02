# Azure DevOps Lab with .NET Core, Docker and VSTS

## Synopsis
#### From first principals and nothing but VS Code, an empty directory and a cmd prompt we will create a working web application running in Azure, in Linux containers, deployed via an automated DevOps CI/CD pipeline

The scenario will cover:
* Azure
* VSTS
* .NET Core (ASP MVC webapp)
* Docker & Docker Machine

You do not need to be an .NET or ASP expert for the coding part but you will need to be unafraid to make basic changes to a C# file, and some HTML. Likewise no prior experience with VSTS and Azure is required (but beneficial). We will also spend some time with the Docker & docker-machine command line clients (but all commands will be supplied). You will be able to complete the lab with either a Windows or Mac machine, but only Windows has been tested.

Note. The scenario purposely does not use Azure Container Service, for this learning scenario Docker Machine presents a simpler & more lightweight way to get started with Docker running in Azure 

The basic overall flow is:
* Create .NET Core ASP app from template
* Git repo setup
* Minor modifications to the HTML to suit your taste :)
* Creation & modification of a Dockerfile
* Creation of VSTS project and code repo
* Push of git repo into VSTS
* Creation of Docker host in Azure
* Build definition in VSTS
* Deploy VSTS agent as container in new Docker host
* Run build in VSTS
* Create release definition in VSTS 
* Resulting in a running Docker container with our web app

---

## Pre-requisites 
Do not ignore this part! You will need the following things set up and installed on your machine: 
* An active [Azure subscription](https://portal.azure.com/). If you do not have a subscription:
  * You may have been given an [Azure Pass](https://www.microsoftazurepass.com/) card & code, please follow the steps given to activate your new subscription.
  * OR - create a [free Azure account and subscription](https://azure.microsoft.com/en-gb/free/)
* An active [VSTS Account](https://app.vsaex.visualstudio.com/)
  * If you don't have an account, [create a free VSTS account](https://www.visualstudio.com/en-gb/docs/setup-admin/team-services/sign-up-for-visual-studio-team-services)
* Install the [.NET Core 1.1 SDK](https://www.microsoft.com/net/download/core#/current)
* Install [VS Code](https://code.visualstudio.com/download)
  * Required extension: Docker (Ctrl+P `ext install vscode-docker`)
  * Optional extension: C# (Ctrl+P `ext install csharp`)
  * Optional extension: Bootstrap 3 Snippets (Ctrl+P `ext install bootstrap-3-snippets`)
* Docker, two options:
  * Install the complete [Docker for Windows package](https://docs.docker.com/docker-for-windows/install/) or [Docker for Mac package](https://docs.docker.com/docker-for-mac/install/)
  * You only need two CLI tools for this exercise, so a much more lightweight option is to [download them from this repo](Docker 1.13 client only Win x64.zip?raw=true) and place the two executables in your path. Sorry this is for Windows users only!
* Install git; [Git for Windows](https://git-scm.com/download/win) or [Git for Mac](https://git-scm.com/download/mac)
* Optional but strongly recommended: [Git credential manager](https://www.visualstudio.com/en-us/docs/git/set-up-credential-managers)


## Initial Setup Steps
#### For detailed instructions for these steps with screenshots [click here](setup/)
Overview of steps:
 1. Create a new VSTS account (or new project if you already have an account)
 2. Create a VSTS agent pool: Settings -> Agent queues -> New queue. Call it: ***DockerAgents***
 3. Make a note of your Azure subscription ID
 4. Create a PAT (personal access token) in VSTS. [How to steps](/setup#4-create-a-pat-in-vsts)
 5. Make a note of your VSTS account name, it's in the URL e.g. `{account_name}.visualstudio.com`
 6. If you've never run git before, run these commands (modifying with your details as required):
 ```
git config --global user.email "your-email@example.com"
git config --global user.name "Your Name"
git config --global credential.helper manager
```
It is recommended you paste the VSTS account name, PAT token and Azure subscription ID to a scratchpad file somewhere. Another suggestion is to create a small batch file which sets these as environmental variables, this will let you copy/paste and run as-is some of the lengthier commands later on, without changes. Some sample files are provided in the file section to get you started

---

# Main Exercise Flow
With all the setup complete, what follows is the full step by step guide to the exercise 

## 1. Create .NET Core MVC webapp
First of all we'll create our .NET Core application project and source code. The .NET Core SDK uses the Yeoman templating system and comes with several built-in templates to get you started quickly. Open a command prompt or terminal and run the following commands:
```
mkdir mywebapp
cd mywebapp
dotnet new mvc
```
If this is the first time running the dotnet command it will take a minute to decompress and cache some stuff.  
> Note. You can call the folder anything you like, but if you change it, the Dockerfile we create later needs to reflect that change

Now open your project folder in VS Code
```
code .
```
Take a look around, you'll see we have a fully functional ASP.NET Core MVC application with views, controllers and static HTML/CSS content all created for us.


## 2. Modify application webserver behavior 
Open **Program.cs** and two prompts will appear, neither are critical but click 'Yes' to the add assets prompt, and 'Close' for the dependencies prompt  
Insert the following after the `UseKestrel()` line (note there is no semicolon!) :
```csharp
.UseUrls("http://*.5000")
```
.NET Core doesn't require IIS and comes with a built-in webserver called Kestrel, by default Kestrel binds to the loopback adapter and we want it to listen on all IPs. Non-coders don't panic, this is the only real code change we need to make.
> Note. This change is not important now, but _critical_ later when the app is running inside a Docker container


## 3. Run and tailor your application
Let's take a look at our app and make sure it runs. In VS Code hit `Ctrl+'` or switch back to your cmd/terminal window, then run: 
```
dotnet restore
dotnet run
```
.NET Core is much more lightweight than old legacy .NET, the `dotnet restore` command pulls down the packages it requires to run our app and nothing more. If you have worked with Node.js it is similar to the "npm install" step. 

Open a browser and go to [`http://localhost:5000`](http://localhost:5000) to see your app. You should see the standard generated website that the SDK has created. It's not very interesting, so let's make it more personal...

Return to VS Code and open **Views/Home/Index.cshtml** and delete all the existing contents of the file.  
If you are comfortable with HTML, then put in something like the snippet below. Using the Bootstrap snippet extension for VS Code makes this simple, just type `bs3-j` and Intellisense will kick in and lets you insert a Bootstrap jumbotron which you can fiddle with and edit.
```
<div class="jumbotron">
    <div class="container">
        <h1>My Azure Web App</h1>
        <p>Microsoft &hearts; DevOps and Docker</p>
        <p>
            <a class="btn btn-primary btn-lg">Pretty cool!</a>
        </p>
    </div>
</div>
```

> Note. If you feeling slightly wimpy just put some plain text in, e.g. "Hello this is my web app". If you are feeling extra wimpy you can skip this step altogether. 
Hit save and return to your browser and hit refresh to automatically see your changes. ASP.NET Core uses a view template system called Razor which doesn't require recompilation to update. Neat!

Return to where you ran `dotnet run` and hit `Ctrl+C` when you are done


## 5. Add Docker support
We want to add Docker support to the project, the Docker extension for VSTS makes this easy. Return to VS Code:
* Hit *Ctrl+Shift+P*, type 'docker' then pick "Add docker files to workspace"
  * Choose '.NET Core'
  * Choose '5000' for the port

This will add a few of files to the project, the key one being `Dockerfile`, open it up, to take a look and make some small changes:
* Change source image tag to version "1.1.1"
  * `FROM microsoft/aspnetcore:1.1.1`
* Change the dll entrypoint (note the dll name matches the folder name for your project)
  * `ENTRYPOINT dotnet mywebapp.dll`
* Change the source variable to a folder called 'pub':
  * `ARG source=.` -> `ARG source=pub`

Your resulting `Dockerfile` should look like this:
```docker
FROM microsoft/aspnetcore:1.1.1
LABEL Name=arse Version=0.0.1 
ARG source=pub
WORKDIR /app
EXPOSE 5000
COPY $source .
ENTRYPOINT dotnet mywebapp.dll
```

The last two changes are subtle ones, which will only be apparent later when we set up the VSTS build task.  

***

*xxxxxxx TODO. add text on jump in point xxxxxxxxxxxx*

Now it is best to multitask, building the Docker host takes some time, so kick that off and then switch to VSTS to create the project and connect up our repo 


## 6. Create Docker host in Azure
Here we'll use Docker Machine to create a running Docker host, this is done with a single `docker-machine create` command. The host will be an Ubuntu 16.04 Linux VM which will have the Docker engine and daemon deployed on it
Most of the command parameters are self explanatory, the open ports are important for the last part of the demo when we want to connect to our app. There's a [tonne of other options](https://docs.docker.com/machine/drivers/azure/) you can add/modify should you wish. 
```
docker-machine create --driver azure --azure-subscription-id %AZURE_SUB% --azure-resource-group my-docker-resources --azure-location northeurope --azure-open-port 32768-32900 dockerhost
```
This will take about 5-8 minutes to complete, you can show it kicking off in the Azure portal but then switch over to VSTS


## 7. Create and VSTS project and load in code
Over in VSTS:
* Create a new project, call it what you like but choose git for version control 
* Now jump back to the command prompt and stage and your code to git locally
```
git init
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


## 8. Install Docker Integration into VSTS 
* [Goto here to the Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.docker&targetId=c13c7b99-2463-4cd0-84a0-5260108a913e) - click "Install"


## 9. Point Docker client at remote host
Once the `docker-machine create` command (step 5) has completed (hopefully successfully) we need some way to interact with it. Docker machine makes this easy, by quickly setting a bunch of environment variables which "points" your docker client at the new host. There's no need to connect via SSH or anything messy, simply run:
```dos
docker-machine env dockerhost
```
This will spit out a bunch of stuff but the last line after the REM is what you need to copy & paste and run, it should look exactly like:
```
@FOR /f "tokens=*" %i IN ('docker-machine env dockerhost') DO @%i
```
What has this done? Well now if you issue any `docker` command on your machine it will be run on the remote Docker host running in Azure. Pretty cool. Run a quick `docker ps` or `docker run hello-world` to test everything is OK. 


## 10. Run VSTS agent as Docker container
We'll now spin up a VSTS agent, this will serve two purposes; to do our .NET code compile/publish, but also build our Docker image. We'll run it as a container which then gives us access to the parent Docker host.  
To run the agent as container in the Docker host you just created, run this command:
```bash 
docker run -e VSTS_WORK='/var/vsts/$VSTS_AGENT' -v /var/vsts:/var/vsts -v /var/run/docker.sock:/var/run/docker.sock -e VSTS_ACCOUNT=%VSTS_ACCT% -e VSTS_TOKEN=%VSTS_PAT% -e VSTS_POOL=DockerAgents -d microsoft/vsts-agent:latest
```
You need your VSTS PAT and VSTS account name. Note the `VSTS_POOL` should match the pool name you created pre-demo.

This might take about a minute to pull the image from Dockerhub and to fire up. You can check it has worked in the VSTS "Agent Queues" view, and check the *DockerAgents* pool/queue, the agent should eventually appear and turn green, the name will be gibberish BTW (if this annoys you can name it with `-e VSTS_AGENT=Foobar`)


## 11. Create build definition
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

## 12. Release our app
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
