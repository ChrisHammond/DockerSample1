# Creating an ASP.NET Core MVC App in a Docker Image and run it on Azure

## Introduction
This repo contains instructions for how to create a new ASP.NET Core MVC Application, add it to a Docker Image, deploy that container to Azure and start a container using the image. The assumption is that you're performing this tutorial in a Windows 10 environment. You can/will incure Azure costs in doing this, be aware, and control your spend. Clean up after you're done.

If you have any feedback on this tutorial, feel free to create an issue for discussion, or submit a PR for any fixes/enhancements.

## Pre-requisites:
In order to successfully complete this tutorial you'll want to have some tools installed.

* VS Code (https://code.visualstudio.com/download)
* Azure CLI (https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* .Net Core CLI SDK  (https://dotnet.microsoft.com/download/dotnet-core/3.1)
* C# Extension in VS Code (https://github.com/OmniSharp/omnisharp-vscode)
* Azure Account (https://portal.azure.com)
* Docker for Windows (https://docs.docker.com/docker-for-windows/)
* For this sample I did this with Linux Containers, not Windows Containers, as I ran into a problem trying to deploy a container created with Windows 10 into Azure.

**Open up VS Code in your Windows Environment.**

**Create a new FILE in VS Code and copy/paste the contents of this README.md there so you can follow along (search and replace text will make this much easier on you)**

## Variables to Replace
Below you will find a List of variables to replace with your names/values. To make this easier, you might simply want to search/replace these right here right now. Choose a <variable>, press Control+H type in your replacement value and click replace all.

* <Mvc_Name> # MVC project name, will be the name of the DLL ex: NetCore.Docker
* <image_name> # Name for docker image, lowercase 
* <docker_container_name> # Name for docker container
* <acr_name_here> # Name for Azure Container Registry
* <demo_group_name> # Azure Resource Group Name 
* <dns_name_for_sample_site> # This will be how you access your site once running in Azure on a container. It will need to be unique across azure https://<dns_name_for_sample_site>.azurecontaiers.io 



**Open Terminal in VS Code (Press Control+`)**

**Navigate to your project location (your choice of where)**

```console
cd f:\projects\
```

**Create new directory**

```console
mkdir demo
```

**Change to that directory**

```console
cd demo
````

**Create a new .Net Core MVC web app called <Mvc_Name>, for giggles we will put it into a folder called App**

```console
dotnet new mvc -o App -n <Mvc_Name>
```

**Navigate to the App folder**

```console
cd app
```

**Run the new MVC app**

```console
dotnet run
```

**Browse to https://localhost:5001**

**Create DockerFile manifest** 

```console
New-Item DockerFile -ItemType file
```
 
**Open DockerFile in VS Code, add the following to the contents.**

```FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/core/aspnet:3.1
WORKDIR /app
COPY --from=build-env /app/out .

ENTRYPOINT ["dotnet", "<Mvc_Name>.dll"]
```


**Build the docker image**

```console
docker build -t <image_name> -f DockerFile .
```

**Run the app in a container**

```console
docker run -d -p 8080:80 --name <docker_container_name> <image_name>
```

**Browse to: http://localhost:8080/**

**Create a resource group in Azure, this will be created in the eastus location, if you change eastus to another location you'll need to change it in the URL towards the end of this file as well.**

```console
az group create --name <demo_group_name> --location eastus
```

**Create an Azure Container Registrty where we will load the image**

```console
az acr create --resource-group <demo_group_name> --name <acr_name_here> --sku Basic
```

**Login to the acr**

```console
az acr login --name <acr_name_here>
```

**Get the login server name so we can tag the image properly**

```console
az acr show --name <acr_name_here> --query loginServer --output table
```

**Copy the result from the above command. Replace all references to <login_server_name> with that value.**

**Display a list of your local images**

```console
docker images
```

**Tag the image we created above, it should be tagged with <login_server_name>/<image_name>**

```console
docker tag <image_name> <login_server_name>/<image_name>:v1
```

**Display local Docker images again**

```console
docker images
```

**Push the Docker image to azure**

```console
docker push <login_server_name>/<image_name>:v1
```


**Retrieve the list from registry**

```console
az acr repository list --name <acr_name_here> --output table
```

**Enable the admin account within the Registry**

```console
az acr update -n <acr_name_here> --admin-enabled true
```

**Login to the Registry with the Admin account**

```console
az acr credential show -n <acr_name_here> --query passwords[0].value --output tsv | docker login <acr_name_here>.azurecr.io -u <acr_name_here> --password-stdin
```

**Retrieve the password registry password for the admin account, copy and replace the string <registry_password> with the results of the command below**

```console
az acr credential show -n <acr_name_here> --query passwords[0].value --output tsv
```


**Deploy the container**

```console
az container create --resource-group <demo_group_name> --name <image_name> --image <login_server_name>/<image_name>:v1 --cpu 1 --memory 1 --registry-login-server <login_server_name> --registry-username <acr_name_here> --registry-password <registry_password> --dns-name-label <dns_name_for_sample_site> --ports 80
```

**Browse to the new site**

```console
http://<dns_name_for_sample_site>.eastus.azurecontainer.io
```

**When you're done, you can purge all this work from Azure by deleting the resource group. BE SURE YOU DON'T HAVE RESOURCES IN THERE YOU WANT TO SAVE**

```console
az group delete --name <demo_group_name>
```





Tutorial customized and tweaked, originally based off of [Deploy ASP.NET Core Applications to Azure Container Instances](https://github.com/dotnet/dotnet-docker/blob/master/samples/deploy-container-to-aci.md)