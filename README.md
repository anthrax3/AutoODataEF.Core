## ASP.Net core, EF core, OData vNext - auto generate controllers and repositories
This project allows you to generate OData v4 ASP.Net core controllers from an existing code first entity framework model. Also generated are repositories for 
entities, along with proxies to intercept repository actions.


## Executing the example
The VS 2017 solution uses ASP.NET Core and targets .net 4.6.1+ - because the Microsoft.AspNetCore.OData.vNext 6.0.2-alpha-rtm only targets .net 4.6.1+. 

The easiest way to see what this solution does (documentation follows this section) is to:

* Clone the repo/Download a zip
* Open the OData-EF-APIGenerator solution
* Restore packages as necessary
* Build the solution
* Open the nuget package manager console, ensure the default project is set to EF.Example, and execute Update-Database, which will build and seed an initial SQL Express database. 
If you have SQL server instead of SQL express, just modify the connection strings in the EF.Example project
* Press F5, which should start the Generated.OData.EF.API project

The OData MVC controllers were built using a T4 template and by auto examining the EF.Example code first entity model. 

Using Postman, or Chrome, you can then try the OData API:

http://localhost:60241/odata/Suppliers(1)/ProductsSupplied

POST http://localhost:60241/odata/Clients, with a body of:
{
"Name": "Agatha Christie",
"Orders": []
}

The headers should include a location in the response:
Location http://localhost:60241/odata/Clients(21)

http://localhost:60241/odata/Clients(2)

http://localhost:60241/odata/Clients?$top=5

http://localhost:60241/odata/Clients?$top=5&$skip=5

## Project specific use

The project API.T4.Templates has a simple T4 template definition, that can be hand modified to suit.

The T4 templates have some limited configuration capability, as below, in Common.ttinclude:
``` 
// Namespaces to include (for EF model and so on)
var includeNamespaces = new List<string> { "EF.Example" };
// The type of the EF context
var contextInterface = "ICompanyContext"; 
// Base namespace for all generated objects
var baseNamespace = "Autogenerated.OData.Api";
// The OData route prefix to use
var routePrefix = "odata";
```

Also, to use your own db context definition, you have to modify the start of the Common.ttinclude file, this part:

```
<#
	var parser = new DbContextParser(typeof(EF.Example.CompanyContext));
	var model = parser.Construct();
#>
```

So, assume you have an EF code first model in an project called OrgProject.EF, the context has an interface IOrgContext, a concrete type of OrgContext, and the namespace for entities is
Organisation.Entity.Model, you'd make these changes and perform these actions to generate an initial OData set of controllers:

* Modify Common.ttinclude, change "EF.Example" to "Organisation.Entity.Model"
* Modify Common.ttinclude, change var contextInterface = "ICompanyContext" to var contextInterface = "IOrgContext"
* Modify Common.ttinclude, change typeof(typeof(EF.Example.CompanyContext)) to typeof(Organisation.Entity.Model.OrgContext)
* Remove the projects reference to EF.Example, and replace with a project reference to OrgProject.EF
* Right click on Api-Generator.tt, and Run Custom tool
* Build the API.T4.Templates project - there should be no errors

Now:
* Open Api-Generator.cs, ctrl A and ctrl C, and paste into the file Generated.OData.EF.API/Generated.cs
* Remove the line ```services.AddScoped<IInterventionProxy<ICompanyContext, Customer>, ClientsInterventionProxy>();``` in Startup.cs
* Remove the file Generated.OData.EF.API/Proxies/ClientsInterventionProxy.cs
* Remove the line ```using EF.Example;``` in Startup.cs
* Edit appsettings.json to modify the Default connection string to whatever is applicable to your installation
* Change the lines:
```
	services.AddScoped<ICompanyContext, CompanyContext>(_ => {
                var builder = new DbContextOptionsBuilder<CompanyContext>();
                builder.UseSqlServer(Configuration["ConnectionStrings:Default"]);
                return new CompanyContext(builder.Options);
         });
```
to use IOrgContext and OrgContext (as per our initial assumptions).

# Declarative use
A few attributes are currently included to allow the generation process to be modified - the EF.Example project entities and context havea few examples:

| Attribute           |Semantics                                    |
|---------------------|---------------------------------------------|
| ApiExclusion|Exclude a DbSet<> from the API|
| ApiResourceCollection|Supply a specific name to a ResourceCollection|
| ApiExposedResourceProperty|Expose a specific entity property as a resource property|
| ApiNullifyOnCreate|Request that property be nullified when the enclosing object is being created|

# Proxies
Proxies are injected into repositories to enable a crude form of AOP. Proxies are generated for all resource collections/controllers.

An example one from the base installation is:

```
public class ClientsProxy : Proxy<ICompanyContext, EF.Example.Customer>  {
		
	public ClientsProxy(IInterventionProxy<ICompanyContext, EF.Example.Customer> proxy = null) : base(proxy) { 
	}

	public override EF.Example.Customer PreCreate(ICompanyContext ctx, EF.Example.Customer entity) { 
		entity.CustomerId = default(System.Int32);
		entity.Orders = null;
		return base.PreCreate(ctx, entity); 	
	}
}
```
The purpose of this implementation is to ensure the entity being created does not have extraneous details; so, the PK Id should be zero, and the 
Orders property is nullified as the definition of Orders in Customer is decorated with the ApiNullifyOnCreate attribute.

You can create your own proxies that are delegated to by the main proxy, by implementing IInterventionProxy<TContext, TEntity>. This is the 
example from the base installation, which does nothing of value, but illustrates the point:

```
public class ClientsInterventionProxy : IInterventionProxy<ICompanyContext, Customer> {
        public Customer PreCreate(ICompanyContext ctx, Customer entity) {
            return entity;
        }
        public Customer PostCreate(ICompanyContext ctx, Customer entity) { return entity; }
        public Customer PreUpdate(ICompanyContext ctx, Customer existing, Customer updated) { return updated; }
        public Customer PostUpdate(ICompanyContext ctx, Customer entity) { return entity; }
        public Customer PreDelete(ICompanyContext ctx, Customer entity) { return entity; }
        public Customer PostDelete(ICompanyContext ctx, Customer entity) { return entity; }
    }
```

When you create an intervention proxy, you have to register it by hand in Startup.cs. See the Generated.OData.EF.API project for an example.
