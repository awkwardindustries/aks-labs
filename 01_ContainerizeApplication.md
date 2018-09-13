# Lab 01: Containerize Existing Applications

---
## Objective
---

In this lab, you will create two .NET Core applications -- an event processor
that receives messages from an Azure Event Hub, and a reporting front-end
website to display any data received. After these simple applications have
been created, you will containerize them by adding the appropriate Dockerfile.
To complete the lab, you should be able to show that the two applications
are running as containers hosted on your local development system.

This lab expects the following prerequisites have been installed or configured:
* Access to either an Azure Subscription or Resource Group for which you are either
in the Owner or Contributor role
* [.NET Core Software Development Kit (SDK)](https://www.microsoft.com/net/learn/get-started-with-dotnet-tutorial) version 2.1 or greater
  * Installs the `dotnet` command-line interface tools and .NET Core libraries
* [Visual Studio Code](https://code.visualstudio.com/) version 1.25 or greater
* [VS Code C# extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)
  * Provides full support for C# IntelliSense and debugging
* [Azure Command-Line Interface (CLI)](https://docs.microsoft.com/cli/azure/install-azure-cli) version 2.0 or greater
* [Docker for Windows]()
  * Setup to build Linux Containers
  * Ensure daemon is running
* (Optional) [VS Code Docker extension]()
  * Provides Dockerfile and docker-compose intellisense
  * Docker Explorer to show Images, Containers, and Registries

---
## Create the Azure Event Hub Namespace and Event Hub
---

### Log on to Azure

1. Open a terminal and run:

    ```console
    az login
    ```

2. Set the current subscription context. Replace `MyAzureSub` with the name of the
Azure subscription you want to use:

    ```console
    az account set --subscription MyAzureSub
    ```

### Provision resources

The following commands will provision the required Event Hubs resources. You will
create a new resource group to hold all provisioned resources for easy cleanup.
You'll need a storage account to enable the Capture feature on our Event Hub.
Finally, you have to create both the namespace and event hub. 

Any value below prefixed with `my` indicates a value that should be replaced with
an appropriate value specific to you since it may need to be globally unique. Any
values may be replaced as long as the resources are successfully provisioned in
your resource group. Note the `AZURE_STORAGE_CONNECTION_STRING` value should be
replaced with the response from the previous command returning the connection
string for the newly created storage account. 

Replace `myResourceGroup`, `myStorage`, `myProcessorContainer`,
`myNamespace`, and `myEventHub` with the appropriate values:

```console
# Create a resource group
az group create --name myResourceGroup --location westus

# Create a general purpose standard storage account
az storage account create --name myStorage --resource-group myResourceGroup --location westus --sku Standard_RAGRS --encryption blob

# Get and save the connection string for container creation
az storage account show-connection-string --name myStorage --resource-group myResourceGroup

# Create the container for event processing (holds lease and checkpoint data)
az storage container create --name myProcessorContainer --connection-string AZURE_STORAGE_CONNECTION_STRING

# Create an Event Hubs namespace
az eventhubs namespace create --name myNamespace --resource-group myResourceGroup -l westus

# Create an event hub
az eventhubs eventhub create --name myEventHub --resource-group myResourceGroup --namespace-name myNamespace

# Get namespace connection string
az eventhubs namespace authorization-rule keys list --resource-group myResourceGroup --namespace-name myNamespace --name RootManageSharedAccessKey
```
> ***Important:*** Take note of the namespace `primaryConnectionString` as we will
> use this value to connect to the event hub to send and receive events. In a 
> non-lab or development environment, you should create a more specific
> AuthorizationRule rather than use the `RootManageSharedAccessKey`.

### Send events to the event hub

1. Navigate to the tester [Event Generator](https://eventgen.azurewebsites.net/).
2. Enter the saved `primaryConnectionString` and your `myEventHub` name used during
provisioning in the Event Hub Settings section.
3. Leave the last three options blank. This will generate 60 events about every
second for about one minute.
4. Click the Submit Job button to start the event generation.
5. Browse to https://portal.azure.com/. Find your Event Hub by searching for
the name of your Event Hub namespace in the search box, selecting the resource.
6. Explore the metrics to see that messages have hit your event hub. 

---
## Create the Event Processor
---

### Create your .NET Core console application

Navigate to the directory in which you would like to create your event processor
application. Run the following commands to create and run the console application:

```console
dotnet new console -o SongRequest.MessageConsumer

cd SongRequest.MessageConsumer

dotnet run
```

### Add required libraries

Our message consumer will be responsible for connecting to our event hub instance
and consuming messages from the stream. The required libraries are available in
the **Microsoft.Azure.EventHubs** and **Microsoft.Azure.EventHubs.Processor**
NuGet packages. Run the following commands to add them to your project:

```console
dotnet add package Microsoft.Azure.EventHubs

dotnet add package Microsoft.Azure.EventHubs.Processor

dotnet restore
```

### Add an event processor

  1. In Visual Studio Code, open the `SongRequest.MessageConsumer` parent directory (since we'll have more than one service) as your workspace:

      ```console
      cd ..

      code .
      ```

  2. Add another file, **SongRequestProcessor.cs** to the **SongRequest.MessageConsumer**
directory (where the **Program.cs** can be found). This file will implement the
**IEventProcessor** to handle consuming events flowing over the event hub.

  3. Add the following `using` statements:

      ```csharp
      using Microsoft.Azure.EventHubs;
      using Microsoft.Azure.EventHubs.Processor;
      using System;
      using System.Collections.Generic;
      using System.Text;
      using System.Threading.Tasks;
      ```

  4. Implement the `IEventProcessor` interface. Add the following implementation:

      ```csharp
      namespace SongRequest.MessageConsumer
      {
          public class SongRequestProcessor : IEventProcessor
          {
              public Task CloseAsync(PartitionContext context, CloseReason reason)
              {
                  Console.WriteLine($"SongRequestProcessor shutting down. Partition '{context.PartitionId}', Reason: '{reason}'.");
                  return Task.CompletedTask;
              }

              public Task OpenAsync(PartitionContext context)
              {
                  Console.WriteLine($"SongRequestProcessor initialized. Partition '{context.PartitionId}'.");
                  return Task.CompletedTask;
              }

              public Task ProcessErrorAsync(PartitionContext context, Exception error)
              {
                  Console.WriteLine($"Error on Partition '{context.PartitionId}', Error: {error.Message}");
                  return Task.CompletedTask;
              }

              public Task ProcessEventsAsync(PartitionContext context, IEnumerable<EventData> messages)
              {
                  foreach (var eventData in messages)
                  {
                      var data = Encoding.UTF8.GetString(eventData.Body.Array, eventData.Body.Offset, eventData.Body.Count);
                      Console.WriteLine($"Message received. Partition: '{context.PartitionId}', Data: '{data}'");
                  }
                  return context.CheckpointAsync();
              }
          }
      }
      ```

### Edit main console method to use event processor

  1. Add the following `using` statements to the top of the Program.cs file:

      ```csharp
      using Microsoft.Azure.EventHubs;
      using Microsoft.Azure.EventHubs.Processor;
      using System;
      using System.Threading;
      using System.Threading.Tasks;
      ```

  2. Add constants to the `Program` class for the event hub connection string and event
hub name:

      ```csharp
      private const string EventHubConnectionString = "REPLACE_WITH_EVENT_HUB_CONNECTION_STRING";
      private const string EventHubName = "REPLACE_WITH_MY_EVENT_HUB_NAME";
      private const string StorageContainerName = "REPLACE_WITH_MY_PROCESSOR_CONTAINER_NAME";
      private const string StorageConnectionString = "REPLACE_WITH_STORAGE_CONNECTION_STRING";
      ```

  3. Add a new method, `MainAsync` to the `Program` class:

      ```csharp
      private static async Task MainAsync(string[] args) {
          Console.WriteLine("Registering SongRequestProcessor...");

          var eventProcessorHost = new EventProcessorHost(
              EventHubName,
              PartitionReceiver.DefaultConsumerGroupName,
              EventHubConnectionString,
              StorageConnectionString,
              StorageContainerName);
          
          try
          {
              // Register the Event Processor Host and start receiving messages
              await eventProcessorHost.RegisterEventProcessorAsync<SongRequestProcessor>();

              Console.WriteLine("SongRequestProcessor registered.");
              Console.WriteLine("Receiving. Press CTRL+C to stop worker.");

              // Prevents this host process from terminating so services keep running
              Thread.Sleep(Timeout.Infinite);
          }
          catch (Exception ex) 
          {
              Console.WriteLine($"Unexpected exception: {ex.Message}. Stopping processor.");
          }

          // Dispose the Event Processor Host
          await eventProcessorHost.UnregisterEventProcessorAsync();
      }
      ```

  4. Replace the `Main` method implementation with the following line of code:

      ```csharp
      MainAsync(args).GetAwaiter().GetResult();
      ```

### Test implementation

  1. Navigate to the SongRequest.MessageConsumer project root and execute:

      ```console
      dotnet build

      dotnet run
      ```

  2. Verify the application is running as expected.

  3. Resend messages to the event hub instance from the [Event Generator](https://eventgen.azurewebsites.net/) and watch your event processor output.

---
## Containerize the Event Processor
---

### Add a Dockerfile

1. Create a new `Dockerfile` file at the root of the SongRequest.MessageConsumer
project. You'll use the base image `microsoft/dotnet` with two different tags --
one for building and one for executing. Use the following content for the Dockerfile:

    ```dockerfile
    FROM microsoft/dotnet:sdk AS build-env
    WORKDIR /app

    # Copy only the project file and restore as a distict layer of image
    COPY *.csproj ./
    RUN dotnet restore

    # Copy all other files and build as a distinct layer of image
    COPY . ./
    RUN dotnet publish -c Release -o out

    # Build the runtime image
    FROM microsoft/dotnet:runtime
    WORKDIR /app
    COPY --from=build-env /app/out .
    ENTRYPOINT ["dotnet", "SongRequest.MessageConsumer.dll"]
    ```

2. Add a `.dockerignore` file to the project root, and include the following lines
to keep your build context as small as possible (so the files in these directories
are not copied to the image layers):

    ```
    bin\
    obj\
    ```

3. Build and run the Docker image by executing the following from the project root
directory:

    ```console
    docker build -t songrequest.messageconsumer:dev .

    docker run -it --name messageconsumer songrequest.messageconsumer:dev
    ```

4. Exit the container by hitting the `CTRL+C` key combination.

---
## Containerize the Reporting Front End
---

1. Download or Clone the latest from the GitHub repository [awkwardindustries/aks-labs-webfrontendservice](https://github.com/awkwardindustries/aks-labs-webfrontendservice) to your application
root directory. This project contains an ASP.NET Core website using Razor pages.

    > Note: Although not required to build, this project uses the new Microsoft 
    > Library Manager (LibMan) to manage the client-side libraries. A 
    > cross-platform LibMan CLI is available. To install, execute:
    > ```
    > dotnet tool install --global Microsoft.Web.LibraryManager.Cli --version 1.0.163
    > ```
    > Client-side libraries can then be added with commands like `libman install jquery`
    > or restored based on the libman.json file with `libman restore`.

2. Examine the `Dockerfile` to see it is almost the same as that used by our .NET
Core console application project, but it uses the `aspnetcore-runtime` image versus
the `runtime` image since the former includes the required ASP.NET Core libraries.

3. Because this container will host a web server, we'll need to expose a port from
the container to the host. In this example, we'll expose port 80 and also map the
host's port 80 to that exposed port. You could also use `-p 8080:80` to map the
host's port 8080 to the container's port 80, but we'll just use the default. Build
and run the Docker image by executing the following from the project root directory:

    ```console
    docker build -t songrequest.webfrontend:dev .

    docker run -d -p 80:80 --name webfrontend songrequest.webfrontend:dev
    ```

4. Navigate to http://localhost/ to see the website running from within the container.
Although the website should run, there is no backend yet to support it, so any
attempt to request songs or display song requests will result in an expected error.
Once we have hosted the backend API, we will need to update this container's
environment variables by setting the `SONGREQUEST_APISERVICE_BASE_URL` variable to 
the correct path. This can be done with the *docker run* command, i.e.,
`docker run -d -p 80:80 -e SONGREQUEST_APISERVICE_BASE_URL='http://localhost:8000/' songrequest.webfrontend:dev` 

---
## Summary
---

By completing this lab, you've started with two key components of our system --
an event processor to handle song requests streaming across an Event Hub, and a
front end that allows a single request to be created and also shows summary
data of most recent requests and top song requests. You should have both of these
running successfully in their own containers hosted on your local machine.