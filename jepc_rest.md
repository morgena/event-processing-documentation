---
title: "JEPC Rest Service"
date: 2023-02-01T10:00:00+01:00
draft: false
weight: 4
---
In the following we assume that the server is running locally (`localhost`) and has been configured to listen on port `8080`. For Setup see [Lambda Engine: Setup]({{< ref "setup" >}}). The service provides several endpoints under the URL `http://localhost:8080/jepc`, which are explained in more detail below.

{{< hint type=note >}}
A test instance of the rest server is available at https://dbs-demo.mathematik.uni-marburg.de/jepc/
See https://dbs-demo.mathematik.uni-marburg.de/swagger-ui/index.html for more details.
{{< /hint >}}

### Ping ###
By calling `http://localhost:8080/jepc/` a simple ping request is executed. This serves only to ensure that the server is running. A ping request is answered with HTTP status 200/OK and contains no payload.

### Get Streams ###
This endpoint returns the names of all available streams as a JSON array. The URL is `http://localhost:8080/jepc/streams`. A response could look like this:
```JSON
[ "Stream_1", "Stream_2" ]
```

### Create Stream ###
This request creates a new empty stream with the given name and schema. The URL is `http://localhost:8080/jepc/streams`. Note that only HTTP/POST requests are allowed. If successful, the HTTP status 200/OK and an empty response will be returned. The request details are specified using a JSON object in the request body. The specification is as follows:
```
{
  "name": String; name of the stream,
  "schema": array of attribute specifications
}
```
The name of the stream must be unique system-wide. If a stream with this name already exists, a corresponding error message is returned. An attribute of the schema is specified as follows:
```
{
  "name": String;
  "type": String; attribute type (BOOLEAN, BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, GEOMETRY)
}
```

The name of the attribute must be unique within the schema. 

{{< hint type=note >}}
The timestamp attribute is defined implicitly and must not be `null`.
{{< /hint >}}

**Example**: The following request creates the stream `S` with the attributes `X` and `Y` of type `DOUBLE`:

```JSON
{
  "name": "S",
  "schema": [
    {
      "name": "X",
      "type": "DOUBLE"
    },
    {
      "name": "Y",
      "type": "DOUBLE"
    }
  ]
}
```

### Get Stream Schema ###
This endpoint returns the schema of a stream. The result is a JSON object with the schema. Each attribute consists of the name and the data type (`BOOLEAN, BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, GEOMETRY`). The URL is `http://localhost:8080/jepc/streams/<streamName>`. Only HTTP/GET requests are allowed.

**Example**: For the following request:
```
http://localhost:8080/jepc/streams/S
```

an answer could look as follows:
```JSON
[
  {
    "name": "X",
    "type": "DOUBLE"
  },
  {
    "name": "Y",
    "type": "DOUBLE"
  }
]
```

### Insert ###
This endpoint inserts events into an existing stream. The URL is `http://localhost:8080/jepc/insert`. Note that only HTTP/POST requests are allowed. If successful, the HTTP status 200/OK and an empty response will be returned. The events to be inserted are specified using a JSON object in the request body. The specification is as follows:

```
{
  "name": String; name of the stream into which the events are inserted.
  "events": Event Array; The events to be inserted.
}
```
`name` specifies the name of the stream to be inserted into, while `events` contains the records. An event is represented by a JSON object. The object consists of key/value pairs -- one for each attribute of the stream. The timestamp is defined by an additional attribute `tstart` of type `long`.

**Example**: The following query inserts two records into the stream `S` created in the previous example:
```JSON
{
  "name": "S",
  "events": [
    {
      "X": 42.0,
      "Y": 1337.0,
      "TSTART": 12
    },
    {
      "X": 17.0,
      "Y": 137.0,
      "TSTART": 14
    }
  ]
}
```

{{< hint type=note >}}
Please note that the Lambda-Engine manages the data in chronological order. The best insertion and query performance is achieved when the data is inserted in the correct order.
{{< /hint >}}

### Create Query ###
This endpoint creates a query with a specified name. The URL is `http://localhost:8080/jepc/queries`. Note that only HTTP/POST requests are allowed. The details of the request are specified using a JSON object in the request body. The specification is as follows:

```
{
  "name": String; The name of the query
  "query": String; The corresponding SQL query
}
```
The `query` parameter contains the SQL query (see [Lambda Engine: Query Language]({{< ref "query_language" >}}) for details on the query language). The response is a HTTP status OK/200 if a query is successfully registered. 

The resulting outputs of this query are event objects consisting of key-value pairs matching the query's output schema. Additionally, the object contains the fields `tstart` and `tend`, which represent the validity interval of the respective event.

{{< hint type=note >}}
Typically, only a single timestamp and no validity interval is used for events. In this case, `tend = tstart+1` applies. If you use windows in your queries, the validity interval reflects the window size.
{{< /hint >}}

**Example:** For a given stream `S` with the following schema:
```
  "X": "DOUBLE",
  "Y": "DOUBLE"
```
a  valid payload is:
```JSON
{
  "name" : "Q",
  "query": "SELECT (X+Y) AS XY FROM S"
}
```

### Get Query Schema ###
This endpoint returns the schema of a registred query. The result is a JSON object with the name of the query and the schema. The URL is `http://localhost:8080/jepc/queries/<queryName>`. Only HTTP/GET requests are allowed.

**Example:** For the above defined stream `S`, and query `Q`, the request HTTP/GET `http://localhost:8080/jepc/queries/Q`
results in:
```JSON
[
  {
    "name": "XY",
    "type": "DOUBLE"
  }
]
```

### Subscribe ###
This endpoint subscribes a sink to an existing query. The URL is `http://localhost:8080/jepc/queries/<queryName>/subscribe` and a HTTP/POST request has to be used. The HTTP status OK/200 response will be returned after successfully subscribing. The payload consists of an `url : String` targeted to a sink that accepts HTTP/POST requests containing the results of the specified query `<queryName>`.

**Example**: HTTP/POST `http://localhost:8080/jepc/queries/Q/subscribe`
```JSON
{
  "url" : "http://localhost:8080/chronicledb/insert"
}
```

The input: 
```JSON
{
  "name": "S",
  "events":  [
    { x: 1.0, y: 3.0, timestamp: 3},
    { x: 5.0, y: 8.0, timestamp: 6},
    { x: 12.0, y: 17.0, timestamp:7},
  ]
}
```
results in:
```JSON
{
  "name": "Q",
  "events":  [
    { XY: 4.0, tstart: 3, tend: 4},
    { XY: 13.0, tstart: 6, tend: 7},
    { XY: 29.0, tstart: 7, tend: 8}
  ]
}
```
The results of this subscribtion are sent to the subscribed url using HTTP/POST requests. 

### Unsubscribe ###
This endpoint unsubscribes a registred sink of a specific query.  The URL is `http://localhost:8080/jepc/queries/<queryName>/subscribe` and a HTTP/DELETE request has to be used. The HTTP status OK/200 response will be returned after successfully unsubscribing. The payload consists of the subscribed `url : String` to the specified `<queryName>`.

**Example**: HTTP/DELETE `http://localhost:8080/jepc/queries/Q/subscribe`
```JSON
{
  "url" : "http://localhost:8080/chronicledb/insert"
}
```

### Delete Query ###
This endpoint deletes a query and is given a name of an existing query as pathvariable.  The URL is `http://localhost:8080/jepc/queries/<queryName>` and a HTTP/DELETE request has to be used. The HTTP status OK/200 response will be returned after successfully deleting a query.

### Delete Streams ### 

This request removes a stream and is given a name of an existing stream as pathvariable. The URL is `http://localhost:8080/jepc/streams/<streamName>` and a HTTP/DELETE request has to be used. A stream can only be removed if there exist no consumers, consuming queries have to be removed before removing a stream. If a stream is successfully removed, a HTTP status OK/200 respond will be returned.
