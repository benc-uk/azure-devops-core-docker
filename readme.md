# Azure DevOps Lab with .NET Core, Docker and VSTS

## Synopsis
#### From first principals and nothing but VS Code, an empty directory and a command terminal we will create a working web application running in Azure, in Linux containers, deployed via an automated DevOps CI/CD pipeline

The scenario will cover:
* .NET Core (ASP MVC webapp)
* Docker & Docker Machine
* Azure
* VSTS

You do not need to be an .NET expert for the coding part but you will need to make basic changes to a C# file, and some HTML. Likewise no prior experience with VSTS and Azure is required (but obviously beneficial). We will also spend some time with the Docker & docker-machine command line tools, but all commands will be supplied. You will be able to complete the lab with either a Windows or Mac machine, but only Windows has been tested.

> Note. The scenario purposely does not use Azure Container Service, for this learning scenario Docker Machine presents a simpler & more lightweight way to get started with Docker running in Azure 

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
:warning: **Do not ignore this part!** :warning:  
You will need the following things set up and installed on your machine: 
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
  * However - you only need two CLI tools for this exercise, so a much more lightweight option is to [download them from this repo](files/Docker 1.13 client only Win x64.zip?raw=true) and place the two executables in your path. Sorry this is for Windows users only!
* Install git; [Git for Windows](https://git-scm.com/download/win) or [Git for Mac](https://git-scm.com/download/mac)
* Optional but strongly recommended: [Git credential manager](https://www.visualstudio.com/en-us/docs/git/set-up-credential-managers)


## Initial Setup Steps
> #### For detailed instructions for these steps with screenshots [click here](setup/)
Overview of steps:
 1. Create a new VSTS account (or new project if you already have an account)
 2. Create a VSTS agent pool: Settings -> Agent queues -> New queue. Call it: ***DockerAgents***
 3. Make a note of your Azure subscription ID
 4. Create a PAT (personal access token) in VSTS. [How to steps](/setup#4-create-a-pat-in-vsts)
 5. Make a note of your VSTS account name, e.g. `{account_name}.visualstudio.com`
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
.UseUrls("http://*:5000")
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


## 4. Add Docker support
We want to add Docker support to the project, and the Docker extension for VS Code makes this super easy. Return to VS Code:
* Hit *Ctrl+Shift+P*, type "docker" then pick "Add docker files to workspace", you will be prompted for two things
  * Choose '.NET Core'
  * Change the port from '3000' to '5000'

This will add a few of files to the project, the key one being `Dockerfile`, open it up, to take a look and we need to make some small changes:
* Change source image tag to version "1.1.1"
  * `FROM microsoft/aspnetcore:1.1.1`
* Change the source variable to a folder called 'pub':
  * `ARG source=.` -> `ARG source=pub`

Your resulting `Dockerfile` should look like this:
```docker
FROM microsoft/aspnetcore:1.1.1
LABEL Name=mywebapp Version=0.0.1 
ARG source=pub
WORKDIR /app
EXPOSE 5000
COPY $source .
ENTRYPOINT dotnet mywebapp.dll
```
> Note 1. The use of pub as the source folder isn't strictly required, but will make more sense later when we set up the CI pipeline in VSTS  
> Note 2. If you are stuck, having problems or want to jump in at this point you can cheat by cloning the following from Github: [ASP.NET Core demo app](https://github.com/benc-uk/dotnet-demoapp)  

---

With our app created we now want to run it in Azure as a Docker container, we have many different ways to achieve this but we'll use Docker Machine. We'll multitask here, building the Docker host takes some time, so we can kick that off and then switch over to VSTS to create the project and connect up our repo with git

## 5. Create Docker host in Azure
Here we'll use Docker Machine to create a running Docker host, this is done with a single `docker-machine create` command. The host will be an Ubuntu 16.04 Linux VM which will have the Docker engine and daemon deployed on it
Most of the command parameters are self explanatory, the open ports are important for the last part of the demo when we want to connect to our app. There's [many other options](https://docs.docker.com/machine/drivers/azure/) but unless you are familiar with Azure run the command as follows: 
```
docker-machine create --driver azure --azure-subscription-id %AZURE_SUB% --azure-resource-group my-docker-resources --azure-location northeurope --azure-open-port 80 dockerhost
```
You will need to manually substitute `%AZURE_SUB%` (unless you have it set as an environmental var). You should have made a note of the Azure subscription ID at the beginning of the scenario.  
This will take about 5-8 minutes to complete, you can watch it deploying in the Azure portal by going into the resource group and taking a look, but don't wait, it's best to switch over to what we need to do in VSTS


## 6. Create and VSTS project and load in code
Create another command/terminal window or use the integrated terminal in VS Code and stage and your code to git locally  
Alternatively if you really command-line phobic use the integrated git support `Ctrl+Shift+G` in VS Code
```
git init
git add .
git commit -m "initial commit"
```

We now want to push our code up into our VSTS repo, use these commands substituting as required:
> Note. You will get the correct URL & syntax for this command by expanding the *"push an existing repository from command line"* section of the project start page or code page
```
git remote add origin https://{vsts_account}.visualstudio.com/_git/{project}
git push -u origin --all
```
If you have the git credential manager installed, authentication should should pop up, so use your VSTS account details.  
If you have trouble you have the option of manually creating git credentials by going into VSTS --> Code --> Generate Git credentials


## 7. Install Docker Integration into VSTS 
Docker support in VSTS is enabled via an extension on the VSTS marketplace 
* Open this link in a new tab: [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-vscs-rm.docker) and click "Install"


## 8. Point Docker client at remote host
Hopefully the `docker-machine create` command we fired off earlier has completed (and successfully!) if not, go grab a quick coffee.  
We need some way to interact with our new Docker host, and Docker machine makes this easy, by quickly setting a bunch of environment variables which "points" your Docker client at the new host. There's no need to connect via SSH or anything messy, simply run:
```
docker-machine env dockerhost
```

This will spit out a bunch of stuff but the last line is what you need to copy & paste and run, it should look like:
```
@FOR /f "tokens=*" %i IN ('docker-machine env dockerhost') DO @%i
```
or on Mac/Linux
```
eval "$(docker-machine env dockerhost)"
```

What has this done? It has set several environmental variables used by the docker client, now if you issue any `docker` command on your machine it will be run on the remote Docker host running in Azure. And this is why we don't need the whole Docker engine installed locally on your machine. Neat!  

Let's run a quick `docker ps` and/or `docker run hello-world` to verify everything is OK. You should get no errors


## 9. Run VSTS agent as Docker container
We'll now spin up a VSTS agent, this will serve two purposes; to do our .NET code compile/publish, but also build our Docker image. We'll run it as a container which then gives us access to the parent Docker host.  
To run the agent as container in the Docker host you just created, run this command:
```bash 
docker run -e VSTS_WORK='/var/vsts/$VSTS_AGENT' -v /var/vsts:/var/vsts -v /var/run/docker.sock:/var/run/docker.sock -e VSTS_ACCOUNT=%VSTS_ACCT% -e VSTS_TOKEN=%VSTS_PAT% -e VSTS_POOL=DockerAgents -d microsoft/vsts-agent:latest
```
You will need to manually substitute `%VSTS_PAT%` and `%VSTS_ACCT%` (unless you have them set as environmental vars). You should have made a note of these at the beginning of the scenario

This might take about a minute to pull the image from Dockerhub and to fire up. You can check it has worked in the VSTS "Agent Queues" view, and check the *DockerAgents* pool/queue, the agent should eventually appear and turn green, the name will be gibberish BTW (if this annoys you can name it with `-e VSTS_AGENT=Foobar`). If it doesn't appear, run `docker ps` and check if the container is running, if not check your command and parameters.


## 10. Create build definition
We're nearly there (I promise! :sweat_smile:), the last major step is to define the build job in VSTS. Unfortunately there's a few steps and they are all manual: 
* In your new VSTS project, go into 'Build & Release' --> 'Builds' --> create new build definition
* Select "ASP.NET Core (PREVIEW)" as the template
* Give it a nice name, as the default is pretty ugly, e.g. "Dotnet CI build for Docker"
* Modify the build as follows:
  * Change the 'Publish' task:
    * Untick both the 'Zip Published Projects' and 'Publish Web Projects' checkboxes
    * Change the 'Arguments' so the last parameter is just `--output pub`
  * Add a new task (place it after the Publish step in the sequence)
    * Search for "Docker"
    * Add the first task in the list (labeled Docker)
    * Change the new Docker task: in the 'Image Name' field and remove the first part of the image tag and hardcode it, something like `mywebapp:$(Build.BuildId)` also tick the 'Include Latest tag' checkbox
 * Add another new task (place it after the Docker task in the sequence)
    * Search for "Copy Files"
    * Add the task in the list labeled "Copy Files"
    * Change the new copy task: Change contents to "Dockerfile" and the target folder to "$(build.artifactstagingdirectory)" (no quotes on either)
  * Click on the "Options" tab and set the default agent queue to *DockerAgents*
  * Click on the "Triggers" tab and turn on 'Continuous Integration'

> #### NOTE: For screenshots of the previous steps, [click here](vsts-build.md)

Click 'Save & Queue' to kick off a manual build, make sure the queue is set to *DockerAgents* then sit back, watch the logs and *Hope It All Works(:tm:)*...  
When the build completes you should have a new Docker image called 'mywebapp' ready for use, you can validate this with a quick `docker images` command


## 11. Release our app
You have two choices at this point, if you're running out of time or tired of fiddling with VSTS, run a quick manual deployment. Otherwise I suggest you press on and complete the VSTS release definition so we have a complete continuous deployment pipeline.

> Note. We use a fixed port for our app (HTTP port 80) which simplifies things, as it means we don't need to inspect our containers to find the dynamic port. However it also means we can only have a single container running.

#### 11.1 Manual Deployment
Return to your terminal and run `docker run -d -p 80:5000 mywebapp` this starts a container running your built .NET core app, and maps port 80 on the host to port 5000 in the container. Now skip to part 12 to view the app.

#### 11.2 Continuous Deployment with VSTS
These steps set up an automated release task in VSTS to run our app as a container each time it is built
 * In your new VSTS project, go into 'Build & Release' --> 'Releases' --> create new definition
 * Choose the 'Empty' option at the bottom of the dialog
 * It should pick up your project and build definition as the source, tick the 'Continuous deployment' checkbox
 * Rename the definition (click the pencil) to something sensible e.g. "Deploy to Docker"
 * Rename the environment if you wish, e.g. "Dev"
 * Click where it says "Run on agent", change the deployment queue to "DockerAgents"
 * Click Add tasks, go into 'Utility' in the catalog and find the 'Command Line' task, add it, then hit close
 * Change the task as follows:
   * Tool: `bash`
   * Arguments: `-c "docker ps --filter \"name=mywebapp\" -q|xargs docker rm -f || true"`
 * Click Add tasks, go into 'All' in the catalog and find the 'Docker' task, add it, then hit close
 * Change the Docker task as follows:
   * Action: Run an image
   * Image Name: `mywebapp:$(Build.BuildId)`
   * Container Name: `mywebapp_$(Release.ReleaseName)`     
   * Ports: `80:5000`

> #### NOTE: For screenshots of the previous steps, [click here](vsts-release.md)

To trigger the pipeline, make a small change to your application code, e.g. change some words in your index.cstml. Them commit your changes to git and push up to VSTS (`git add .` then  `git commit -m "HTML tweak"` then `git push`)
* Back in VSTS you should see your build being triggered and run.  
* Once the build completes you should see the release trigger and the "Deploy to Docker" job running with a release number e.g. "Release-1".  


## 12. View your deployed web app
Final Step... To connect to our running container and web app we'll need the public IP of the Docker host, we can get this from the Azure portal, [click here for steps with screenshots](azure.md).  
Alternatively run `echo %DOCKER_HOST%` and the IP address will be in the resulting URL. Once you have the IP address, open a new browser tab and go to `http://{docker_host_public_ip}` and you should see your web application up and running.  
Gosh wow amazing! etc. :sunglasses: 

---

# Summary
You should now have a containerized .NET Core web application, a Docker host running in Azure, and fully working release pipeline in VSTS. Feel free to experiment from here, some ideas you can look at
* Use dynamic Docker ports to allow multiple containers to be released & running
* Using Azure Container Registry
* Integrating testing to your pipeline with web checks and unit tests
* Deploying to a Docker cluster with Azure Container Service, e.g. Kubernates or Docker Swarm
* Using an Azure Linux Web App

---

# Appendix

## Suggested cleanup & removal tasks
 * Remove the docker-machine config and deployed VM in Azure with `docker-machine rm dockerhost`. Note. Annoyingly this leaves the storage account and resource group in Azure, so go into the portal and tidy up
 * Remove VSTS agent from *DockerAgents* queue
 * Remove the VSTS project
 * Delete the local *mydemoapp* folder


## Run Docker management web UI
To make things a bit more visually exciting you can run a nice little web UI for your Docker host called Portainer, this runs as a container:
```
docker run -d -p 9000 -v "/var/run/docker.sock:/var/run/docker.sock" portainer/portainer 
```
After you've started it, get the dynamic port number and Docker host IP as described above, and connect in your browser
