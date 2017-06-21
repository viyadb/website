Usage
======

The following section describes how to configure and run a ViyaDB instance.

## Configuring DB Instance

Store descriptor preresents a configuration file of a single ViyaDB instance. The format is the following:

```json
{
  "query_threads": 1,
  "cpu": [ ... ],
  "port": 52341,
  "tables": [ ... ]
}
```

Parameters:

 * query\_threads - Number of threads that will serve queries, defaults to 1.
 * cpu - List of zero-based CPU indices that this process will use, defaults to all available CPUs.
 * port - REST service port number, defaults to 52341.
 * tables - List of [table descriptors](#usage-creating-tables).

## Creating Tables

Table descriptors can be either a part of [store descriptor], or they can be created on-demand using REST API.

```json
{
  "name": "<table name>",
  "dimensions": [ ... ],
  "metrics": [ ... ]
}
```

Parameters:

 * name - Table name
 * dimensions - List of [dimension descriptors](#usage-defining-dimensions)
 * metrics - List of [metric descriptors](#usage-defining-metrics)

#### Defining Dimensions

#### Defininig Metrics

## Data Ingestion

### Loading from TSV files

Save load descriptor into `load.json` file:

```json
{
  "table": "target_table",
  "format": "tsv",
  "type": "file",
  "file": "/path/to/data.tsv"
}
```

Post this load descriptor to running ViyaDB instance:

    ~$ curl --data-binary @load.json http://<viyadb-host>:<viyadb-port>/load

Important notes:

 * The `data.tsv` file must be accessible to the ViyaDB instance you're loading the data into.
 * Order of columns in .tsv file must be as follows: first dimensions, then metrics as they appear in the table descriptor.

## Querying

Query is described using 

