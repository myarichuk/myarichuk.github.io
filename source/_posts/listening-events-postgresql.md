---
title: Listening to data changes in PostgreSQL and C#
date: 2019-12-02 09:23:25
tags:
  - PostgreSQL
  - C#
  - SQL
  - Event-handling
categories:
  - Programming
  - Databases
author: Michael Yarichuk
cover: /2019/12/02/listening-events-postgresql/post_icon.jpg
top_img: top.jpg
---
### To poll or not to poll?  
Nowadays, fast, responsive UI that reacts to data changes is pretty much a given (well, not always, yes, but often enough to talk about it!). If it is an application with just one user it is not that hard to implement, but what if there are multiple users in a system that affect each other?  
The answer would be... not to poll if you can, because more often than not there is another, better way. Many databases, both NoSQL and RDBMS offer functionality to push events to the connected clients.  

### So, what are we doing in this post?
RavenDB has awesome feature called [Data Subscriptions](https://ravendb.net/docs/article-page/4.2/csharp/client-api/data-subscriptions/what-are-data-subscriptions), which allow its client-api to receive near real-time notifications and changed data.  
I was curious to see how such feature works in other databases. The following is the result of my tinkering with PostgreSQL, I tried (and succeeded!) making PostgreSQL C# driver to receive events of any change of data in the database.

### Setting the server-side to send notifications about data changes
In order to set up change event listening, I will be using [NOTIFY](https://www.postgresql.org/docs/12/sql-notify.html))/[LISTEN](https://www.postgresql.org/docs/12/sql-listen.html) commands.
#### Setting up the database
I installed the latest version PostgreSQL server at the time of writing, version 12. (But you don't have to use version 12 - from what I've read, the following SQL code should work for any PostgreSQL that supports JSON natively)  
As a test dataset I have used a Northwind database from [this repository](https://github.com/pthom/northwind_psql), but any dataset can be used, really.

#### A function that will be used in change triggers
Any table we want to watch for changes in its data would have a trigger that would "forward" the change to a function that would use [NOTIFY statement](https://www.postgresql.org/docs/12/sql-notify.html) to listening clients.
The following is such function:  
```pgsql
CREATE FUNCTION public."NotifyOnDataChange"()
  RETURNS trigger
  LANGUAGE 'plpgsql'
AS $BODY$ 
DECLARE 
  data JSON;
  notification JSON;
BEGIN
  -- if we delete, then pass the old data
  -- if we insert or update, pass the new data
  IF (TG_OP = 'DELETE') THEN
    data = row_to_json(OLD);
  ELSE
    data = row_to_json(NEW);
  END IF;

  -- create json payload
  -- note that here can be done projection 
  notification = json_build_object(
            'table',TG_TABLE_NAME,
            'action', TG_OP, -- can have value of INSERT, UPDATE, DELETE
            'data', data);  
            
    -- note that channel name MUST be lowercase, otherwise pg_notify() won't work
    PERFORM pg_notify('datachange', notification::TEXT);
  RETURN NEW;
END
$BODY$;
```

#### Change triggers
Now, we will set up triggers to wire change events with **NotifyOnDataChange** function.
```pgsql
CREATE TRIGGER "OnDataChange"
  AFTER INSERT OR DELETE OR UPDATE 
  ON public.orders
  FOR EACH ROW
  EXECUTE PROCEDURE public."NotifyOnDataChange"();
```
That is nice, but what if we want notifications from all of the tables in the database?  
The following function will iterate over all tables and create triggers in each of them
```pgsql
CREATE FUNCTION public."CreateOnDataChangeForAllTables"()
  RETURNS void
  LANGUAGE 'plpgsql'
AS $BODY$
DECLARE  
  createTriggerStatement TEXT;
BEGIN
  FOR createTriggerStatement IN SELECT
    'CREATE TRIGGER OnDataChange AFTER INSERT OR DELETE OR UPDATE ON '
    || tab_name
    || ' FOR EACH ROW EXECUTE PROCEDURE public."NotifyOnDataChange"();' AS trigger_creation_query
  FROM (
    SELECT
      quote_ident(table_schema) || '.' || quote_ident(table_name) as tab_name
    FROM
      information_schema.tables
    WHERE
      table_schema NOT IN ('pg_catalog', 'information_schema')
      AND table_schema NOT LIKE 'pg_toast%'
  ) as TableNames
  LOOP
    EXECUTE  createTriggerStatement; --actually create the trigger
  END LOOP;
END$BODY$;
```
  
That's it! The server-side is ready to send notifications for data changes.

### Setting-up PostgreSQL client to receive events
For a client test app, I used .Net Core 3.0 project with [Npgsql data provider](https://www.nuget.org/packages/Npgsql/4.1.2).  
The following is code for listening for data changes.
```cs
class Program
{
  static async Task Main(string[] args)
  {
    const string connString = "<connection string>";

    await using var conn = new NpgsqlConnection(connString);
    await conn.OpenAsync();

    //e.Payload is string representation of JSON we constructed in NotifyOnDataChange() function
    conn.Notification += (o, e) => Console.WriteLine("Received notification: " + e.Payload);

    await using (var cmd = new NpgsqlCommand("LISTEN datachange;", conn))
      cmd.ExecuteNonQuery();

    while (true) 
      conn.Wait(); // wait for events
  }
}
```
  
There we have it. A simple way to listen for changes in PostgreSQL. 
Note, if the NpgsqlConnection client is not connected and changes happen, data change event will obviously be missed. Considering that RavenDB's [Data Subscriptions](https://ravendb.net/docs/article-page/4.2/csharp/client-api/data-subscriptions/what-are-data-subscriptions) allow handling "missed" events as well, I consider this a flaw of PostgreSQL notifications feature.