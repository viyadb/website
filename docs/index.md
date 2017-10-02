Overview
========

ViyaDB is in-memory columnar analytical data store, featuring:

- Fast ad-hoc analytical queries
- Random access update pattern
- Built-in cardinality protection
- Real-time query compilation to machine code
- Dynamic period based rollup
- REST API interface with intuitive JSON-based language

Introduction slides are available [here](/slides/intro.html).

## Use cases

 * Customer-facing applications that serve ad-hoc analytical queries (aggregations).
 * Incoming events are not organized by any column, thus random updates are very common.

## Data Model

This section explain some assumptions we made in ViyaDB that make different optimizations possible.

### Event Structure

Every event consists of two sets:

 * Dimensions
 * Metrics
 
Dimensions are event descriptors. Example of dimensions are: country, user agent, event time, install time, etc. Metrics are values, mostly numeric. Examples are: events count, temperature, revenue, etc.

Incoming events are aggregated during data ingestion based on defined metric functions, which means that the database stores aggregated data.

For more information on supported data types, please refer to Data Ingestion section on [usage](/usage) page.

### Column Names

Dimension names are global per database instance. That means if dimension is named 'country' in a two different tables it must have the same meaning (or, more specifically, must share the same set of values).

### Cardinality Protection

Cardinality protection is built into ViyaDB, which basically means that you can (and should) define the maximum number of distinct elements of any given dimension. This not only allows for filtering out irrelevant values (while still keeping record of their metrics as "Other"), but also makes possible doing optimizations that improve database performance and save memory.

Dimension cardinality can be applied either on a dimension independently or based on a set of other dimensions. For instance, you can disallow more than 100 different event names coming from a single mobile application per single day.

## Query Compilation

The first time a query is introduced to ViyaDB, a highly optimized C++ source code is generated, and the compiled version of this code is used for physical data access. That allows to minimize branch mis-predictions, increase the level of CPU cache locality, etc. That means that the first execution of a new query type will take some extra time needed for compiling it, all the subsequent queries of the same type will use already compiled version.

## Supported Systems

The only supported system is Linux x86\_64 with the latest GCC or CLang compiler installed. This strict requirement allows focusing on extracting the most of performance from the underlying system.

