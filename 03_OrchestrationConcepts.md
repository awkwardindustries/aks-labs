# Lab 03: Orchestration Concepts

---
## Objective
---

In this lab, we'll look at scaling both our cluster and a specific deployment
to our cluster. You'll also add an additional service to our application.

---
## Scale Cluster
---

Before we add an additional service, let's scale our cluster from one to three
nodes. This operation will take some time as it spins up new Virtual Machines
with Managed Disks under the covers. Execute the following after replacing
`myAKSCluster` and `myResourceGroup` with the appropriate values:

```console
az aks scale --name myAKSCluster --resource-group myResourceGroup --node-count 3
```

Once you see the *Running ..* feedback, you may continue with the next section
while this operation executes. When it is finished, you'll see a JSON-formatted
response describing the current state of your cluster.

---
## Add Event Generator
---

### Create your .NET Core console application

Navigate to the directory in which you would like to create your event generator
application. Run the following commands to create and run the console application:

```console
dotnet new console -o SongRequest.MessageProducer

cd SongRequest.MessageProducer

dotnet run
```

### Add required libraries

Our message consumer will be responsible for connecting to our event hub instance
and sending messages. The required library is available in
the **Microsoft.Azure.EventHubs** NuGet package. We'll also use a CSV Helper
library to assist with reading/writing from a file containing a list of
song requests over which we'll randomly select for requests. Run the following 
commands to add the libraries to your project:

```console
dotnet add package Microsoft.Azure.EventHubs

dotnet add package CsvHelper

dotnet restore
```

### Add an EventGenerator.cs

  1. In Visual Studio Code, open the `SongRequest.MessageProducer` parent directory as your workspace:

      ```console
      cd ..

      code .
      ```

  2. Add another file, **EventGenerator.cs** to the **SongRequest.MessageProducer**
directory (where the **Program.cs** can be found). This file will handle sending
messages to the event hub.

  3. Add the following `using` statements:

      ```csharp
      using System;
      using System.Collections.Generic;
      using System.IO;
      using System.Linq;
      using System.Text;
      using System.Threading.Tasks;
      using CsvHelper;
      using Microsoft.Azure.EventHubs;
      using Newtonsoft.Json;
      ```

  4. Implement the EventGenerator class. Add the following implementation:

      ```csharp
      namespace SongRequest.MessageProducer
      {
          public class EventGenerator
          {
              private EventHubClient eventHubClient;
              private IList<Song> songs;
              private Random random = new Random();

              public EventGenerator()
              {
              }

              public async Task Generate(EventGeneratorArguments args)
              {
                  // Our connection string is to the namespace, so we'll use this builder to ensure we're
                  // connecting and publishing to our specific event hub.
                  var connectionStringBuilder = new EventHubsConnectionStringBuilder(args.EventHubConnectionString)
                  {
                      EntityPath = args.EventHubName
                  };

                  eventHubClient = EventHubClient.CreateFromConnectionString(connectionStringBuilder.ToString());
                  var pathToSongs = Path.Combine(Path.GetDirectoryName(System.Reflection.Assembly.GetEntryAssembly().Location), "songs.csv");
                  var csvReader = new CsvReader(File.OpenText(pathToSongs));
                  songs = csvReader.GetRecords<Song>().ToList();

                  Console.WriteLine($"Sending to {args.EventHubName} at {args.EventHubConnectionString}");
                  Console.WriteLine($"  {args.NumberOfEventsPerBatch} messages per batch for {args.NumberOfBatchesToRun} batches with delay of {args.DelayInMillisBetweenBatch} milliseconds between batches");

                  await SendMessagesToEventHub(args.NumberOfEventsPerBatch, args.NumberOfBatchesToRun, args.DelayInMillisBetweenBatch);

                  await eventHubClient.CloseAsync();

                  Console.WriteLine("");
                  Console.WriteLine("Finished publish. Exiting.");
              }

              private async Task SendMessagesToEventHub(int numberOfEventsPerBatch, int numberOfBatches, int delayBetweenBatches)
              {
                  int batchesLeftToSend = numberOfBatches;
                  bool sendForever = (batchesLeftToSend == -1);

                  Console.Write("Publishing ");

                  while (batchesLeftToSend > 0 || sendForever)
                  {
                      batchesLeftToSend--;
                      var batch = GenerateBatch(numberOfEventsPerBatch);
                      await eventHubClient.SendAsync(batch);

                      Console.Write(".");

                      await Task.Delay(delayBetweenBatches);
                  }
              }

              private EventDataBatch GenerateBatch(int numberToGenerate)
              {
                  var batch = eventHubClient.CreateBatch();
                  for (int i = 0; i < numberToGenerate; i++)
                  {
                      int randomIndex = random.Next(songs.Count);
                      var randomSong = songs[randomIndex];
                      var eventBytes = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(randomSong));
                      batch.TryAdd(new EventData(eventBytes));
                  }

                  return batch;
              }
          }
      }
      ```

### Add an EventGeneratorArguments.cs

Add a new file to the project called `EventGeneratorArguments.cs` and implement
as follows:

```csharp
namespace SongRequest.MessageProducer
{
    public class EventGeneratorArguments
    {
        public string EventHubConnectionString { get; set; }
        public string EventHubName { get; set; }
        public int NumberOfEventsPerBatch { get; set; }
        public int NumberOfBatchesToRun { get; set; }
        public int DelayInMillisBetweenBatch { get; set; }

    }
}
```

### Add a Song.cs

Add a new file to the project called `Song.cs` and implement as follows:

```csharp
namespace SongRequest.MessageProducer
{
    public class Song
    {
        public string Title { get; set; }
        public string Artist { get; set; }
        public string Genre { get; set; }
    }
}
```

### Add a songs.csv

Add a new file to the project called `songs.csv` and copy the following content:

```csv
Artist,Title,Genre
"Drake","In My Feelings","HipHop"
"Maroon 5","Girls Like You","Pop"
"Cardi B","I Like It","HipHop"
"6ix9ine","FEFE","Pop"
"Post Malone","Better Now","Rap"
"Eminem","Lucky You","Rap"
"Juice WRLD","Lucid Dreams","Rap"
"Eminem","The Ringer","Rap"
"Travis Scott","Sicko Mode","HipHop"
"Tyga","Taste","HipHop"
"Khalid & Normani","Love Lies","HipHop"
"5 Seconds Of Summer","Youngblood","Pop"
"Ella Mai","Boo'd Up","HipHop"
"Ariana Grande","God Is A Woman","Pop"
"Imagine Dragons","Natural","Rock"
"Ed Sheeran","Perfect","Pop"
"Taylor Swift","Delicate","Pop"
"Florida Georgia Line","Simple","Country"
"Luke Bryan","Sunrise, Sunburn, Sunset","Country"
"Jason Aldean","Drowns The Whiskey","Country"
"Childish Gambino","Feels Like Summer","HipHop"
"Weezer","Africa","Rock"
"Panic! At The Disco","High Hopes","Rock"
"Eric Church","Desperate Man","Country"
"Nicki Minaj","Barbie Dreams","Rap"
```

Edit the `SongRequest.MessageProducer.csproj` file to ensure the songs.csv
file is always copied on build. Add the following before the final
closing `</Project>` tag:

```xml
<ItemGroup>
  <None Include="songs.csv" CopyToOutputDirectory="Always" />
</ItemGroup>
```

### Edit main console method to use the event generator

  1. Add the following `using` statements to the top of the Program.cs file:

      ```csharp
      using System;
      using System.Threading.Tasks;
      ```

  2. Add constants to the `Program` class for the event hub connection string and event
hub name and helpers for the command line feedback:

      ```csharp
      private const string EventHubConnectionString = "REPLACE_WITH_EVENT_HUB_CONNECTION_STRING";
      private const string EventHubName = "REPLACE_WITH_MY_EVENT_HUB_NAME";

      private const string EventHubNameOption = "--eh-name";
      private const string EventHubConnectionStringOption = "--eh-cs";
      private const string NumberEventsPerBatchOption = "--num-events";
      private const string NumberOfBatchesOption = "--run-for";
      private const string DelayInMillisBetweenBatchOption = "--delay";

      private static EventGenerator eventGenerator = new EventGenerator();
      ```

  3. Replace the existing `Main` implementation with the following new
  methods:
      ```csharp
      static void Main(string[] args)
      {
          var processedArgs = ProcessArgs(args);
          if (processedArgs == null)
          {
              PrintHelp();
          }
          else
          {
              MainAsync(processedArgs).GetAwaiter().GetResult();
          }
      }

      private static EventGeneratorArguments ProcessArgs(string[] args)
      {
          try
          {
              EventGeneratorArguments processedArgs = new EventGeneratorArguments();
              // Set defaults
              processedArgs.EventHubConnectionString = EventHubConnectionString;
              processedArgs.EventHubName = EventHubName;
              processedArgs.NumberOfEventsPerBatch = 1;
              processedArgs.NumberOfBatchesToRun = 60;
              processedArgs.DelayInMillisBetweenBatch = 1000;

              // Treat args as a pair
              for (int index = 0; index < args.Length; index = index + 2)
              {
                  if (EventHubNameOption.Equals(args[index]))
                  {
                      processedArgs.EventHubName = args[index + 1];
                  }
                  else if (EventHubConnectionStringOption.Equals(args[index]))
                  {
                      processedArgs.EventHubConnectionString = args[index + 1];
                  }
                  else if (NumberEventsPerBatchOption.Equals(args[index]))
                  {
                      processedArgs.NumberOfEventsPerBatch = int.Parse(args[index + 1]);
                  }
                  else if (NumberOfBatchesOption.Equals(args[index]))
                  {
                      processedArgs.NumberOfBatchesToRun = int.Parse(args[index + 1]);
                  }
                  else if (DelayInMillisBetweenBatchOption.Equals(args[index]))
                  {
                      processedArgs.DelayInMillisBetweenBatch = int.Parse(args[index + 1]);
                  }
                  else
                  {
                      // There was likely a problem. Exit.
                      return null;
                  }
              }

              return processedArgs;
          }
          catch (Exception ex)
          {
              Console.Error.WriteLine($"Unexpected error: {ex.Message}");
          }

          return null;
      }

      private static void PrintHelp()
      {
          Console.WriteLine();
          Console.WriteLine($"Usage: dotnet [path-to-application] [options]");
          Console.WriteLine($"Usage: dotnet [path-to-application] {EventHubNameOption} songrequests {EventHubConnectionStringOption} Endpoint=sb://EH_NAMESPACE.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=REPLACE_WITH_KEY");
          Console.WriteLine();
          Console.WriteLine("options:");
          Console.WriteLine($"  {EventHubNameOption}\tThe entity name of the event hub");
          Console.WriteLine($"  {EventHubConnectionStringOption}\tThe connection string to the event hub namespace. Sender rights required.");
          Console.WriteLine($"  {NumberEventsPerBatchOption}\t(Optional: default 1) The number of events to generate per batch.");
          Console.WriteLine($"  {NumberOfBatchesOption}\t(Optional: default 60) The number of batches the processor will run. -1 to run without limit.");
          Console.WriteLine($"  {DelayInMillisBetweenBatchOption}\t(Optional: default 1000) The delay in milliseconds between batches.");
          Console.WriteLine();
          Console.WriteLine("path-to-application:");
          Console.WriteLine("  The path to the SongRequest.MessageProducer.dll file to execute.");
      }

      private static async Task MainAsync(EventGeneratorArguments args)
      {
          await eventGenerator.Generate(args);
      }
      ```

### Test implementation

  1. Navigate to the SongRequest.MessagePublisher project root and execute:

      ```console
      dotnet build

      dotnet run
      ```

  2. Verify the application is running as expected. By default the processor should
  send one message every second for about a minute. Check your container logs to
  see if messages are being picked up by the SongRequest.MessageConsumer hosted
  in your cluster.

### Containerize and Deploy the Event Generator

Apply what you've learned to containerize and deploy the event generator.
Here are some hints to make sure you've covered everything:

  - [ ] Create a Dockerfile
  - [ ] Tag and Push the container image to ACR
  - [ ] Update the songrequest.yaml to inclue a new Deployment

To create the container image, you'll need to add another `Dockerfile` to the
root of the project directory. Rather than provide the content, try to
build the Dockerfile on your own. It will be very similar to other
Dockerfiles created for this application.

One unique requirement for this Dockerfile will be related to how the
command line arguments work. Because it's a container that is expected to
run constantly, producing random song requests, we need it to always run.
Setting the option `--run-for -1` will achieve this. Try to figure out how
to incorporate these additional command line arguments into the Dockerfile
as part of the container image.

Don't forget to test the generated container image locally before publishing
to the ACR repository.

---
## Scale Deployment
---

Now that we have an always-on message producer, let's scale the message consumer
to ensure it can keep up with the volume of messages. For this specific
deployment, we'll just adjust the replica count with the following command:

```console
kubectl scale deployment songrequest-messageconsumer --replicas=3
```

Watch the dashboard to see the additional pods get spun up for the
`songrequest-messageconsumer` deployment.

---
## Summary
---

In this lab, you demonstrated how easy it is to scale out your cluster to 
add additional capacity. You also added an event generator and exercised
what you've learned in containerizing applications and deploying them to
your AKS cluster. Finally, we scaled out a specific deployment to help
lessen strain on one specific pod.
