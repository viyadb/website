Overview
========

ViyaDB is in-memory analytical data store.

## Use cases

 * Customer-facing applications, which serve analytical queries.
 * Incoming events are not organized by any column, thus random updates are very much common.

## Data Model

### Event Structure

Every event consists of two sets:

 * Dimensions
 * Metrics
 
Dimensions are event descriptors. Example of dimensions are: country, user agent, event time, install time, etc.
Metrics are values, mostly numeric. Examples are: events count, temperature, revenue, etc.

### Cardinality Protection

Cardinality protection is built into ViyaDB, which basically means that you can (and should) define the maximum number of distinct elements of any given dimension. This not only allows for filtering out irrelevant values (while still keeping record of their metrics as "Other"), but also makes possible doing optimizations that improve database performance and save memory.
