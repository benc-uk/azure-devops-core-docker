## Optional: Create a batch script to store tokens & IDs
This is for convenience and means you can copy & paste the commands in this guide 'as is' without having IDs and tokens hard-coded. Save it somewhere and run it before you carry on. For example create `demo-env.cmd` and populate it with the relevant information from the initial set-up steps

File: `demo-env.cmd` *(On Mac OSX: change SET to EXPORT and extension to .sh)*
```bat
SET VSTS_PAT=1234567890_secret_token
SET AZURE_SUB=1234567890_subscription_id
SET VSTS_ACCT=1234567890_account_name
```


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



Return to your terminal and run `docker run -d -p 80:5000 mywebapp` this starts a container running your compiled and built .NET core app. In order to connect to the app you will need to get the dynamic port number, to find this run `docker ps` and look at the container at the top of the list, make a note of the port in the section looking like `0.0.0.0:xxxxx->5000/tcp`. If this is the first time you've started it the port is likely to be 32768 but it increases by one each time. Now skip to part 12 to view the app.