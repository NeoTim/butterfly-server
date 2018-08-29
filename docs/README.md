# Overview

This animation shows an [example](#example) application with three clients automatically sychronized with Butterfly Server .NET...

![Demo](https://raw.githubusercontent.com/firesharkstudios/butterfly-server-dotnet/master/img/demo.gif) 

Butterfly Server .NET provides...

- Ability to define a [Web API](#creating-a-web-api)
- Ability to define a [Subscription API](#creating-a-subscription-api) that allow pushing real-time data to clients
- Ability to modify, retrieve, and publish data change events on a [Database](#accessing-a-database)

Butterfly Server .NET targets *.NET Framework 2.0* and does **not** have any dependencies on ASP.NET.

# Getting Started

## Install from Nuget

| Name | Package | Install |
| --- | --- | --- |
| Butterfly.Core | [![nuget](https://img.shields.io/nuget/v/Butterfly.Core.svg)](https://www.nuget.org/packages/Butterfly.Core/) | `nuget install Butterfly.Core` |
| Butterfly.EmbedIO | [![nuget](https://img.shields.io/nuget/v/Butterfly.EmbedIO.svg)](https://www.nuget.org/packages/Butterfly.EmbedIO/) | `nuget install Butterfly.EmbedIO` |
| Butterfly.MySQL | [![nuget](https://img.shields.io/nuget/v/Butterfly.MySQL.svg)](https://www.nuget.org/packages/Butterfly.MySQL/) | `nuget install Butterfly.MySQL` |
| Butterfly.SQLite | [![nuget](https://img.shields.io/nuget/v/Butterfly.SQLite.svg)](https://www.nuget.org/packages/Butterfly.SQLite/) | `nuget install Butterfly.SQLite` |

## Install from Source Code

Get the source from [GitHub](https://github.com/firesharkstudios/butterfly-server-dotnet).

# Example

You can see an animation of running this example in the [Overview](#overview) section.

## Try It

Run this in a terminal or command prompt...

```
git clone https://github.com/firesharkstudios/butterfly-server-dotnet

cd butterfly-server-dotnet\Butterfly.Example.Todo.Server
dotnet run -vm
```

Run this in a second terminal or command prompt...

```
cd butterfly-server-dotnet\Butterfly.Example.Todo.Client
npm install
npm run dev
```

You should see http://localhost:8080/ open in a browser. Try opening a second browser instance at http://localhost:8080/. Notice that changes are automatically synchronized between the two browser instances.

There is also a [Cordova Todo Client](https://github.com/firesharkstudios/butterfly-server-dotnet/tree/master/Butterfly.Example.Todo.CordovaClient) and an [Electron Todo Client](https://github.com/firesharkstudios/butterfly-server-dotnet/tree/master/Butterfly.Example.Todo.ElectronClient).

## The Server

Here is all server code for our todo list manager...

```csharp
using System;

using Butterfly.Core.Util;

using Dict = System.Collections.Generic.Dictionary<string, object>;

namespace Butterfly.Example.HelloWorld.Server {
    class Program {
        static void Main(string[] args) {
            using (var embedIOContext = new Butterfly.EmbedIO.EmbedIOContext("http://+:8000/")) {
                // Create a MemoryDatabase (no persistence, limited features)
                var database = new Butterfly.Core.Database.Memory.MemoryDatabase();
                database.CreateFromText(@"CREATE TABLE todo (
	                id VARCHAR(50) NOT NULL,
	                name VARCHAR(40) NOT NULL,
	                PRIMARY KEY(id)
                );");
                database.SetDefaultValue("id", tableName => $"{tableName.Abbreviate()}_{Guid.NewGuid().ToString()}");

                // Listen for API requests
                embedIOContext.WebApi.OnPost("/api/todo/insert", async (req, res) => {
                    var todo = await req.ParseAsJsonAsync<Dict>();
                    await database.InsertAndCommitAsync<string>("todo", todo);
                });
                embedIOContext.WebApi.OnPost("/api/todo/delete", async (req, res) => {
                    var id = await req.ParseAsJsonAsync<string>();
                    await database.DeleteAndCommitAsync("todo", id);
                });

                // Listen for subscribe requests...
                // - The handler must return an IDisposable object (gets disposed when the channel is unsubscribed)
                // - The handler can push data to the client by calling channel.Queue()
                embedIOContext.SubscriptionApi.OnSubscribe("todos", (vars, channel) => {
                    return database.CreateAndStartDynamicView("SELECT * FROM todo", dataEventTransaction => channel.Queue(dataEventTransaction));
                });

                embedIOContext.Start();

                Console.ReadLine();
            }
        }
    }
}
```

The above C# code...
- Creates a Memory [database](#accessing-a-database) with a single *todo* table
- Defines a [Web API](#creating-a-web-api) to insert and delete *todo* records
- Defines a [Subscription API](#creating-a-subscription-api) to subscribe to a *todos* channel that retrieves all *todo* records **and** any changes to the *todo* records

See [Todo Server](https://github.com/firesharkstudios/butterfly-server-dotnet/tree/master/Butterfly.Example.Todo.Server) for the working server code.

## The Client

Now, let's see how a client might interact with this server using the Butterfly Client (`npm install butterfly-client`).

First, the client should maintain an open WebSocket to the server by using the *WebSocketChannelClient* class...

```js
let channelClient = new WebSocketChannelClient({
    url: `ws://${window.location.host}/ws`
});
channelClient.connect();
```

Next, the client will want to subscribe to a channel to receive data...

```js
let todosList = [];
channelClient.subscribe(
    new ArrayDataEventHandler({
        arrayMapping: {
            todo: todosList
        }
    }),
    'todos'
);
```

This subscription will cause the local *todosList* array to be synchronized with the *todo* records on the server.

Next, let's invoke a method on our API to add a new *todo* record (use whatever client HTTP library you wish)...

```js
$.ajax('/api/todo/insert', {
  method: 'POST',
  data: JSON.stringify({
    name: 'My First To-Do',
  }),
});
```

After the above code runs, the server will have a new *todo* record and a new *todo* record will automagically be sychronized from the server to the client's local *todosList* array.

See [Butterfly.Example.Todo.Client](https://github.com/firesharkstudios/butterfly-server-dotnet/tree/master/Butterfly.Example.Todo.Client) for a full working client based on Vuetify and Vue.

# Concepts

## Creating a Web Api

### Overview

[IWebApi](https://butterflyserver.io/docfx/api/Butterfly.Core.WebApi.IWebApi.html) allows defining a RESTlike API using HTTP verbs like this...

```cs
webApi.OnPost("/api/todo/insert", async (req, res) => {
    var todo = await req.ParseAsJsonAsync<Dict>();
    await database.InsertAndCommitAsync<string>("todo", todo);
});
webApi.OnPost("/api/todo/delete", async (req, res) => {
    var id = await req.ParseAsJsonAsync<string>();
    await database.DeleteAndCommitAsync("todo", id);
});
```

This is "RESTlike API" because it's not following the standard practice of using HTTP Verbs to define the actions (which is often problematic with entities with a large number of actions). 

You need an implementation of [IWebApi](https://butterflyserver.io/docfx/api/Butterfly.Core.WebApi.IWebApi.html) like [EmbedIO](#using-embedio).

## Creating a Subscription API

### Overview

[ISubscriptionApi](https://butterflyserver.io/docfx/api/Butterfly.Core.WebApi.ISubscriptionApi.html) allows defining a Subscription API that can push real-time data to clients like this...

```cs
subscriptionApi.OnSubscribe("todos", (vars, channel) => {
    return database.CreateAndStartDynamicViewAsync("todo", dataEventTransaction => {
	    channel.Queue(dataEventTransaction);
    });
});
```

In the example above, a subscription to the *todos* channel creates a *DynamicView* instance that pushes data changes over the channel to the client. 

You need an implementation of [ISubscriptionApi](https://butterflyserver.io/docfx/api/Butterfly.Core.WebApi.ISubscriptionApi.html) like [EmbedIO](#using-embedio).

### Defining Subscriptions

Example of a subscription returning multiple datasets and a dataset that uses a JOIN...

```cs
subscriptionApi.OnSubscribe("todo-page", async(vars, channel) => {
  var dynamicViewSet = database.CreateDynamicViewSet(dataEventTransaction => channel.Queue(dataEventTransaction);

  string userId = channel.Connection.AuthToken;

  // DynamicViews can include JOINs and will update if 
  // any of the joined tables change the resultset
  // (note this requires using a database like MySQL that supports JOINs)
  dynamicViewSet.CreateDynamicView(
    @"SELECT td.id, td.name, td.user_id, u.name user_name
    FROM todo td
      INNER JOIN user u ON td.user_id=u.id
    WHERE u.id=@userId",
    new {
      userId
    }
  );

  // A channel can return multiple resultsets as well
  dynamicViewSet.CreateDynamicView(
    @"SELECT id, name
    FROM tag
    WHERE user_id=@userId",
    new {
      userId
    }
  );

  return dynamicViewSet;
);
```

In this example, a client subscribing to *todo-page* will get a *todo* collection and a *tag* collection both filtered by user id.  

Because the new *todo* collection is the result of a join, the client will receive updates if changes to either of the underlying *todo* table or *user* table would change the resultset.

## Accessing a Database

### Overview

An [IDatabase](https://butterflyserver.io/docfx/api/Butterfly.Core.Database.IDatabase.html) instance allows creating transactions, modifying data, retrieving data, and subscribing to data change events.

```cs
var id = await database.InsertAndCommitAsync<string>("todo", new {
    name = "My Todo"
});
await database.UpdateAndCommitAsync("todo", new {
    id,
    name = "My New Todo"
});
await database.DeleteAndCommitAsync("todo", id);

var name = await database.SelectValueAsync<string>("SELECT name FROM todo", id);
```

The [IDatabase](https://butterflyserver.io/docfx/api/Butterfly.Core.Database.IDatabase.html) instance also support transactions and the ability to publish data change events on tables and even complex SELECT statements.


### Using a Memory Database

[Butterfly.Core.Database.MemoryDatabase](https://butterflyserver.io/docfx/api/Butterfly.Core.Database.Memory.MemoryDatabase.html) database is included in [Butterfly.Core](api/Butterfly.Core.md).

In your application...

```csharp
var database = new Butterfly.Core.Database.Memory.MemoryDatabase();
database.CreateFromText(@"CREATE TABLE todo (
	id VARCHAR(50) NOT NULL,
	name VARCHAR(40) NOT NULL,
	PRIMARY KEY(id)
);");
```

### Creating the Database

You can either create the database structure by...

- Executing CreateFromTextAsync() or CreateFromResourceAsync() in Butterfly Server .NET
- Creating the database yourself outside of Butterfly Server .NET

### Selecting Data

Call *IDatabase.SelectRowsAsync()* with alternative values for the *sql* parameter and alternative values for the *vars* parameter...

```cs
var sql1 = "employee"; // SELECT statement is auto generated
var sql2 = "SELECT * FROM employee"; // WHERE statement is auto generated 
var sql3 = "SELECT * FROM employee WHERE id=@id"; // Specify exact SELECT statement

var vars1 = "123"; // Can specify just the primary key value
var vars2 = new { id = "123" }; // Can specify parameters using an anonymous type
var vars3 = new Dictionary<string, object> { { "id", "123" } }; // Can specify parameters using a Dictionary
 
// Any combination of sql1/sql2/sql3 and vars1/vars2/vars3 would yield identical results
Dictionary<string, object>[] rows = await database.SelectAsync(sql1, vars1);
```

Retrieve multiple rows, a single row, or a single value...

```cs
// Retrieve multiple rows
Dictionary<string, object>[] rows = await database.SelectRowsAsync("SELECT * FROM employee");

// Retrieve a single row 
Dictionary<string, object> row = await database.SelectRowAsync("SELECT * FROM employee", "123")

// Retrieve a single value
string name = await database.SelectValueAsync<string>("SELECT name FROM employee", "123")
```

The WHERE clause will be auto-generated or rewritten in specific scenarios...

```cs
// Executes WHERE department_id = '123'
Dictionary<string, object>[] rows = await database.SelectRowsAsync("employee", new {
	department_id = "123"
});

// Executes WHERE department_id IS NULL
Dictionary<string, object>[] rows = await database.SelectRowsAsync("employee", new {
	department_id = (string)null
});

// Executes WHERE department_id IS NOT NULL
Dictionary<string, object>[] rows = await database.SelectRowsAsync(
	"SELECT * employee WHERE department_id!=@did", 
	new {
		did = (string)null
	}
);

// Executes WHERE department_id IN ('123', '456')
Dictionary<string, object>[] rows = await database.SelectRowsAsync("employee", new {
	department_id = new string[] { "123", "456"}
});

// Executes WHERE department_id NOT IN ('123', '456')
Dictionary<string, object>[] rows = await database.SelectRowsAsync(
	"SELECT * employee WHERE department_id!=@did", 
	new {
		did = new string[] { "123", "456"}
	}
);
```
### Modifying Data

```cs
// Execute a single INSERT and return the value of the primary key
string id = database.InsertAndCommitAsync<string>("employee", new {
	first_name = "Jim",
	last_name = "Smith",
	balance = 0.0f,
});

// Assuming the employee table has a unique index on the id field, 
// this updates the balance field on the matching record
database.UpdateAndCommitAsync<string>("employee", new {
	id = "123",
	balance = 0.0f,
});

// Assuming the employee table has a unique index on the id field, 
// this deletes the matching record
database.DeleteAndCommitAsync<string>("employee", "123");

```
### Transactions

```cs
// If either INSERT fails, neither INSERT will be saved
using (ITransaction transaction = await database.BeginTransactionAsync()) {
	string departmentId = transaction.InsertAsync<string>("department", new {
		name = "Sales"
	});
	string employeeId = transaction.InsertAsync<string>("employee", new {
		name = "Jim Smith",
		department_id = departmentId,
	});
	await transaction.CommitAsync();
}
```
### Synchronizing Data

Use *ITransaction.SynchronizeAsync* to determine the right INSERT, UPDATE, and DELETE statements to synchronize two collections...

```cs
// Assumes an article_tag table with article_id and tag_name fields
public async Task SynchronizeTag(string articleId, string[] tagNames) {
	// First, create the existingRecords from the database
	Dict[] existingRecords = database.SelectRowsAsync(
		@"SELECT article_id, tag_name 
		FROM article_tag 
		WHERE article_id=@articleId",
		new {
			articleId
		}
	);

	// Next, create the newRecords collection from the tagNames parameter
	Dict[] newRecords = tagNames.Select(x => new Dictionary<string, object> {
		{ "article_id", articleId },
		{ "tag_name", x },
	});

	// Now, execute the right INSERT, UPDATE, and DELETE statements to make
	// the newRecords collection match the existingRecords collection
	using (ITransaction transaction = database.BeginTransactionAsync()) {
		bool changed = await transaction.SynchronizeAsync(
			"article_tag", 
			existingRecords, 
			newRecords, 
			existingRecord => new Dictionary<string, object> {
				article_id = existingRecord.GetAs("article_id", (string)null),
				tag_name = existingRecord.GetAs("tag_name", (string)null),
			},
		);
	}
}
```

### Defaults, Overrides, and Preprocessors

Can be defined globally or per table...

```cs
// Add an id field to all INSERTs with values like at_58b5fff4-322b-4fe8-b45d-386dac7a79f9
// if INSERTing on an auth_token table
database.SetDefaultValue(
    "id", 
    tableName => $"{tableName.Abbreviate()}_{Guid.NewGuid().ToString()}"
);

// Add a created_at field to all INSERTS with the current time
database.SetDefaultValue("created_at", tableName => DateTime.Now.ToUnixTimestamp());

// Remap any DateTime values to UNIX timestamp values
database.AddInputPreprocessor(BaseDatabase.RemapTypeInputPreprocessor<DateTime>(
    dateTime => dateTime.ToUnixTimestamp()
));

// Remap any $NOW$ values to the current UNIX timestamp
database.AddInputPreprocessor(BaseDatabase.RemapTypeInputPreprocessor<string>(
    text => text=="$NOW$" ? DateTime.Now.ToUnixTimestamp().ToString() : text
));

// Remap any $UPDATE_AT$ values to be the same value as the updated_at field
database.AddInputPreprocessor(BaseDatabase.CopyFieldValue("$UPDATED_AT$", "updated_at"));
```

### Dynamic Views

Dynamic views allow receiving both the initial data and data change events when the data changes.

```cs
// Create a DynamicViewSet that simply prints all data events to the console
using (DynamicViewSet dynamicViewSet = database.CreateDynamicViewSet(
	dataEventTransaction => Console.WriteLine(dataEventTransaction)
) {
	// Create dynamic view for all employees in department
	dynamicViewSet.CreateDynamicView("employee", new {
		department_id = "123"
	});

	// Create dynamic view for all resources in department
	dynamicViewSet.CreateDynamicView("resource", new {
		department_id = "123"
	});

	// This will cause each DynamicView above to execute
	// and all the initial data in each DynamicView to be
	// echoed to the console in a single DataEventTransaction
	dynamicViewSet.Start();

	// This will cause a new DataEventTransaction with a single
	// INSERT event to be echoed to the console
	database.InsertAndCommitAsync("employee", new {
		name = "Joe Smith",
		department_id = "123",
	});

	// This will NOT cause a new DataEventTransaction to
	// be printed to the console because this INSERT
	// does not change the DynamicView results above
	database.InsertAndCommitAsync("employee", new {
		name = "Joe Smith",
		department_id = "456",
	});
}
```

## Implementations

### Using EmbedIO

[EmbedIO](https://github.com/unosquare/embedio) is a capable low footprint web server that can be used to implement both the *IWebApi* and *ISubscriptionApi* interfaces. 

The *EmbedIOContext* class is a convenience class that creates IWebApi and ISubscriptionApi instances from a running EmbedIO web server.

In the *Package Manager Console*...

```
Install-Package Butterfly.EmbedIO
```

In your application...

```csharp
var embedIOContext = new Butterfly.EmbedIO.EmbedIOContext("http://+:8000/");

// Declare your Web API and Subscription API like...
embedIOContext.WebApi.OnPost("/api/todo/insert", async (req, res) => {
   // Do something
});
embedIOContext.WebApi.OnPost("/api/todo/delete", async (req, res) => {
   // Do something
});
embedIOContext.SubscriptionApi.OnSubscribe("todos", (vars, channel) => {
   // Do something
});

embedIOContext.Start();
```

### Using MySQL

In the *Package Manager Console*...

```
Install-Package Butterfly.MySql
```

In your application...

```csharp
var database = new Butterfly.MySql.MySqlDatabase("Server=127.0.0.1;Uid=test;Pwd=test!123;Database=butterfly_db_demo");
```

### Using SQLite

In the *Package Manager Console*...

```
Install-Package Butterfly.SQLite
```

In your application...

```csharp
var database = new Butterfly.SQLite.SQLiteDatabase("Filename=./my_database.db");
```

# API Documentation

Click [here](https://butterflyserver.io/docfx/api/) for the API Documentation

# In the Wild

[Build Hero](https://www.buildhero.io) is a collaborative tool for general contractors, subcontractors, and customers to collaborate on remodel projects.  The [my.buildhero.io](https://my.buildhero.io) site, the Android app, and the iOS app are all powered by Butterfly Server .NET.

# Similar Projects

- [dotNetify](https://github.com/dsuryd/dotNetify)
- [SignalR](https://github.com/SignalR/SignalR)
- [SignalW](https://github.com/Spreads/SignalW)

# Contributing

If you'd like to contribute, please fork the repository and use a feature
branch. Pull requests are warmly welcome.

# Licensing

The code is licensed under the [Mozilla Public License 2.0](http://mozilla.org/MPL/2.0/).