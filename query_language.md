# Data Model #  

The Lambda Engine and its core components JEPC and ChronicleDB are dedicated event stream processing and database systems specialized for storing and querying high volume event data, which are inherently ***temporal*** in nature. The basic underlying data model of events requires that data items can be expressed in following form:

> Event: (att1, ..., attN, time_start, time_end)

In essence an **event** has a fixed schema of N attributes in addition to a temporal information, which is a half open time interval [time_start, time_end) for which time_start is usually used to express the time of occurrence an event.

A **stream** in ChronicleDB is a collection of events with the same schema and can be considered akin to a table in a database system. The timestamps cannot and should not be specified as part of the schema. ChronicleDB does support out-of-order events, i.e., events do not necessarily have to be inserted by ascending value of time_start, but a high degree of out-of-order insertions results in performance degradation.

{{< hint type=note >}}
Due to the temporal nature of ChronicleDB it is similar to a time series database. However, since events such as streams from sensor boxes often carry multiple values per time-stamp, ChronicleDB inherently works as a multi-variate time series database. Furthermore, there is no requirement/support for ticks, missing value interpolations or similar concepts.
{{< /hint >}}

We currently support the following types of attributes, which need to be specified in the **schema** when creating a new stream in ChronicleDB:

```
BOOLEAN, BYTE, SHORT, INTEGER, LONG, FLOAT, DOUBLE, STRING, GEOMETRY
```
The GEOMETRY type is used for spatial data processing and may contain 2D Points, LineStrings or Polygons. Geometry constants can be specified using the widely used WKT format.

# Example Queries #

Since ChronicleDB is a dedicated event database system, the main query usage is either as batch-oriented event stream processing system or for post-mortem analysis. Therefore, the main query types follow typical SQL-like semantics also found in livestream event processing systems, i.e. a basic `SELECT ... FROM ... WHERE ...` structure.

In the following, we will present the basic syntax for the main query operators in ChronicleDB through a couple of example queries. For those queries, we assume ChronicleDB has two streams, each specified with a stream name and a schema composed of attribute names followed by their respective type after the colon: 

> Name=CarStream; Schema={Id:LONG, Speed:DOUBLE, Model:STRING, Position:GEOMETRY}

> Name=TrafficLightsStream; Schema={Id:LONG, Status:STRING, Position:GEOMETRY}

We assume that each stream is composed of multiple unique car and traffic light sensors which sent their events to ChronicleDB.

{{< hint type=warning >}}
Since ChronicleDB is currently not stable, this is in no way, shape or form intended to be a comprehensive up-to-date specification for our query language. There may be enabled features we do not choose to list. The query example may not work in the future. Or in the present. If you have any questions, contact the person responsible (listed below).
{{< /hint >}}

## Filter ##

The basic structure of a ChronicleDB query can be exemplified by the following query:
```
SELECT Id, Speed
FROM CarStream
WHERE Speed > (30+5) AND Speed < 50
```
In this query, we analyze the stored stream for car events that exhibit a speed value over 35 and under 50 which comprise the query results. Furthermore, we can use the select clause for projections. In this case, we only forward Id & Speed while discarding Model & Position.

{{< hint type=note >}}
Based on the ChronicleDB data model, each event has temporal validity expressed through timestamps. These timestamp are not only implicitly part of each //stored// event without specifying them in the schema, but results also always include them. The behavior carries over from the events processing nature of ChronicleDB, where an event stream queries output is per default an event stream, and thus, has to carry temporal information.
{{< /hint >}}

You can also use the following predicates within a where clause:
```
<, >, =, >=, <=, !=, <>, LIKE, AND, OR, FALSE, TRUE
```
As seen with the (30+5) statement in the query above, those predicates can use arithmetic expressions.  An exhaustive list of available expressions can be found [here](#expressions).

## Windows ##

One of the most important concepts for event stream processing is windowing. A window can be viewed as temporal grouping of events and is used to limit the amounts of events a live stream processing engine has to view. Effectively, a window extends an events lifetime by the specified length of the window.

A time window considering the last 5 minutes would be initiated by the following clause:
```
WINDOW (TIME 5 MINUTES)
```
It is possible to use granularities between `MILLISECONDS` and `WEEKS`. Meanwhile, a count window considering the last 100 number of events would be specified as follows:
```
WINDOW (COUNT 100 EVENTS)
```
Both temporal and numerical specification can be followed by an optional `JUMP` clause in the same domain (time or count). This would determine how many time units or events pass between the following window evaluation.

{{< hint type=note >}}
Windows have no tangible effect for pure filter queries and are most important for the following Aggregation, Join and Pattern Matching queries.
{{< /hint >}}

{{< hint type=important >}}
It is strongly advised to use windows only in conjunction with a input stream relation stored in ChronicleDB. Windows in arbitrary positions within nested queries are not well-defined and can lead to unexpected behavior and/or errors.
{{< /hint >}}

## Aggregation ##

Aggregations can be specified within the SELECT clause and are typically used to aggregate informations over previously defined windows.
```
SELECT Id, AVG(Speed)
FROM CarStream
WHERE Model="DMC DeLorean"
GROUP BY Id
WINDOW (TIME 5 MINUTES)
```
The above examples replays the CarStream stream with an applied 5 minute window, progressing the window 1 event at a time. The `WHERE` clause results in a filter query that only selects the specified car models. For those models, ChronicleDB will compute the average speed within a time window. Furthermore, the query features the optional `GROUP BY` clause, meaning each unique car instance as derived from the Id Attribute will be evaluated separately.

{{< hint type=note >}}
The amount of results reported for an aggregate will be automatically determined by changes in the underlying events of which the aggregate is comprised of. That means, if the window jumps forward but no event expires or is added due to the jump, there will not be a new result. Instead, the previous result's temporal validity will be extended.
{{< /hint >}}

{{< hint type=important >}}
It is probably currently not possible to use aggregates without a window. I guess.
{{< /hint >}}

Results of aggregates can be filtered by using the `HAVING` clause as known from standard SQL:
```
SELECT Id, AVG(Speed)
FROM CarStream
WHERE Model="DMC DeLorean"
GROUP BY Id
HAVING AVG(Speed) > 88
WINDOW (TIME 5 MINUTES)
```
This query outputs only time-travelling DeLoreans, indicated by an average speed of more than 88 mph over the past 5 minutes.

A list of all available aggregates can be found [here](#aggregates).

## Join ##
Joins operate on exactly two input streams and produce combined records of temporally overlapping events of each respective stream.
```
SELECT *
FROM 
  (SELECT * FROM CarStream WINDOW (TIME 10 SECONDS)) Cars,
  (SELECT * FROM TrafficLightsStream WINDOW (TIME 10 SECONDS)) TrafficLights
WHERE TrafficLights.STATUS="RED" AND ST_DISTANCE(TrafficLights.Position, Cars.Position) < 0.5
```
The above example aims to find cars potentially crossing red lights by combining the CarStream with the TrafficLightsStream. We filter out TrafficLight Events that are not red. (Yes, you could push that filter down. Have a cookie. :cookie:). Then we restrict the temporally overlapping of crossproduct with a spatial predicate to indicate spatial closeness in addition to the temporal overlap. Finally, we select every attribute in the combined event. (TODO: I think the framework does something to automatically to resolve the duplicate position naming of both stream attributes that will appear in the result, but I'm not sure what.)

{{< hint type=important >}}
It is probably currently not possible to use joins without the respective amount of windows. I guess.
{{< /hint >}}

{{< hint type=warning >}}
Joins SHOULD work, but I'm not sure we ever actually used them. So there are probably bugs. Or it straight up does not work.
{{< /hint >}}

Alternate syntax for joins, as in standard SQL, is also possible:
```CarStream l JOIN CarStream r using ID```
or
```CarStream l JOIN CarStream r on l.ID = r.ID```

## Extrapolation ##
ChronicleDB allows to extrapolate events into the future via an extrapolation clause. The extrapolation clause directly follows the from-clause:
```
SELECT * 
FROM CarStream
EXTRAPOLATE(
  5 MINUTES
  DEDUP=TRUE
  IF ID = 5
  Model=lvcf
  SPEED=linear
  POSITION=linear
  PARTITION BY ID
)

```
The extrapolation operator forwards all events to downstream operators and generates additional events, after the input stream has ended. In our example, the operator extrapolates an event five minutes from the end (`5 MINUTES`) of the input stream for each car (`PARTITION BY`). We specify linear extrapolation for the fields `Speed` and `Position`, and use the last known value for the `Model` attribute. If no extrapolation method is specified for an attribute, we fall back to `lvcf` (Last known value). Further, there are two optional parameters for this operator:

- **DEDUP**: Specifies, if deduplication should be applied to incoming events. That is, if the payload of two consecutive events is equal, only the first is stored. This is especially useful for linear extrapolation, which relies on the values of two past events. If both events are equal, linear extrapolation will behave as `LVCF` which may not be the intended behavior.
- **IF**: Allows to specify a condition that determines if extrapolation is applied for a partition or not.

## Sequential Pattern Matching ##

Sequential Pattern Matching is one of the most unique operators in Event Processing and until the SQL Standard 2016 had no equivalent database operation. The general idea is to search for a consecutive sequence of events following a specified regular expression comprised of boolean predicates. In ChronicleDB, we utilize the MATCH_RECOGNIZE clause to specify patterns.
```
SELECT * 
FROM CarStream
MATCH_RECOGNIZE(
 MEASURES X.Id AS Id, X.Position AS StartPos, Z.Position AS EndPos
 PATTERN X Y+ Z
 DEFINE
   X AS Speed < 50,
   Y AS PREV(Speed) < Speed,
   Z AS Speed > 100
 PARTITION BY Id
 WITHIN 2 MINUTES
)
```
The above query identifies cars which accelerate their speed from below 50 to above 100 without any deceleration in-between within a time frame of 2 minutes. 

This is accomplished within the `MATCH_RECOGNIZE` clause. `MEASURES` is used to specify the output of the pattern matching query and can capture the state of any event which contributes to a match. In this case, we capture Ids and position attributes of the first and last event. The `PATTERN`clause is used to define the regular expression while `DEFINE` specifies what the parts within `PATTERN` mean. In the example, X is associated with the predicate Speed < 50. Thus, each event fulfilling this expression will be viewed as an X by ChronicleDB. As an example, if within a stream there are 5 events that can be viewed X Y Y Y Z respectively, this matches the defined regular expression and would return a result. In this case, the position of the first event (X) and the last event (Z) would be returned. Similar to `GROUP BY`for aggregates, `PARTITION BY` is used for partition matching queries to isolate events within a stream based on attributes. `WITHIN` is basically a time window.

## Quantifiers ##
With Quantifier you can determine how often a symbol should occure in a match for example, `X Y* Z`. ChronicleDB supports following quantifiers within a regular expression:
| Quantifier | Meaning 
| -----  | ----- 
| `*`   | 0 or more. 
| `+`   | 1 or more. 
| `?`   | 0 or 1. 
| `{n}`   | exactly n. 
| `{n,}`   | n or more.
| `{n, m}`   | n to m. 

{{< hint type=caution >}}
The quantifier in ChronicleDB are reluctant and not greedy, which means the quantifier tries to match as little as possible.
{{< /hint >}}

## Predicates in Define clause ##
Predicates within the `DEFINE` clause can use special boolean predicates.

- `PREV(Column [, <offset> ])`: This indicates a comparison with the event directly preceding the current event within the same partition. `PREV` can also be used with an offset. E.g., `PREV(Speed, 5)` would effectively jump back 5 positions in the stream and use that event's speed value as a point of comparison. if the stream cannot jump back 5 events (e.g., at the start of a stream), this is an illegal state and will result in an exception. You need to make sure there are enough preceding symbol for it to work.
- `NEXT(Column [, <offset> ])`: The `NEXT` predicate indicates a comparison with the event directly following the current event within the same partition. As with PREV, an offset is possible. E.g. `NEXT(Speed, 3)` would effectively jump forward 3 positions in the stream and use that event's speed value as a point of comparison. 
- `FIRST(<symbol>.column [, <offset> ])`: The `FIRST` predicate offers the possibility of a comparison with the first event which was bound to a specific symbol within the same matching attempt. `FIRST(Y.Speed)` uses the first event's speed value bound to the symbol Y as a point of comparison. If no value is bound to the symbol Y `null` is returned. An offset is possible, e.g. `FIRST(Y.Speed, 4)` uses the fourth event's speed bound to the symbol Y. 
- `LAST(<symbol>.column [, <offset> ])`: The `LAST` predicate functions the same way as the `FIRST` predicate. In contrast to first, the last event that was bound to a specific symbol within in the same matching attemp is taken instead of the first. `LAST(Y.Speed)` uses the last event's speed value bound to the symbol Y as point of comparison. As with `FIRST` if no value is bound to the symbol Y `null` is returned. An offset is possible, e.g. `LAST(Y.Speed, 3)` uses the third last event's speed bound to the symbol Y. 

```
SELECT * 
FROM CarStream
MATCH_RECOGNIZE(
 MEASURES X.Id AS Id, FIRST(X.Speed) AS StartSpeed, LAST(X.Speed) AS PeekSpeed
 PATTERN X{10,} Z
 DEFINE
   X AS NEXT(Speed) > Speed,
   Y AS LAST(X.Speed) > Speed
 PARTITION BY Id
 WITHIN 2 MINUTES
)
```
This query finds all acceleration phases of the car, which consists of at least 10 consecutive measurements. Notice that by using `LAST(X.Speed) > Speed`, you can bypass the reluctant property of the quantifier but of course it needs an extra event which fulfills the condition.

## AFTER MATCH SKIP ##
The `AFTER MATCH SKIP` clause can be used to limit the search for new matches after a match has been found.

- `TO NEXT ROW (default)`: When using `TO NEXT ROW`the matching continues after the first row of the found match which can result in overlapping matches.
- `PAST LAST ROW`: When using `PAST LAST ROW`, the matching continues after the last row of the found match. As a result, no overlapping matches will be returned.
- `TO FIRST <symbol>`: Continues pattern matching at the first row mapped to the given symbol. If jumping to the first row of the found match an exception is thrown. This allows overlapping matches.
- `TO LAST <symbol>`: Continues pattern matching at the last row mapped to the given symbol. As with `TO FIRST`, if jumping to the first row of the found match an exception is thrown. This allows overlapping matches.

```
SELECT * 
FROM CarStream
MATCH_RECOGNIZE(
 MEASURES X.Id AS Id, FIRST(X.Speed) AS StartSpeed, LAST(X.Speed) AS PeekSpeed 
 AFTER MATCH SKIP PAST LAST ROW 
 PATTERN X{10,} Z
 DEFINE
   X AS NEXT(Speed) > Speed,
   Y AS LAST(X.Speed) > Speed
 PARTITION BY Id
 WITHIN 2 MINUTES
)
```

This query differs only by the adding `AFTER MATCH SKIP PAST LAST ROW` to the query shown above. In this case, no overlapping patterns are found, filtering out redundant information about the car's acceleration phases. 

## ALL ROWS PER MATCH and ONE ROW PER MATCH ##
To specify the output in ChronicleDB these two options are available:
- `ONE ROW PER MATCH (default)`: The default output of a found match in chronicleDB is one event, which contains the information about the match, depending on the output specified in the measure clause. 
- `ALL ROWS PER MATCH`: If more information is needed about the individual events that are part of a match, the `ALL ROWS PER MATCH` clause returns all events of the match including the measure attributes and the input schema of the event. It is possible that a row occurs more than once due to the selected `AFTER MATCH SKIP` condition. To find out more about the association, for example which event belongs to which symbol, Measure functions can also be helpful when using `ALL ROWS PER MATCH`. 

```
SELECT Id, Speed, StartSpeed, PeekSpeed 
FROM CarStream
MATCH_RECOGNIZE(
 MEASURES FIRST(X.Speed) AS StartSpeed, LAST(X.Speed) AS PeekSpeed 
 ALL ROWS PER MATCH
 AFTER MATCH SKIP PAST LAST ROW 
 PATTERN X{10,} Z
 DEFINE
   X AS NEXT(Speed) > Speed,
   Y AS LAST(X.Speed) > Speed
 PARTITION BY Id
 WITHIN 2 MINUTES
)
```
This query allows, through the use of `ALL ROWS PER MATCH` the comparison of the measured speed of the car with the initial and maximum speed during acceleration.

{{< hint type=note >}}
When using `ALL ROWS PER MATCH` the attributes `tstart` and `tend` in the output indicate the time span of the last event  which is part of the found match. The attributes `originaltstart`and `originaltend` however indicate the time span of the individual event.
{{< /hint >}} 

## Measure functions ##
Measure functions can be used to gain more information about matches that do not emerge from the events themselves. As the name implies, these are specified in the measure clause and for the most part only useful with ALL ROWS PER MATCH. The following measure functions are available:
- `CLASSIFIER()`: Creates an attribute in the output containing the symbol that the respective row matched. If using `CLASSIFIER()` with `ONE ROW PER MATCH` an exception is thrown.
- `MATCH_NUMBER()`: Creates an attribute in the output containing the sequential number of the match, starting by 1 for each partition.  Operates the same way in `ALL ROWS PER MATCH` as in `ONE ROW PER MATCH`. 
- `MATCH_SEQUENCE_NUMBER()`: Creates an attribute in the output containing the row number within a match, starting by 1. If using `MATCH_SEQUENCE_NUMBER()` with `ONE ROW PER MATCH` an exception is thrown.

```
SELECT Id, Position, MN
FROM (
     SELECT * 
     FROM CarStream
     MATCH_RECOGNIZE(
     MEASURES MATCH_NUMBER() AS MN, CLASSIFIER() AS CF 
     ALL ROWS PER MATCH
     AFTER MATCH SKIP PAST LAST ROW 
     PATTERN X{10,} Z
     DEFINE
       X AS NEXT(Speed) > Speed,
       Y AS LAST(X.Speed) > Speed
     PARTITION BY Id
     WITHIN 2 MINUTES
      )
) AS Positions
WHERE CF = 'X'
GROUP BY Id,MN
```
Through this query the positions are returned during the acceleration of the car. By adding `MATCH_NUMBER()` the individual acceleration phases can be grouped well for further processing.


## Group Pattern Matching ##
While sequential pattern matching looks for a strict sequence of events within a stream, group patterns look for events within a specified temporal vicinity which fulfill a predicate.  Group patterns come in two flavors. First, a `GPATTERN` identifies groups of objects which fulfill a given predicate (independent of each other). For instance, the pattern:
```
SELECT * 
FROM CarStream
GPATTERN(
  CONDITION SPEED > 50
  MEMBERS 5
  DURATION 10 MINUTES
  GRANULARITY 30 SECONDS
  TIMEOUT 1 MINUTE
  PARTITION BY  ID
  INTERPOLATION LVCF
)
```
logically partitions the stream using the car ids (`PARTITION BY`) and detects groups of five ore more (`MEMBERS`) cars that drive faster than 50 mph (`CONDITION`)  for at least 10 minutes (`DURATION`). The pattern is evaluated every 30 seconds (`GRANULARITY`) on a snapshot of the data. Snapshots are generated from historical data using the specified interpolation method (`INTERPOLATION`). Cars that delivered no updates for more than 1 minute are not considered anymore (`TIMEOUT`).

The second type of group patterns are **Cross patterns (XPATTERN)**. They detect groups of objects via a predicated defined on a pair of objects. For instance the pattern:
```
SELECT * 
FROM CarStream
XPATTERN(
  CONDITION 
    ST_DISTANCE(LEFT.POSITION,RIGHT.POSITION) < 10 AND
    LEFT.SPEED > 90 AND
    RIGHT.SPEED > 90
  MEMBERS 2
  DURATION 3 MINUTES
  GRANULARITY 10 SECONDS
  TIMEOUT 1 MINUTE
  PARTITION BY  ID
  INTERPOLATION LVCF
)
```
detects groups of 2 or more cars that drive fast (more than 90 mph) in close proximity. I.e., are a potential threat. An XPATTERN can be viewed as self-join. Thus, to specify predicates between two objects, their attributes must be qualified with `left` and `right` respectively. The semantics of the remaining parameters are the same as for GPATTERN.

### Interpolation ###
Group pattern operators generate snapshots at the configured granularity rate using one of the following mehtods:

- **LVCF**: Last Value Carried Forward simply uses the value of the last seen event
- **EXTRAPOLATE**: Linearly extrapolates values from the past two events if applicable. I.e., extrapolation is applied to numeric and positional attributes, while the last known value is used for strings and complex geometries (LineString, Polyong).
- **INTERPOLATE**: Interpolates values from two temporally surrounding events. Note that this method potentially causes delays, since "future" events must be available before interpolating a value. 

---
# Expressions #

## Boolean / Logical ##
ChronicleDB supports all common logical expressions/operations:
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `true`   | 0 | Boolean constant `true`. | `true` |
| `false`   | 0 | Boolean constant `false`. | `false` |
| `AND` | 2 | Logical `AND`. | `true AND false` |
| `OR` | 2 | Logical `OR`. | `true OR false` |
| `NOT` | 1 | Negates the result of the input expression. | `NOT false` |

## Predicates ##
The following section lists all predicates available in ChronicleDB.

### Equality ### 
Equality checks may be performed on any data type.
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `=`   | 2 | Returns `true` if the left input expressions equals the right input expression. | `5 = (2+3)` |
| `!=`,`<>` | 2 | Returns `true` if the left input expressions is not equal to the right input expression. | `5 <> 7` |

### Comparisons ###
Comparisons are applicable to all numeric types as well as strings.
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `<`   | 2 | Returns `true` if the left input expressions is strictly less than the right input expression. | `5 < 7` |
| `<=` | 2 | Returns `true` if the left input expressions is less than or equal to the right input expression. | `5 <= 7` |
| `>`   | 2 | Returns `true` if the left input expressions is strictly greater than the right input expression. | `7 = 5` |
| `>=` | 2 | Returns `true` if the left input expressions is greater than or equal to the right input expression. | `7 >= 5` |
| `BETWEEN` | 3 | Returns `true` if the left input expressions is within the value range specified by the 2nd and 3rd input expression. The bounds are inclusive. | `5 BETWEEN 3 AND 7` |

### Strings ###
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `LIKE`   | 2 | Returns `true` if the left input expressions matches the right LIKE-expression. | `'ChronicleDB' LIKE '%icle..'` |


### Spatial Predicates ###
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `ST_CONTAINS` | 2 | Returns `true` if the left input contains the right input expression. | `ST_CONTAINS( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_COVEREDBY` | 2 | Returns `true` if the left input is covered by the right input expression. | `ST_COVEREDBY( POINT( 0 0 ), POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)) )` |
| `ST_COVERS` | 2 | Returns `true` if the left input covers the right input expression. | `ST_COVERS( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_CROSSES` | 2 | Returns `true` if the left input crosses the right input expression. | `ST_CROSSES( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_DISJOINT` | 2 | Returns `true` if the left input is disjoint from the right input expression. | `ST_DISJOINT( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_EQUALSNORM` | 2 | Returns `true` if the normalized left input equals the normalized right input expression. | `ST_EQUALSNORM( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_EQUALSTOPO` | 2 | Returns `true` if the left input is topologically equal to the right input expression. | `ST_EQUALSTOPO( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_ISEMPTY` | 1 | Returns `true` if the given input is an empty geometry. | `ST_ISEMPTY( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)))` |
| `ST_ISRECTANGLE` | 1 | Returns `true` if the given input is a rectangle. | `ST_ISRECTANGLE( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)))` |
| `ST_ISSIMPLE` | 1 | Returns `true` if the given input is a simple geometry. | `ST_ISSIMPLE( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)))` |
| `ST_ISVALID` | 1 | Returns `true` if the given input is a valid geometry. | `ST_ISVALID( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)))` |
| `ST_INTERSECTS` | 2 | Returns `true` if the left input intersects the right input expression. | `ST_INTERSECTS( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_ISWITHINDISTANCE` | 3 | Returns `true` if the distance between the first and the second input is less than or equal to the third input expression | `ST_ISWITHINDISTANCE( POINT( 0 0 ), POINT( 1 1 ), 2.0)` |
| `ST_OVERLAPS` | 2 | Returns `true` if the left input overlaps the right input expression. | `ST_OVERLAPS( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_TOUCHES` | 2 | Returns `true` if the left input touches the right input expression. | `ST_TOUCHES( POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)), POINT( 0 0 ))` |
| `ST_WITHIN` | 2 | Returns `true` if the left input is within the right input expression. | `ST_WITHIN(POINT( 0 0 ), POLYGON((-1 -1) (1 -1) (1 1) (-1 1) (-1 -1)))` |






## Arithmetic Expressions ##
The following section lists all available arithmetic expressions, which can be used in `WHERE` and `SELECT` clauses of a statement.

### Numbers ###

Besides the common infix operations (`+,-,*,/,%`), the following expressions are supported:
| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `Abs` | 1 | Computes the absolute value from the given input expression. | `ABS(5-7) = 2` |
| `Acos` | 1 | Computes the arccosine of the given input expression. | `Acos(0) = Pi/2` |
| `Asin` | 1 | Computes the arcsine of the given input expression. | `Asin(0) = 0` |
| `Atan` | 1 | Computes the arctan of the given input expression. | `Atan(0) = 0` |
| `Atan2` | 2 | Computes the arctan2 of the given input expressions. | `Atan2(0,1) = Pi/2` |
| `Cos` | 1 | Computes the cosine of the given input expression. | `Cos(0) = 1` |
| `Exp` | 1 | Computes e^x, where x is a numeric expression. | `Exp(0) = e^0 = 1` |
| `Log` | 2 | Computes log_y(x), where x and y  are numeric expressions. | `Log(8,2) = 3` |
| `Pow` | 2 | Computes x^y, where x and y  are numeric expressions. | `Pow(2,3) = 8` |
| `Sin` | 1 | Computes the sine of the given input expression. | `Sin(0) = 0` |
| `Sqrt` | 1 | Computes the square root of the given input expression. | `Sqrt(9) = 3` |
| `Tan` | 1 | Computes the tan of the given input expression. | `Tan(0) = 0` |


### Strings ###

| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `Concat` | 2 | Concatenates two strings. | `CONCAT('HELLO', ' WORLD!') = 'HELLO WORLD!'` |

### Spatial -> Spatial ###

| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----
| `ST_BOUNDARY` | 1 | Computes the boundary of the given spatial input expression. |  `ST_BOUNDARY(POINT(1.0 1.0))` |
| `ST_BUFFER` | 3 | Computes a buffer around the spatial input expression. The second argument specifies the size ("thickness") of the buffer, the third argument specifies the cap style (one of `'ROUND'`,`'FLAT'`,`'SQUARE'`) |  `ST_BUFFER(POINT(1.0 1.0), 1.0, 'FLAT' )` |
| `ST_CENTROID` | 1 | Computes the centroid of the given spatial input expression. |  `ST_CENTROID(POINT(1.0 1.0))` |
| `ST_CONVEXHULL` | 1 | Computes the convex hull of the given spatial input expression. |  `ST_CONVEXHULL(POINT(1.0 1.0))` |
| `ST_DIFFERENCE` | 2 | Computes the closure of the point-set of the points contained in first input expression that are not contained in the second input expression |  `ST_DIFFERENCE(POINT(1.0 1.0),POINT(1.0 1.0))` |
| `ST_ENVELOPE` | 1 | Computes the envelope of the given spatial input expression. |  `ST_ENVELOPE(POINT(1.0 1.0))` |
| `ST_GEOMETRYCOLLECTION` | variable | Creates a geometry collection from thegiven input expressions |  `ST_GEOMETRYCOLLECTION(POINT(1.0 1.0), POINT(1.0 2.0))` |
| `ST_GEOMETRYN` | 2 | Retrieves the n-th geometry of a geometry collection. |  `ST_GEOMETRYN( MULTIPOINT((1.0 1.0), (1.0 2.0)), 1)` |
| `ST_INTERIORPOINT` | 1 | Computes an interior point of the given spatial input expression. |  `ST_INTERIORPOINT(POINT(1.0 1.0))` |
| `ST_INTERSECTION` | 2 | Computes intersection of the given spatial input expressions. |  `ST_INTERSECTION(POINT(1.0 1.0), POINT(1.0 1.0))` |
| `ST_NORM` | 1 | Computes a normalized version of the given spatial input expression. |  `ST_NORM(POINT(1.0 1.0))` |
| `ST_SYMDIFFERENCE` | 2 | Computes the closure of the point-set which is the union of the points in left input expression which are not contained in the right input expression with the points in the right argument not contained in the left argument.  |  `ST_SYMDIFFERENCE(POINT(1.0 1.0),POINT(1.0 1.0))` |
| `ST_UNION` | 2 | Computes union of the given spatial input expressions. |  `ST_UNION(POINT(1.0 1.0), POINT(1.0 2.0))` |
| `ST_MAKEPOINT` | 2 | Generates a 2D Point from two numeric input expressions. |  `ST_MAKEPOINT( 1.0, 2.0 )` |
| `ST_LINEPROJECTION` | variable | Generates a 2D linestring from the given  input expressions (must be points). |  `ST_LINEPROJECTION( POINT(1.0 2.0), POINT(1.0 1.0) )` |

### Spatial -> Number ###

| Name | Arg-Count | Description | Example |
| -----  | -----  | -----   | -----  
| `ST_AREA` | 1 | Computes the surface area of the given spatial input expression. |  `ST_AREA(POINT(1.0 1.0))` |
| `ST_BOUNDARYDIMENSION` | 1 | Computes the boundary dimension of the given spatial input expression. |  `ST_BOUNDARYDIMENSION(POINT(1.0 1.0))` |
| `ST_DIMENSION` | 1 | Computes the dimension of the given spatial input expression. |  `ST_DIMENSION(POINT(1.0 1.0))` |
| `ST_AREA` | 1 | Computes the surface area of the given spatial input expression. |  `ST_AREA(POINT(1.0 1.0))` |
| `ST_DISTANCE` | 2 | Computes the raw distance between the given spatial input expressions. |  `ST_DISTANCE(POINT(1.0 1.0),POINT(2.0 1.0))` |
| `ST_LATLONDISTANCE` | 2 | Computes the distance in meters between the given spatial input expressions. This expressions assume lat/lon coordinates. |  `ST_LATLONDISTANCE(POINT(1.0 1.0),POINT(2.0 1.0))` |
| `ST_LENGTH` | 1 | Computes the length of the given spatial input expression. |  `ST_LENGTH(POINT(1.0 1.0))` |
| `ST_NUMGEOMETRIES` | 1 | Computes the number of geometries contained in the given spatial input expression. |  `ST_NUMGEOMETRIES(POINT(1.0 1.0))` |
| `ST_NUMPOINTS` | 1 | Computes the number of points of the given spatial input expression. |  `ST_NUMPOINTS(POINT(1.0 1.0))` |

---
# Aggregates #
ChronicleDB supports the following aggregates
| Name | Description | Example |
| -----  | -----  | ----- 
| `COUNT` | Counts the values of the given attribute column, ignoring `NULL` values. | `COUNT(X)` |
| `SUM` | Computes the sum over the given input column, ignoring `NULL` values. | `SUM(X)` |
| `MIN` | Computes the minimum over the given input column, ignoring `NULL` values. | `MIN(X)` |
| `MAX` | Computes the maximum over the given input column, ignoring `NULL` values. | `MAX(X)` |
| `AVG` | Computes the average over the given input column, ignoring `NULL` values. | `AVG(X)` |
| `STDDEV` | Computes the standard deviation over the given input column, ignoring `NULL` values. | `STDDEV(X)` |
| `FIRST` | Computes the first non-null value of the given column | `FIRST(X)` |
| `LAST` | Computes the last non-null value of the given column | `LAST(X)` |
| `ST_MAKELINE` | Generates a line-string from the coordinates of a given `GEOMETRY` input column. | `ST_MAKELINE(POS)` |
