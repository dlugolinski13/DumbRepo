# Working with Print 2.0

## Creating a Print Section
When creating a print section we need to follow the following steps:
1. Go to the Vue project
    1. Open talis.patientrecordpdf.vue\src\assets\jsonFiles\TestFile.json
    1. Add your new object to the json for use in the vue code
        ```Json
        {
            "gridData": [
            {
                "Color": "Red",
                "GridItemType": "Event",
                "DataPoints": [
                    {
                        "Id": 123,
                        "GridText": "E1",
                        "GridDateTime": "2023-02-02T16:00:00.000Z"
                    },
                    {
                        "Id": 124,
                        "GridText": "E2",
                        "GridDateTime": "2023-02-02T16:05:00.000Z"
                    },
                    {
                        "Id": 125,
                        "GridText": "E3",
                        "GridDateTime": "2023-02-02T16:12:00.000Z"
                    }
                ]
            },
            {
                "Color": "Blue",
                "GridItemType": "Comment",
                "DataPoints": [
                    {
                        "Id": 123,
                        "GridText": "C1",
                        "GridDateTime": "2023-02-02T16:20:00.000Z"
                    },
                    {
                        "Id": 124,
                        "GridText": "C2",
                        "GridDateTime": "2023-02-02T16:25:00.000Z"
                    },
                    {
                        "Id": 125,
                        "GridText": "C3",
                        "GridDateTime": "2023-02-02T16:32:00.000Z"
                    }
                ]
            }
        }
        ```
1. Go to the Contracts project (Talis.PatientRecordPDF.Contract)
    1. Add your classes needed for the Data type you just created into the folder structure (*Some times you may need to add multiple and aggregate the classes into an object you want for the Vue component*)

        1. Create the Data folder for your class
        1. Add your classes inherit from the ContractBaseEntity class
        ```C#
        public class CommentData : ContractBaseEntity
        {
            public int Id { get; set; }
            public string CommentText { get; set; }
            public DateTimeOffset CommentDateTime { get; set; }
        }

        public class GridData : ContractBaseEntity
        {
            public int Id { get; set; }
            public string GridText { get; set; }
            public string GridItemType { get; set; }
            public DateTimeOffset GridDateTime { get; set; }
        }
        ```
    1. Create your query class (*In this case I am creating a simple query that only needs the PrintIdentifier.... it contains the EncounterId*)
        ```C#
        public class GridDataQuery
        {
            public PrintIdentifier PrintIdentifier { get; set; }
        }
        ```
        > This class will also be used in the javascript for gathering your data (*more on this later*)
    1. Create your repo interface for accessing your data
        > Inherit from the IRepo class
        ```C#
        public interface IGridDataRepo : IRepo2
        {
            Task<List<CommentData>> GetCommentDataForEncounter(GridDataQuery query);
        }
        ```
1. Go to the DataTransformation project (Talis.PatientRecordPDF.DataTransformation)
    1. Go to the Core\data-util.ts file and add your same query class as an interface
        ```typescript
        interface GridDataQuery {
            printIdentifier: PrintIdentifier
        };
        ```
    1. add your interface file named `<yourclass>-record-interfaces` for the example I am going through it would be grid-record-interfaces.ts
        ```typescript
        enum GridItemType {
            Event,
            Comment,
            Lab        
        };
        
        interface GridData{
            Id: number,
            GridText: string,
            GridItemType: GridItemType,
            GridDateTime: Date
        };

        ```
    1. Add your creation of this datatype to the base for that printout (ex. In this case it is base for all Grids so it would go into the data-util.ts but if for a specific type of printout it would go in that file)

        > When creating this function you will need to make it look like the following
        ```typescript
        async getCommentData(query?: GridDataQuery | undefined): Promise<CommentData[]> {
            if (query === null || undefined) {
                query = {
                    printIdentifier: this.PrintIdentifier
                }
            }
            // @ts-ignore
            Console.WriteLine("getCommentData is called");
            // @ts-ignore         
            let gData = await new PrintDataHelper().getCommentDataCore(query);
            // @ts-ignore
            let gridData = await new GridRecordHelper().getCommentDataArray(gData);

            return gridData;
        }
        ```
        > What this code does is take in the query (*if passed in*) and then calls the Db through the Core function that we still need to create.  It then calls the GridRecordHelper (we will create later) to format the data in the way we need it to return it to any javascript transform function that is using it
    1. Add your core function to the PrintDataHelper 
        1. Go to the file Talis.PatientRecordPDF.DataTransformation\Core\register-csharp-function.js
        1. Add your core call to the file
            ```javascript
            async getCommentDataCore(query) {
                let data = await PrintRepo.GetCommentDataForEncounter(JSON.stringify(query)).ToPromise();

                return data;
            }
            ```
            > Add the internal call from the previous function to the V8 engine

        1. Go to the Talis.PatientRecordPDF.DataTransformation\IPatientRecordPDFRepo.cs file and add your function call
            ```C#
            public interface IPatientRecordPDFRepo
            {
                Task<List<CommentData>> GetCommentDataForEncounter(string dataQuery);
            }
            ```
1. Go to the Print Project (Talis.PatientRecordPDF.Print\PrintRepoBase.cs) file and implement your interface
    > You may need to create your repo as a member variable and add it to the constructor
    ```C#
    private readonly IGridDataRepo _gridDataRepo;

    /// YOU NEED INJECT YOUR REPO FOR USE IN YOUR FUNCTION
    public PrintRepoBase(IParameterDataRepo parameterDataRepo, IEventDataRepo eventDataRepo, IGraphDataRepo graphDataRepo, IGridDataRepo gridDataRepo)
    {            
        _parameterDataRepo = parameterDataRepo;
        _eventDataRepo = eventDataRepo;
        _graphDataRepo = graphDataRepo;
        _gridDataRepo = gridDataRepo;
    }
    
    public Task<List<CommentData>> GetCommentDataForEncounter(string dataQuery)
    {
         _ = dataQuery ?? throw new ArgumentException($"{nameof(dataQuery)}");
            var query = JsonSerializer.Deserialize<GridDataQuery>(dataQuery, _jsonSerializerOptions);
            _ = query ?? throw new Exception(
               $@"{nameof(dataQuery)} didn't Deserialize to {nameof(GridDataQuery)}");
            return _gridDataRepo.GetCommentDataForEncounter(query);
    }
    ```
1. Go to the DAL (Talis.PatientRecordPDF.DAL)
    1. Add a your new Repository class Talis.PatientRecordPDF.DAL\Repository\\`<your new repo>`
    1. Inherit from RepositoryBase and implement your IRepo class you created earlier
        ```C#
        public class GridDataRepository : RepositoryBase, IGridDataRepo
        {
            public GridDataRepository(IDbContextFactory contextFactory) : base(contextFactory)
            {
            }

            public async Task<List<CommentData>> GetCommentDataForEncounter(GridDataQuery query)
            {
                /// IMPLEMENT YOUR REPO FUNCTIONALITY HERE
            }
        }
        ```
        > Before Implementing you need to add any DbSets you may need to the Entity Framework see [Adding DbSets to EF Core](#adding-dbsets-to-ef-core)
    1. Once you have your DbSet in place you can write your query

## Transformation Development
Once you have written your function for accessing the data back in step 3 of the [Creating a Print Section](#creating-a-print-section) you can use this to gather your data for you transformation type
1. Go to Talis.PatientREcordPDF.DataTransformation and create a `<name of section>`-record.ts file
1. In it create your helper functions that you called in step 3 (In my case it is **GridRecordHelper().getCommentDataArray(gData);**)
    ```typescript
    class GridRecordHelper {
        constructor() {

        }

        getCommentDataArray(gcData: any): CommentData[] {
            let ret: CommentData[] = [];

            for (let gc of gcData) {
                ret.push({
                    Id: gc.Id,
                    CommentText: gc.CommentText,
                    CommentDateTime: gc.CommentDate
                });
            };

            return ret;
        };
    }
    ```
1. Now you can add a method to your specific PrintoutBase or if it is for all printouttypes add the function to DataUtil class to get your data for your Type which you will pass back in Json (*In my case I will be adding to the DataUtil class*)
    1. Go to Talis.PatientRecordPDF.DataTransformation\Core\data-util.ts
    1. Add your method to the class to get your type
        ```typescript
        // IN THIS CASE I NEEDED TO ADD THE OBJECT TO THE BASE BECAUSE ALL PRINTOUTTYPES GET A GRID
        class DataUtil {
            public readonly gridData: GridData[];

            async createGridData(query?: GridDataQuery | undefined): Promise<GridData[]> {
                if (query === null || undefined) {
                    query = {
                        printIdentifier: this.PrintIdentifier
                    };
                }

                let commentData = await this.getCommentData(query);
                let eventData = await this.getEventData(null);

                // handle creating return type GridData[]
                for (let cData of commentData) {
                    this.gridData.push({
                        Id: cData.Id,
                        GridText: cData.CommentText,
                        GridItemType: GridItemType.Comment,
                        GridDateTime: cData.CommentDateTime
                    });
                };

                return this.gridData;
            }
        }
        ```
    1. Go to the base for your printouttype (*perfusion-record-base.ts*)
    1. Add your return type to the base 
        > In my case it is the base but in certain cases we may need to add it to the actual printout type
        ```typescript
        abstract class PerfusionRecordBase extends RecordBase {
            public gridData: GridData[];
        
            // ADDING THE CALL TO GET THE BASE GRID DATA FROM THE DATAUTIL CLASS
            async creatGridData(query?: GridDataQuery) {
                await this.dataUtil.createGridData(query);
            }
        }
        ```
    1. Go to the specific printout you are working on (*In this case I am working on perfusionRecord.ts*)
    1. Add the return to your Json
        ```typescript
        async getPatientData(): Promise<string> {

            await this.createPatient()
            await this.getEventData(null);
            await this.createGraphData(null);        

            return JSON.stringify({
                'printIdentifier': this.printIdentifier,
                'patientInfo': this.MemorialPatientInfo,
                'events': this.eventData,
                'graph1Data': this.graph1Data,
                'graph2Data': this.graph2Data,
                //THIS WAS ADDED RETURN
                'gridData': this.gridData
            });
        }
        ```


## Frontend Development
1. Setup of development in Visual Studio uses the Developer PowerShell
    * View -> Other Windows -> Command Window
1. cd to the Talis.PateintRecordPDF\Talis.PateintRecordPDF.Vue project directory
1. Type npm install (***Do this after every get latest***)
1. Type npm run build (*This is to run the build for development locally*)
1. Type npm run serve (*This will start the npm server on localhost:8088*)
1. Navigate to the page and view the Vue

### Vue Components
Components are layed out in a Customer specific way.  The core components go into the core folder.  Core components can be used at any customer, so if you need to modify a core component for a specific customer just copy that component into the customers folder and point to it from the customers MainLayout.vue
![vue project layout](VueLayout.png)
1. Create the component where it needs to go
1. Import the component into the MainLayout.vue where it is needed and add the component to the template
    ```javascript
    <template>
        <section class="print-container">
            <AmChartGrid :gridData="gridData"
                    :startTime="gsa.min"
                    :endTime="gsa.max"
                    :key="gsa[index]" />
        </section>
    </template>

    <script setup>
        import AmChartGrid from '../../core/AmChartGrid.vue'; 
    </script>
    ```
1. Modify your component as needed to see the data you want and test


## Adding DbSets to EF Core 
1. When adding tables to the context for EF Core you need to run the scaffold-db command. 
1. Find the file EntityFrameworkCommand.txt (*Talis.PatientRecordPDF.DAL\Repository\EntityFrameworkCommand.txt*)
1. Open the file in it is the command to run in the Package Manager Console
    > You may need to find the Package Manager Console it is in the View menu
1. Figure out what tables you want to add to the Context (In my example I need the CommentData and the EncounterLabPanels)  
1. Add those tables to the command with comma separated
    ```powershell
    Scaffold-DbContext "Data Source=talissqldev.eastus.cloudapp.azure.com;Initial Catalog=ACGApplication_MHS01;user id=ACGMHS01;password=7@!Smhs01;TrustServerCertificate=true" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models -ContextDir Context -DataAnnotations -f -Context ApplicationContext -Tables dbo.ParameterDefinition,dbo.GraphParameterDefinition,dbo.CaseConfiguration,dbo.Encounter,dbo.EventDefinition,dbo.EventDataDetail,dbo.EventData,dbo.MonitorData,dbo.MonitorDataManual,dbo.CommentData
    ```
    > **Warning:** If you fail to Update the EntityFrameworkCommand.txt the next time it is ran your code will become an error!
1. Go to the Package Manager Console and click the Default Project drop down and select Talis.PatientRecord.DAL
1. Copy and paste the command from the file into the Package Manager Console window
1. Run the command
    > This will build your solution so make sure you are in a good spot before doing this
1. Once done it will open the Context you added your DbSets to
    * Edit this file and remove the two constructors as well as the OnConfiguring Method
    ```C#
    public ApplicationContext()
    {
    }

    public ApplicationContext(DbContextOptions<ApplicationContext> options)
        : base(options)
    {
    }


    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    #warning To protect potentially sensitive information in your connection string, you should move it out of source code. You can avoid scaffolding the connection string by using the Name= syntax to read it from configuration - see https://go.microsoft.com/fwlink/?linkid=2131148. For more guidance on storing connection strings, see http://go.microsoft.com/fwlink/?LinkId=723263.
            => optionsBuilder.UseSqlServer("Data Source=talissqldev.eastus.cloudapp.azure.com;Initial Catalog=ACGApplication_MHS01;user id=ACGMHS01;password=7@!Smhs01;TrustServerCertificate=true");

    ```
1. Copy your new DbSets and go to the Talis.PatientRecordPDF.DAL\Context\IDbContextFactory.cs interface and paste your new DbSets
    ```C#
    public interface IDbContext : IDisposable
    {
        DbSet<CaseConfiguration> CaseConfigurations { get; set; }
        DbSet<CommentDatum> CommentData { get; set; }
    }
    ```
1. You will need to add this DbSet to the Mocking for testing but more about that later
