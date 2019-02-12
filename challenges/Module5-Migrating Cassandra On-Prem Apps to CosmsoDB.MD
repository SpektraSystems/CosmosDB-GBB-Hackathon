# Migrating Cassandra On-Prem Apps to CosmsoDB

## Step 1 - Create Cosmos DB account with Cassandra API

1. Please login to the Azure portal and choose to Create New Resource, Databases and Azure Cosmos DB.
 
 <img src="images/ca1.jpg"/><br/>

 
2. Please choose your resource group (Create a new resource group if you do not have any resource group) and give a name to Cosmos DB account. Also, choose the geographic location for the Cosmos DB account. Make sure you have chosen Cassandra as API. Currently, there are 5 APIs available in Cosmos DB.

 <img src="images/cassandra.jpg"/><br/>

3. You can review the account details and after successful validation please click the “Create” button.
 
 <img src="images/casandra1.jpg"/><br/>
 
4. It will take some to finish the deployment. You can see the status.

 <img src="images/ca2.jpg"/><br/>

5. After some time, Cosmos DB will be successfully deployed.

<img src="images/ca3.jpg"/><br/>
 
6. You can click “Go to resource” button and open the Cosmos DB account. Please click the “Connection String” tab and get the connection details about the Cassandra database. We will use these connection details in our Blazor application later.

7. We can go to the “Data Explorer” tab to create a new Keyspace now. The keyspace is like a database and tables are created under this Keyspace.
 
8. We can create a new Keyspace now.

<img src="images/key.jpg"/><br/>

9. After successful creation of Keyspace, we can create a Table now. For Table creation, we must use CQL (Cassandra Query Language) commands.
```
Most of the CQL commands are like SQL commands. 
CREATE TABLE IF NOT EXISTS sarathlal.addressbook
(id text PRIMARY KEY, firstname text, lastname text,
 gender text, address text, zipcode text,
 country text, state text, phone text)
 ```
 <img src="images/key2.jpg"/><br/>
## Step 2 - Create Blazor application.

1. Please open Visual Studio 2017 (I am using a free community edition) and create a Blazor app. Choose an ASP.NET Core Web Application project template.

<img src="images/ca4.jpg"/><br/>

2. We can choose Blazor (ASP.NET Core hosted) template.<br/>
 <img src="images/cs6.jpg"/><br/>
3. Our solution will be ready in a moment. Please note that there are three projects created in our solution - “Client”, “Server”, and “Shared”.The client project contains all the client-side libraries and Razor Views, while the server project contains the Web API Controller and other business logic. The shared project contains the commonly shared files, like models and interfaces.<br/>
<img src="images/ca5.jpg"/><br/>
4. By default, Blazor created many files in these three projects. We can remove all the unwanted files like “Counter.cshtml”, “FetchData.cshtml”, “SurveyPrompt.cshtml” from Client project and “SampleDataController.cs” file from Server project and delete “WeatherForecast.cs” file from shared project too.<br/>

5. We can add “CassandraCSharpDriver” NuGet package to Shared project and server project. Both projects need this package. This package is used for creating connectivity between Blazor app and Cassandra.
 <img src="images/ca7.jpg"/><br/>
6. As I mentioned earlier, we will create an Address Book application. We can create AddressBook model class now. Please create a “Models” folder in the Shared project and create AddressBook class inside this folder

* Please copy below code and paste to a new class.

**AddressBook.cs**
```
using Cassandra.Mapping.Attributes;  
  
namespace BlazorWithCassandraAddressBook.Shared.Models  
{  
    [Table("addressbook")]  
    public class AddressBook  
    {  
        [Column("id")]  
        public string Id  
        {  
            get;  
            set;  
        }  
        [Column("firstname")]  
        public string FirstName  
        {  
            get;  
            set;  
        }  
        [Column("lastname")]  
        public string LastName  
        {  
            get;  
            set;  
        }  
        [Column("gender")]  
        public string Gender  
        {  
            get;  
            set;  
        }  
        [Column("address")]  
        public string Address  
        {  
            get;  
            set;  
        }  
        [Column("zipcode")]  
        public string ZipCode  
        {  
            get;  
            set;  
        }  
        [Column("country")]  
        public string Country  
        {  
            get;  
            set;  
        }  
        [Column("state")]  
        public string State  
        {  
            get;  
            set;  
        }  
        [Column("phone")]  
        public string Phone  
        {  
            get;  
            set;  
        }  
    }  
}  
```
```
Please note : we have used “Table” and “Column” properties in this class. This is derived from “Cassandra.Mapping.Attributes” and used to mention the Table name and Column names of Cassandra table.
```

7. We can create an “IDataAccessProvider” interface now. This interface will contain all the signature of DataAccess class and we will implement this interface in our DataAccess class later.

* Please copy the below code and paste to the interface.

**IDataAccessProvider.cs**
```
using System.Collections.Generic;  
using System.Threading.Tasks;  
  
namespace BlazorWithCassandraAddressBook.Shared.Models  
{  
    public interface IDataAccessProvider  
    {  
        Task AddRecord(AddressBook addressBook);  
        Task UpdateRecord(AddressBook addressBook);  
        Task DeleteRecord(string id);  
        Task<AddressBook> GetSingleRecord(string id);  
        Task<IEnumerable<AddressBook>> GetAllRecords();  
    }  
}  
```

8. We can go to our Server project and create a new folder “DataAccess”. We will create a static class “CassandraInitializer” inside this folder. This class will be called from application Startup class and this will be used to initialize our Cassandra database.

**CassandraInitializer.cs**
```
using Cassandra;  
using System.Diagnostics;  
using System.Net.Security;  
using System.Security.Authentication;  
using System.Security.Cryptography.X509Certificates;  
using System.Threading.Tasks;  
  
namespace BlazorWithCassandraAddressBook.Server.DataAccess  
{  
    public static class CassandraInitializer  
    {  
        // Cassandra Cluster Configs        
        private const string UserName = "Fill the details";  
        private const string Password = "Fill the details";  
        private const string CassandraContactPoint = "Fill the details";  
        private static readonly int CassandraPort = 10350;  
        public static ISession session;  
  
        public static async Task InitializeCassandraSession()  
        {  
            // Connect to cassandra cluster  (Cassandra API on Azure Cosmos DB supports only TLSv1.2)  
            var options = new SSLOptions(SslProtocols.Tls12, true, ValidateServerCertificate);  
            options.SetHostNameResolver((ipAddress) => CassandraContactPoint);  
            Cluster cluster = Cluster.Builder().WithCredentials(UserName, Password).WithPort(CassandraPort).AddContactPoint(CassandraContactPoint).WithSSL(options).Build();  
  
            session = await cluster.ConnectAsync("sarathlal");  
        }  
  
        public static bool ValidateServerCertificate(object sender, X509Certificate certificate, X509Chain chain, SslPolicyErrors sslPolicyErrors)  
        {  
            if (sslPolicyErrors == SslPolicyErrors.None)  
                return true;  
  
            Trace.WriteLine("Certificate error: {0}", sslPolicyErrors.ToString());  
            // Do not allow this client to communicate with unauthenticated servers.  
            return false;  
        }  
    }  
}  
```
9. We have given the Cosmos DB Cassandra connection details inside the above class. A static Cassandra “Session” will be created at the application startup and will be shared across the entire application.

10. We can create the class “DataAccessCassandraProvider” inside the DataAccess folder. This class will implement the interface IDataAccessProvider and define all the CRUD operations. We will call these methods from our Web API controller later.

* Please copy the below code and paste to class.

**DataAccessCassandraProvider.cs**
```
DataAccessCassandraProvider.cs
using BlazorWithCassandraAddressBook.Shared.Models;  
using Cassandra;  
using Cassandra.Mapping;  
using Microsoft.Extensions.Logging;  
using System;  
using System.Collections.Generic;  
using System.Linq;  
using System.Threading.Tasks;  
  
namespace BlazorWithCassandraAddressBook.Server.DataAccess  
{  
    public class DataAccessCassandraProvider : IDataAccessProvider  
    {  
        private readonly ILogger _logger;  
        private readonly IMapper mapper;  
        public DataAccessCassandraProvider(ILoggerFactory loggerFactory)  
        {  
            _logger = loggerFactory.CreateLogger("DataAccessCassandraProvider");  
            mapper = new Mapper(CassandraInitializer.session);  
        }  
  
        public async Task AddRecord(AddressBook addressBook)  
        {  
            addressBook.Id = Guid.NewGuid().ToString();  
            await mapper.InsertAsync(addressBook);  
        }  
  
        public async Task UpdateRecord(AddressBook addressBook)  
        {  
            var updateStatement = new SimpleStatement("UPDATE addressbook SET " +  
                " firstname = ? " +  
                ",lastname  = ? " +  
                ",gender    = ? " +  
                ",address   = ? " +  
                ",zipcode   = ? " +  
                ",country   = ? " +  
                ",state     = ? " +  
                ",phone     = ? " +  
                " WHERE id  = ? ",  
                addressBook.FirstName, addressBook.LastName, addressBook.Gender,  
                addressBook.Address, addressBook.ZipCode, addressBook.Country,  
                addressBook.State, addressBook.Phone, addressBook.Id);  
  
            await CassandraInitializer.session.ExecuteAsync(updateStatement);  
        }  
  
        public async Task DeleteRecord(string id)  
        {  
            var deleteStatement = new SimpleStatement("DELETE FROM addressbook WHERE id = ? ", id);  
            await CassandraInitializer.session.ExecuteAsync(deleteStatement);  
        }  
  
        public async Task<AddressBook> GetSingleRecord(string id)  
        {  
            AddressBook addressBook = await mapper.SingleOrDefaultAsync<AddressBook>("SELECT * FROM addressbook WHERE id = ?", id);  
            return addressBook;  
        }  
  
        public async Task<IEnumerable<AddressBook>> GetAllRecords()  
        {  
            IEnumerable<AddressBook> addressBooks = await mapper.FetchAsync<AddressBook>("SELECT * FROM addressbook");  
            return addressBooks;  
  
        }  
    }  
}  
```

11. Please note we have defined all the CRUD operations inside above class. For that, we have used the Mapper object for inserting a Cassandra record and getting records from Cassandra. For update and delete we used Cassandra session Execute method.

12. We can initiate this DataAccess provider class inside the “Startup” class using dependency injection. We will also call the “InitializeCassandraSession” method inCassandraInitializer class from this Startup class.

<img src="images/ca8.jpg"/><br/>

**Startup.cs**
``` 
using BlazorWithCassandraAddressBook.Server.DataAccess;  
using BlazorWithCassandraAddressBook.Shared.Models;  
using Microsoft.AspNetCore.Blazor.Server;  
using Microsoft.AspNetCore.Builder;  
using Microsoft.AspNetCore.Hosting;  
using Microsoft.AspNetCore.ResponseCompression;  
using Microsoft.Extensions.DependencyInjection;  
using System.Linq;  
using System.Net.Mime;  
using System.Threading.Tasks;  
  
namespace BlazorWithCassandraAddressBook.Server  
{  
    public class Startup  
    {  
        public void ConfigureServices(IServiceCollection services)  
        {  
            services.AddMvc();  
  
            services.AddResponseCompression(options =>  
            {  
                options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(new[]  
                {  
                    MediaTypeNames.Application.Octet,  
                    WasmMediaTypeNames.Application.Wasm,  
                });  
            });  
  
            services.AddScoped<IDataAccessProvider, DataAccessCassandraProvider>();  
  
            Task t = CassandraInitializer.InitializeCassandraSession();  
            t.Wait();  
        }  
  
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)  
        {  
            app.UseResponseCompression();  
  
            if (env.IsDevelopment())  
            {  
                app.UseDeveloperExceptionPage();  
            }  
  
            app.UseMvc(routes =>  
            {  
                routes.MapRoute(name: "default", template: "{controller}/{action}/{id?}");  
            });  
  
            app.UseBlazor<Client.Program>();  
        }  
    }  
} 
```
13. We can create the Web API controller “AddressBooksController” now.

**AddressBooksController.cs**
```
using BlazorWithCassandraAddressBook.Shared.Models;  
using Microsoft.AspNetCore.Mvc;  
using System.Collections.Generic;  
using System.Threading.Tasks;  
  
namespace BlazorWithCassandraAddressBook.Server.Controllers  
{  
    public class AddressBooksController : Controller  
    {  
        private readonly IDataAccessProvider _dataAccessProvider;  
  
        public AddressBooksController(IDataAccessProvider dataAccessProvider)  
        {  
            _dataAccessProvider = dataAccessProvider;  
        }  
  
        [HttpGet]  
        [Route("api/AddressBooks/Get")]  
        public async Task<IEnumerable<AddressBook>> Get()  
        {  
            return await _dataAccessProvider.GetAllRecords();  
        }  
  
        [HttpPost]  
        [Route("api/AddressBooks/Create")]  
        public async Task Create([FromBody]AddressBook addressBook)  
        {  
            if (ModelState.IsValid)  
            {  
  
                await _dataAccessProvider.AddRecord(addressBook);  
            }  
        }  
  
        [HttpGet]  
        [Route("api/AddressBooks/Details/{id}")]  
        public async Task<AddressBook> Details(string id)  
        {  
            return await _dataAccessProvider.GetSingleRecord(id);  
        }  
  
        [HttpPut]  
        [Route("api/AddressBooks/Edit")]  
        public async Task Edit([FromBody]AddressBook addressBook)  
        {  
            if (ModelState.IsValid)  
            {  
                await _dataAccessProvider.UpdateRecord(addressBook);  
            }  
        }  
  
        [HttpDelete]  
        [Route("api/AddressBooks/Delete/{id}")]  
        public async Task Delete(string id)  
        {  
            await _dataAccessProvider.DeleteRecord(id);  
        }  
    }  
}  
```
14. We have initialized “IDataAccessProvider” interface in the controller constructor.

15. We have defined all the HTTP GET, PUT, DELETE, and POST methods for our CRUD actions inside this controller. Please note all these methods are asynchronous.
16. We have completed the coding part for Shared and Server projects. We can go to the Client project and modify existing “NavMenu.cshtml” file. This is a Razor view file and used for navigation purposes.

**NavMenu.cshtml**
```
<div class="top-row pl-4 navbar navbar-dark">    
    <a class="navbar-brand" href="">Address Book App</a>    
    <button class="navbar-toggler" onclick=@ToggleNavMenu>    
        <span class="navbar-toggler-icon"></span>    
    </button>    
</div>    
    
<div class=@(collapseNavMenu ? "collapse" : null) onclick=@ToggleNavMenu>    
    <ul class="nav flex-column">    
    
        <li class="nav-item px-3">    
            <NavLink class="nav-link" href="" Match=NavLinkMatch.All>    
                <span class="oi oi-home" aria-hidden="true"></span> Home    
            </NavLink>    
        </li>    
    
        <li class="nav-item px-3">    
            <NavLink class="nav-link" href="/listaddressbooks">    
                <span class="oi oi-list-rich" aria-hidden="true"></span> Address Book Details    
            </NavLink>    
        </li>    
    </ul>    
</div>    
    
@functions {    
bool collapseNavMenu = true;    
    
void ToggleNavMenu()    
{    
    collapseNavMenu = !collapseNavMenu;    
}    
}    
```
17. We can add “ListAddresBooks.cshtml” Razor view now. This view will be used to display all Address Book details.

**ListAddresBooks.cshtml**
```
@using BlazorWithCassandraAddressBook.Shared.Models    
@page "/listaddressbooks"    
@inject HttpClient Http    
    
<h1>Address Book Details</h1>    
<p>    
    <a href="/addaddressbook">Create New Address Book</a>    
</p>    
@if (addressBooks == null)    
{    
    <p><em>Loading...</em></p>    
}    
else    
{    
    <table class='table'>    
        <thead>    
            <tr>    
                <th>First Name</th>    
                <th>Last Name</th>    
                <th>Gender</th>    
                <th>Address</th>    
                <th>ZipCode</th>    
                <th>Phone</th>    
            </tr>    
        </thead>    
        <tbody>    
            @foreach (var address in addressBooks)    
            {    
                <tr>    
                    <td>@address.FirstName</td>    
                    <td>@address.LastName</td>    
                    <td>@address.Gender</td>    
                    <td>@address.Address</td>    
                    <td>@address.ZipCode</td>    
                    <td>@address.Phone</td>    
                    <td>    
                        <a href='/editaddressbook/@address.Id'>Edit</a>    
                        <a href='/deleteaddressbook/@address.Id'>Delete</a>    
                    </td>    
                </tr>    
            }    
        </tbody>    
    </table>    
}    
@functions {    
AddressBook[] addressBooks;    
    
protected override async Task OnInitAsync()    
{    
    addressBooks = await Http.GetJsonAsync<AddressBook[]>("/api/AddressBooks/Get");    
   }      
}   
```
* Please add the below three Razor views also.
 
**AddAddressBook.cshtml**
```
@using BlazorWithCassandraAddressBook.Shared.Models    
@page "/addaddressbook"    
@inject HttpClient Http    
@inject Microsoft.AspNetCore.Blazor.Services.IUriHelper UriHelper    
    
<h2>Create Address Book</h2>    
<hr />    
<form>    
    <div class="row">    
        <div class="col-md-4">    
            <div class="form-group">    
                <label for="FirstName" class="control-label">First Name</label>    
                <input for="FirstName" class="form-control" bind="@addressBook.FirstName" />    
            </div>    
            <div class="form-group">    
                <label for="Gender" class="control-label">Gender</label>    
                <select for="Gender" class="form-control" bind="@addressBook.Gender">    
                    <option value="">-- Select Gender --</option>    
                    <option value="Male">Male</option>    
                    <option value="Female">Female</option>    
                </select>    
            </div>    
            <div class="form-group">    
                <label for="ZipCode" class="control-label">ZipCode</label>    
                <input for="ZipCode" class="form-control" bind="@addressBook.ZipCode" />    
            </div>    
            <div class="form-group">    
                <label for="State" class="control-label">State</label>    
                <input for="State" class="form-control" bind="@addressBook.State" />    
            </div>    
        </div>    
        <div class="col-md-4">    
            <div class="form-group">    
                <label for="LastName" class="control-label">Last Name</label>    
                <input for="LastName" class="form-control" bind="@addressBook.LastName" />    
            </div>    
            <div class="form-group">    
                <label for="Address" class="control-label">Address</label>    
                <input for="Address" class="form-control" bind="@addressBook.Address" />    
            </div>    
            <div class="form-group">    
                <label for="Country" class="control-label">Country</label>    
                <input for="Country" class="form-control" bind="@addressBook.Country" />    
            </div>    
            <div class="form-group">    
                <label for="Phone" class="control-label">Phone</label>    
                <input for="Phone" class="form-control" bind="@addressBook.Phone" />    
            </div>    
        </div>    
    </div>    
    <div class="row">    
        <div class="col-md-4">    
            <div class="form-group">    
                <input type="button" class="btn btn-default" onclick="@(async () => await CreateAddressBook())" value="Save" />    
                <input type="button" class="btn" onclick="@Cancel" value="Cancel" />    
            </div>    
        </div>    
    </div>    
</form>    
@functions {    
    
AddressBook addressBook = new AddressBook();    
    
protected async Task CreateAddressBook()    
{    
    await Http.SendJsonAsync(HttpMethod.Post, "/api/AddressBooks/Create", addressBook);    
    UriHelper.NavigateTo("/listaddressbooks");    
}    
    
void Cancel()    
{    
    UriHelper.NavigateTo("/listaddressbooks");    
}    
}   
```
**EditAddressBook.cshtml**
```
@using BlazorWithCassandraAddressBook.Shared.Models    
@page "/editaddressbook/{Id}"    
@inject HttpClient Http    
@inject Microsoft.AspNetCore.Blazor.Services.IUriHelper UriHelper    
    
<h2>Edit</h2>    
<h4>Address Book</h4>    
<hr />    
    
<form>    
    <div class="row">    
        <div class="col-md-4">    
            <div class="form-group">    
                <label for="FirstName" class="control-label">FirstName</label>    
                <input for="FirstName" class="form-control" bind="@addressBook.FirstName" />    
            </div>    
            <div class="form-group">    
                <label for="Gender" class="control-label">Gender</label>    
                <select for="Gender" class="form-control" bind="@addressBook.Gender">    
                    <option value="">-- Select Gender --</option>    
                    <option value="Male">Male</option>    
                    <option value="Female">Female</option>    
                </select>    
            </div>    
            <div class="form-group">    
                <label for="ZipCode" class="control-label">ZipCode</label>    
                <input for="ZipCode" class="form-control" bind="@addressBook.ZipCode" />    
            </div>    
            <div class="form-group">    
                <label for="State" class="control-label">State</label>    
                <input for="State" class="form-control" bind="@addressBook.State" />    
            </div>    
        </div>    
        <div class="col-md-4">    
            <div class="form-group">    
                <label for="LastName" class="control-label">LastName</label>    
                <input for="LastName" class="form-control" bind="@addressBook.LastName" />    
            </div>    
            <div class="form-group">    
                <label for="Address" class="control-label">Address</label>    
                <input for="Address" class="form-control" bind="@addressBook.Address" />    
            </div>    
            <div class="form-group">    
                <label for="Country" class="control-label">Country</label>    
                <input for="Country" class="form-control" bind="@addressBook.Country" />    
            </div>    
            <div class="form-group">    
                <label for="Phone" class="control-label">Phone</label>    
                <input for="Phone" class="form-control" bind="@addressBook.Phone" />    
            </div>    
        </div>    
    </div>    
    <div class="row">    
        <div class="form-group">    
            <input type="button" class="btn btn-default" onclick="@(async () => await UpdateAddressBook())" value="Save" />    
            <input type="button" class="btn" onclick="@Cancel" value="Cancel" />    
        </div>    
    </div>    
</form>    
    
@functions {    
    
[Parameter]    
string Id { get; set; }    
    
AddressBook addressBook = new AddressBook();    
    
protected override async Task OnInitAsync()    
{    
    addressBook = await Http.GetJsonAsync<AddressBook>("/api/AddressBooks/Details/" + Id);    
}    
    
protected async Task UpdateAddressBook()    
{    
    await Http.SendJsonAsync(HttpMethod.Put, "api/AddressBooks/Edit", addressBook);    
    UriHelper.NavigateTo("/listaddressbooks");    
    
}    
    
void Cancel()    
{    
    UriHelper.NavigateTo("/listaddressbooks");    
}    
}     
```
**DeleteAddressBook.cshtml**
```
@using BlazorWithCassandraAddressBook.Shared.Models    
@page "/deleteaddressbook/{Id}"    
@inject HttpClient Http    
@inject Microsoft.AspNetCore.Blazor.Services.IUriHelper UriHelper    
    
<h2>Delete</h2>    
<p>Are you sure you want to delete this Address with Id :<b> @Id</b></p>    
<br />    
<div class="col-md-4">    
    <table class="table">    
        <tr>    
            <td>FirstName</td>    
            <td>@addressBook.FirstName</td>    
        </tr>    
        <tr>    
            <td>LastName</td>    
            <td>@addressBook.LastName</td>    
        </tr>    
        <tr>    
            <td>Gender</td>    
            <td>@addressBook.Gender</td>    
        </tr>    
        <tr>    
            <td>Country</td>    
            <td>@addressBook.Country</td>    
        </tr>    
        <tr>    
            <td>Phone</td>    
            <td>@addressBook.Phone</td>    
        </tr>    
    </table>    
    <div class="form-group">    
        <input type="button" value="Delete" onclick="@(async () => await Delete())" class="btn btn-default" />    
        <input type="button" value="Cancel" onclick="@Cancel" class="btn" />    
    </div>    
</div>    
@functions {    
    
[Parameter]    
string Id { get; set; }    
    
AddressBook addressBook = new AddressBook();    
    
protected override async Task OnInitAsync()    
{    
    addressBook = await Http.GetJsonAsync<AddressBook>    
("/api/AddressBooks/Details/" + Id);    
}    
    
protected async Task Delete()    
{    
    await Http.DeleteAsync("api/AddressBooks/Delete/" + Id);    
    UriHelper.NavigateTo("/listaddressbooks");    
}    
    
void Cancel()    
{    
    UriHelper.NavigateTo("/listaddressbooks");    
}    
}   
```
18. We have completed all the coding part now. We can run the Address Book app now.
 
 <img src="images/blozar.jpg"/><br/>

19. We can add a new Address Book record now.
 
 <img src="images/blazor3.jpg"/><br/>

20. We can add one more record and display the records.
 
 <img src="images/blazor4.jpg"/><br/>

21. We can edit one record.
 
 <img src="images/blazor3.jpg"/><br/>

22. I have changed the address field.

23. Now we can delete one record.

 <img src="images/blazor1.jpg"/><br/>

24. We can check the data in Azure Cosmos DB account also. Please open Data Explorer tab and click CQL Query Text and Run. We will see one address book record created by our Blazor application.

<img src="images/blozar.jpg"/><br/>

 
> In this article, we have seen how to create a Cosmos DB account with Cassandra API and we created a Blazor application with ASP.NET Core hosted template. We have used CassandraCSharpDriver NuGet package to connect Blazor with Cassandra. We used some CQL (Cassandra Query Language) commands inside our DataAccess provider class. We have seen all CRUD operations with our Address Book application.
 

