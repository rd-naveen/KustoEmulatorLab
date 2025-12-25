### Kusto Data Explorer Emulator installation:

I have tested all of the below steps in the Windows 11 Enterprise editions, it should work, but if you face any issues or wish modify some steps(because of they are outdated) feel free to create a PR.

The data and resources used in these repository are mostly from publicly available sources, such as Microsoft `ADX In A Day` workshop repo (kudos to them), and other sources such as blogs and articles.

Refer: https://sandervandevelde.wordpress.com/2023/07/06/turn-your-laptop-into-an-azure-data-explorer-cluster/


Disclaimer: The products/tools used in these exercise are covers under  `Microsoft Software License` and other products copies, read the terms and conditions provided in the official documentation before installation and usage. And strictly should not be used in production and use with caution.

### Word from Microsoft official docs

`The Kusto emulator is a local environment that encapsulates the query engine. You can use the environment to facilitate local development and automated testing. Since the environment runs locally, it doesn't require provisioning Azure services or incurring any cost; it's a free offering under the Microsoft Software License Terms.`

Refer: https://kustoes.blob.core.windows.net/kusto/ADX-Emulator-Evaluation-Terms.pdf

Refer: https://github.com/Azure/ADX-in-a-Day (they have a cool two day plan for achieving basic master in ADX topics)



#### Step 1: Install docker Desktop on windows or linux env

    Download the community version of `Docker for Desktop` from the official site. 
    
    https://docs.docker.com/desktop/setup/install/windows-install/


    Note: Read the requirements and `install the MSI/installer only from the official site`.

    Note: Feel free to use the commercial version if you have any. 

#### Step 2: Pull the latest kusto emulator image from azure artifacts repository and Run it (on custom port and with mounted storage for persisitance)

For understanding the difference between free tier and kusto emulator, as well as the limitations of Kusto emulator, refer this pages. https://learn.microsoft.com/en-us/azure/data-explorer/kusto-emulator-overview


`#docker run -v d:\host\local:/kustodata -e ACCEPT_EULA=Y -m 8G -d -p 8080:8080 -t mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest`
    
    
```
#docker ps
CONTAINER ID  IMAGE                                                        COMMAND             CREATED         STATUS         PORTS                    NAMES
d0002cd59605   mcr.microsoft.com/azuredataexplorer/kustainer-linux:latest   "/start-kusto.sh"   7 minutes ago   Up 7 minutes   0.0.0.0:8080->8080/tcp   admiring_bhabha
```
        
Note: we don't need to create the folder `d:\host\local`, docker will create it and use it in the creations, Even if we created it it won't affect the command execution.

Note: in the official documentation, it is recommend to use windows version of container, But i found that linux container is very light, and suitable for low bandwidth networks and devices with low processing capacity.  

We are mounting the docker storage in the host system for the persistence mechanism, we could also choose to go without using mountain, but the data we ingest in the kusto won't be persistent across restarts. 

#### Step 3: Verify the mounted folders in host is created, 
        
        ls d:\host\

#### Step 4: Configure the Kusto Data Explorer to connect with Local Kusto emulator

According to the official documentation, we can choose Kusto Data Explorer, Kusto.CLI or Kusto.SDK to interact with the kusto clusters running in the containers, Unfortunately we can't experience the web-UI in kusto emulator. 

Refer: https://learn.microsoft.com/en-us/azure/data-explorer/kusto-emulator-install?tabs=linux-container


For now, we will use the `Kusto Data Explorer` windows application to explore the kusto emulator features.  

Download the `Kusto Data explorer` app from  https://aka.ms/ke

`Kusto Data explorer` configurations for connecting local kusto emulator,

    Tools Tab -> Connections -> 
        1. check the "Allow unsafe connections" box
        2. Change "Query Server timeout" to 01:00:00
        3. Change "Admin Command Server timeout" to 01:00:00

    Right click the `Connection` pane in the kusto data explorer and select "Add connection"
        cluster connection -> http://localhost:8080
        Security -> Client security (None)


    To verify if everything is working fine, Right click the newly added connections and click 'refresh` we should not see any errors.

#### Step 5: Create a database to use in the local kusto emulator

Now, let's create a database in the local kusto emulator for experimenting with it's features. 

Run the following admin commands in the query tab

```
.create database localtestdb persist (
    @"/kustodata/dbs/localtestdb/md",
    @"/kustodata/dbs/localtestdb/data"
)
```

Change the database `localtestdb` to something else, if you want

After executing the command, in the host system the following paths should be created, 

`# ls D:\host\local\dbs\localtestdb`


    Directory: D:\host\local\dbs\localtestdb


    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d-----        24-05-2025     09:00                data
    d-----        24-05-2025     09:00                md
    d-----        24-05-2025     09:00                mdmd1

#### Step 6: Now create a table schema

Select the newly created database name in the Connection pane before executing any database/table oriented commands. 

        .create table logsRaw(Timestamp:datetime, Source:string, Node:string, Level:string, Component:string, ClientRequestId:string, Message:string, Properties:dynamic)

        .show tables

        TableName	DatabaseName	Folder	DocString
        logsRaw	NetDefaultDB		

#### Step 7: Next, Ingest the data from remote storage blob to table (For smaller files located in the remote blob)

Note: we need to select the database where we need out data to be, before executing any of the ingest/create commands, else the data will be goes to default database. 

##### Method A: Ingest directly

        .ingest into table logsRaw (h'https://adxsamplefiles.blob.core.windows.net/publiccsvsamples/logsbenchmark-onegb/2014/03/08/00/data.csv.gz') with (format='csv', creationTime='2014-03-08T00:00:00Z');

We should get the output something like this, with "HasErrors" field value is False, if otherwise, there are some errors during ingestion, we need to check why 

        ExtentId	ItemLoaded	Duration	HasErrors	OperationId
        99e885d9-ed56-4f9f-acb9-f0c757a9f74d	https://adxsamplefiles.blob.core.windows.net/publiccsvsamples/logsbenchmark-onegb/2014/03/08/00/data.csv.gz	00:00:11.3491886	False	69df9f57-353e-4f33-b06c-d4fa1f807420

##### Method B: Ingest large from remote blob storage to table, asynchronously (we will get the operation id and we can check it's status later)

        .ingest async  into table logsRaw (h'https://adxsamplefiles.blob.core.windows.net/publiccsvsamples/logsbenchmark-onegb/2014/03/08/00/data.csv.gz') with (format='csv', creationTime='2014-03-08T00:00:00Z');
    
We should get the output something like this, if we only get partial result, which means the ingestion is in-progress    

        .show operation operation-id

        OperationId	Operation	NodeId	StartedOn	LastUpdatedOn	Duration	State	Status
        69df9f57-353e-4f33-b06c-d4fa1f807420	DataIngestPull	0	2025-05-24 03:53:26.0086723	2025-05-24 03:53:37.3578609	00:00:11.3491886	Completed	Extent(s) created; metadata updated; cluster map updated
        69df9f57-353e-4f33-b06c-d4fa1f807420	DataIngestPull	0	2025-05-24 03:53:26.0086723	2025-05-24 03:53:37.3567731	00:00:11.3481152	InProgress	Extent(s) created; metadata updated; cluster map updated

##### Method C: Now let's look at how we can ingest data located at locally to the table. 

For this exercise, Let's download the above example file from the blob url, and store it in the host mapped location.
    
        https://adxsamplefiles.blob.core.windows.net/publiccsvsamples/logsbenchmark-onegb/2014/03/08/00/data.csv.gz

        copy data.csv.gz d:\host\local\

and now we can ingest the data into table

        .ingest into table logsRaw( @"/kustodata/data.csv.gz") with (format='csv')

Note: In the above file, the file extension is .gz, but in the format attribute we are mentioning "csv", because Kusto is capable of recognizing .zip and .gz files and extracting files. 

Refer https://learn.microsoft.com/en-us/kusto/ingestion-supported-formats?view=azure-data-explorer

#### Step 8: View the ingested Logs:
    
    logsRaw
    | limit 10

