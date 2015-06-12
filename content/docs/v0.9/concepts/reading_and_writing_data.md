# Reading and Writing Data
There are many ways to write data into InfluxDB including the built-in HTTP API, client libraries, and integrations with external data sources such as Collectd.

## Writing data using the HTTP API
The HTTP API is the primary means of getting data into InfluxDB. To write data simply send a `POST` to the endpoint `/write`. The body of the POST contains the destination database, retention policy, and time-series data you wish to store. An example request sent to InfluxDB running on localhost, which writes a single point, is shown below.

```
# Create your new database, this only needs to be done once.
curl -G http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"

# Write your data to your new database.
curl -i -XPOST 'http://localhost:8086/write?db=mydb&rp=mypolicy' -d 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
```

In the example above the destination database is `mydb`, and the data will be stored in the retention policy named `mypolicy`, which are assumed to exist. The actual data represents the short-term CPU-load on a server server01 in region _us-west_. `database` must be specified in the request body and must already exist. `rp` is optional, but if specified the retention policy must already exist. If `rp` is not specified the default retention policy for the given database is used. Strictly speaking `tags` are optional but most series include tags to differentiate data sources. The `timestamp` is also optional. If you do not specify a timestamp the server's local timestamp will be used.

### Schemaless Design
InfluxDB is schemaless so the series and columns (fields and tags) get created on the fly. You can add columns to existing series without penalty, and integers, floats, strings, booleans, and raw bytes are all supported as types. If you attempt to write data with a different type than previously used (for example writing a string to a tag that previously accepted integers), InfluxDB will reject the data.

### Writing multiple points
As you can see in the example below, you can post multiple points to multiple series at the same time by separating each point with a new line. Batching points in this manner will result in much higher performance.

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb' -d '
cpu_load_short,host=server01,region=us-west value=0.64
cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257
cpu_load_short,direction=in,host=server01,region=us-west value=23422.0 1422568543702900257'
```

### Tags
Each point can have a set of key-value pairs associated with it. Both keys and values must be strings. Tags allow data to be easily and efficient queried, including or excluding data that matches a set of keys with particular values.

### Time format
The following time format is accepted:

_Epoch and Precision_

Timestamps can be supplied as an integer value at the end of the line. The precision is configurable per-request by including a `precision` url parameter. If no precision is specified, the line protocol will default to nanosecond precision. For example to set the time in seconds, use the following request.

```
curl -i -XPOST 'http://localhost:8086/write?db=mydb&precision=s' -d 'cpu_load_short,host=server01,region=us-west value=0.64 1434059627'
```

`n`, `u`, `ms`, `s`, `m`, and `h` are all supported and represent nanoseconds, microseconds, milliseconds, seconds, minutes, and hours, respectively.

### Response
Once a configurable number of servers have acknowledged the write, the node that initially received the write responds with `HTTP 200 OK`.

#### Errors
If an error was encountered while processing the data, InfluxDB will respond with either a `HTTP 400 Bad Request` or, in certain cases, with `HTTP 200 OK`. The former is returned if the request could not be understood. In the latter, InfluxDB could understand the request, but processing cannot be completed. In this case a JSON response is included in the body of the response with additional error information.

For example, issuing a bad query such as:

```
curl -G http://localhost:8086/query --data-urlencode "db=foo" --data-urlencode "q=show"
```

will result in `HTTP 400 Bad Request` with the the following JSON in the body of the response:

```json
{"error":"error parsing query: found EOF, expected SERIES, CONTINUOUS, MEASUREMENTS, TAG, FIELD, RETENTION at line 1, char 6"}
```

## Querying data using the HTTP API
The HTTP API is also the primary means for querying data contained within InfluxDB. To perform a query send a `GET` to the endpoint `/query`, set the URL parameter `db` as the target database, and set the URL parameter `q` as your query. An example query, sent to a locally-running InfluxDB server, is shown below.

```
curl -G 'http://localhost:8086/query' --data-urlencode "db=mydb" --data-urlencode "q=SELECT value FROM cpu_load_short WHERE region='us-west'"
```

Which returns:

```json
{
    "results": [
        {
            "series": [
                {
                    "measurement": "cpu_load_short",
                    "tags": {
                        "host": "server01",
                        "region": "us-west"
                    },
                    "columns": [
                        "time",
                        "value"
                    ],
                    "values": [
                        [
                            "2015-01-29T21:51:28.968422294Z",
                            0.64
                        ]
                    ]
                }
            ]
        }
    ]
}
```

In general the response body will be of the following form:

```json
{
    "results": [
        {
            "series": [{}],
            "error": "...."
        }
    ],
    "error": "...."
}
```

There are two top-level keys. `results` is an array of objects, one for each query, each containing the keys for a `series`. Each _row_ contains a data point returned by the query. If there was an error processing the query, the `error` key will be present, and will contain information explaining why the query failed. An example of this type of failure would be attempt to query a series that does not exist.

The second top-level key is also named `error`, and is set if the API call failed before InfluxDB could perform any *query* operations. A example of this kind of failure would be invalid authentication credentials.

### Timestamp Format
The format of the returned timestamps complies with RFC3339, and has nanosecond precision.

### Multiple queries

Multiple queries can be sent to InfluxDB in a single API call. Simply delimit each query using a semicolon, as shown in the example below.

```
curl -XGET 'http://localhost:8086/query' --data-urlencode "db=mydb" --data-urlencode "q=SELECT * FROM cpu_load_short WHERE region=us-west;SELECT * FROM cpu_load_long"
```

## Authentication
Authentication is disabled by default. If authentication is enabled, user credentials must be supplied with every query. These can be supplied via the URL parameters `u` and `p`. For example, if the  user is "bob" and the password is "mypass", then endpoint URL should take the form `/query?u=bob&p=mypass`.

The credentials may also be passed using _Basic Authentication_. If both types of authentication are present in a request, the URL parameters take precedence.

## Pretty Printing
When working directly with the API it is often convenient to have pretty-printed JSON output. To enable pretty-printed output, append `pretty=true` to the URL. For example:

```
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT * FROM cpu_load_short"
```

Pretty-printed output is useful for debugging or when querying directly using tools like `curl`, etc., It is not recommended for production use as it consumes unnecessary network bandwidth.
