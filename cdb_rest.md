In the following we assume that the server is running locally (`localhost`) and has been configured to listen on port `8080`. For Setup see [Lambda Engine: Setup]({{< ref "setup" >}}). This service provides several endpoints under the URL `http://localhost:8080/chronicledb`, to access the persistance layer (ChronicleDB), which are explained in more detail below.

{{< hint type=note >}}
A test instance of the rest server is available at https://dbs-demo.mathematik.uni-marburg.de/chronicledb/
See https://dbs-demo.mathematik.uni-marburg.de/swagger-ui/index.html for more details.
{{< /hint >}}

### Ping ###
By calling `http://localhost:8080/chronicledb/` a simple ping request is executed. This serves only to ensure that the server is running. A ping request is answered with HTTP status 200/OK and contains no payload.

### Get Streams ### 
This endpoint returns the names of all available streams as a JSON array. The URL is `http://localhost:8080/chronicledb/streams`. A response could look like this:
```
[ "Stream_1", "Stream_2" ]
```

### Create Stream ###
This request creates a new empty stream with the given name and schema. The URL is `http://localhost:8080/chronicledb/streams`. Note that only HTTP/POST requests are allowed. If successful, the HTTP status 200/OK and an empty response will be returned. The request details are specified using a JSON object in the request body. The specification is as follows:
```
{
  "name": String; name of the stream,
  "schema": array of attribute specifications
}
```
The name of the stream must be unique system-wide. If a stream with this name already exists, a corresponding error message is returned. An attribute of the schema is specified as follows {anchor #schema}:
```
{
  "name": String; attribute name
  "type": String; attribute type (BOOLEAN, BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, GEOMETRY)
  "properties": Map; additional properties (see below)
}
```

The name of the attribute must be unique within the schema. In addition to name and type, two properties are currently supported for attributes:
1. `nullable:Boolean`: Indicates whether `null` values are allowed for the attribute.
2. `index:Boolean`: Indicates whether lighweight indexing is applied to the attribute.

Lightweight indexes are essentially materialized aggregates that are managed in the inner nodes of the tree structure of the primary index. Under certain conditions, they can speed up request processing (for detailed information about this technique, see here: ). Currently, lightweight indexing is supported for numeric and geometry attributes. Depending on the type, the following aggregates are managed:

- numeric: Count, Min, Max, Sum
- geometry: Count, Bounding Box

{{< hint type=note >}}
The timestamp attribute is defined implicitly and must not be `null`.
{{< /hint >}}

**Example**: The following request creates the stream `S` with the attributes `X` and `Y` of type `DOUBLE`. A lightweight index is created on both attributes and they must not be `null`:

```JSON
{
  "name": "S",
  "schema": [
    {
      "name": "X",
      "type": "DOUBLE",
      "properties": {
        "nullable": false,
        "index": true
      }
    },
    {
      "name": "Y",
      "type": "DOUBLE",
      "properties": {
        "nullable": false,
        "index": true
      }
    }
  ]
}
```

{{< hint type=note >}}
The additional `properties` are optional. If they are not specified, no lightweight index is created and the attribute is **not** nullable.
{{< /hint >}}


### Delete Streams ### 
This request removes a stream and is given a name of an existing stream as parameter. The URL is `http://localhost:8080/chronicledb/streams/<streamName>` and a HTTP/DELETE request has to be used.


### Stream Info ###
This endpoint returns information about a stream. The result is a JSON object with the name of the stream, the number of events stored, the time range covered, and the schema. The URL is `http://localhost:8080/chronicledb/streams/<streamName>/info`. Only HTTP/GET requests are allowed.

**Example**: For the following request:
```
http://localhost:8080/chronicledb/streams/S/info
```

an answer could look as follows:
```JSON
{
  "name": "S",
  "eventCount": 1337,
  "timeInterval": {
    "lower": 0,
    "upper": 8192,
    "lowerInclusive": true,
    "upperInclusive": true
  },
  "schema": [
    {
      "name": "X",
      "type": "DOUBLE",
      "properties": {
        "nullable": "false"
      }
    },
    {
      "name": "Y",
      "type": "DOUBLE",
      "properties": {
        "nullable": "false"
      }
    }
  ]
}
```

### Insert ###
This endpoint inserts events into an existing stream. The URL is `http://localhost:8080/chronicledb/insert`. Note that only HTTP/POST requests are allowed. If successful, the HTTP status 200/OK and an empty response will be returned. The events to be inserted are specified using a JSON object in the request body. The specification is as follows:

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
Please note that ChronicleDB manages the data in chronological order. The best insertion and query performance is achieved when the data is inserted in the correct order.
{{< /hint >}}

### Query ###
This endpoint executes SQL statements. The URL is `http://localhost:8080/chronicledb/queries`. Note that only HTTP/POST requests are allowed. The details of the request are specified using a JSON object in the request body. The specification is as follows:

```
{
  "query": String; SQL-Query,
  "startTime": long; lower bound timestamp (inclusive, optional),
  "endTime": long; upper bound timestamp (inclusive, optional)
}
```
The two parameters `startTime` and `endTime` restrict the request to the specified time range and are optional. The `query` parameter contains the SQL query (see [[public/projects/event-processing/query-language/ | Lambda Engine: Query Language]] for details on the query language). The response to a query is a JSON array containing event objects. An event object consists of key-value pairs matching the query's output schema. Additionally, the object contains the fields `tstart` and `tend`, which represent the validity interval of the respective event.

{{< hint type=note >}}
Typically, only a single timestamp and no validity interval is used for events. In this case, `tend = tstart+1` applies. If you use windows in your queries, the validity interval reflects the window size.
{{< /hint >}}

**Example:** For a given stream `S` with the following schema:
```
  "X": "DOUBLE",
  "Y": "DOUBLE"
```
and events:
```JSON
[
  { x: 1.0, y: 3.0, timestamp: 3},
  { x: 5.0, y: 8.0, timestamp: 6},
  { x: 12.0, y: 17.0, timestamp:7},
  { x: 17.0, y: 21.0, timestamp:9}
]
```
the query:
```JSON
{
  "query": "SELECT (X+Y) AS XY FROM S",
  "startTime": 5,
  "endTime": 7
}
```
results in:
```JSON
[
  { XY: 13.0, tstart: 6, tend: 7},
  { XY: 29.0, tstart: 7, tend: 8}
]
```

### Schema ###
This endpoint returns the output schema for any SQL statement. The URL is `http://localhost:8080/chronicledb/queries/schema`. Note that only HTTP/POST requests are allowed. The result is a JSON array with the attributes of the output schema. Each attribute consists of the name and the data type (`BOOLEAN, BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, GEOMETRY`). The request body is the same as for the **Query** endpoint, but the parameters `startTime` and `endTime` can be omitted. 

**Example**: Based on the **Query** endpoint example, for the following query:
```JSON
{
  "query": "SELECT (X+Y) AS XY FROM S"
}
```
the schema endpoint returns:
```JSON
[
  {
    "name": "XY",
    "type": "DOUBLE" 
  }
]
```

### Register Queries ###
This endpoint registers SQL statements to a specified Id. The URL is `http://localhost:8080/chronicledb/queries/registered`. Note that only HTTP/POST requests are allowed. The details of the request are specified using a JSON object in the request body. The specification is as follows:

```
{
  "queryId": String; Query-Id (query name),
  "query": String; SQL-Query
}
```
The `queryId` paramter specifies the unique Id of the registered query. The `query` parameter contains the SQL query (see [[public/projects/event-processing/query-language/ | Lambda Engine: Query Language]] for details on the query language).

**Example:** For a given stream `S` a valid register query request is:
```JSON
{
  "queryId": "aQuery"
  "query": "SELECT * FROM S"
}
```

### Query registered queries ###
This endpoint queries a registered query by a specified `queryId`. The URL is `http://localhost:8080/chronicledb/queries/registered/<queryId>`. Note that only HTTP/POST requests are allowed. The details of the request are specified using a JSON object in the`request body. The specification is as follows:

```
{
  "startTime": long; lower bound timestamp (inclusive, optional),
  "endTime": long; upper bound timestamp (inclusive, optional)
}
```
The `queryId` paramter referencs the unique identifier of a query. The two parameters `startTime` and `endTime` restrict the request to the specified time range and are optional. The response to a query is a JSON array containing event objects. An event object consists of key-value pairs matching the query's output schema. Additionally, the object contains the fields `tstart` and `tend`, which represent the validity interval of the respective event.


### Delete registered queries ###
This endpoint deletes a registered query specified by a unique `queryId`. The URL is `http://localhost:8080/chronicledb/queries/registered/<queryId>`. Note that only HTTP/DELETE requests are allowed. 

### Get registered query schema ###
This endpoint computes the schema of a registered SQL statement for a specified Id. The URL is `http://localhost:8080/chronicledb/queries/registered/<queryId>`. Note that only HTTP/GET requests are allowed. The response is the schema of the registered query, see `Create Stream`.

### Get registered queries ###
This endpoint returns a list of all registered `queryId`s and corresponding `queryString`s. The URL is `http://localhost:8080/chronicledb/queries/registered`. Note that only HTTP/GET requests are allowed. The response is a list of the registered queryIds and their queryString.

**Example:** For two registered queris `aQuery`, `bQuery`, and two streams `S`, `T`, the response of this endpoint is:
```
{
  {
    "queryId": "aQuery",
    "query": "select * from S"
  },
  {
    "queryId": "bQuery",
    "query": "select * from T"
  }
}
```
