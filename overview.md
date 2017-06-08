ViyaDB
=======

In-memory analytical data store.

## Use case

ViyaDB aims to provide fast analytical queries on non-timeseries or semi-organized data.

## Assumptions

This data store uses several assumptions about the data and the way the data is stored to make optimizations possible.

 * Events are represented by dimensions and metrics. Dimensions are string values (country, user agent, etc.), while metrics are either of long or double types.
 * Data ingestion operation is an UPSERT where metrics are aggregated based on a given set of dimensions.
 * New rows values are coming in the same order as table definition: first dimensions, then metrics.
 * Dimension values cardinality can and should be limited. For example, limiting cardinality of country names to 200 will allow using single byte encoding.
 * Dimension values cardinality can also be limited per subset of other dimensions, which is useful from memory space saving perspective.

## Building from source

If your development machine is not Linux, unfortunately, refer to [this](devenv) document for instructions.

### Prerequisites

You must have the following prerequisites installed:

 * CMake >= 3.2
 * Boost >= 1.64.0
 * g++ >= 7.1

Additional third party dependencies are included into the project as Git submodules.

### Building

To fetch third party dependencies for the first time, run:

    git submodule update --init --recursive

To update third party dependencies when needed, run:

    git submodule update --recursive --remote

To build the project, run:

    mkdir build/
    cd build/
    cmake ..
    make -j8

### Testing

Unit tests are built as part of the main build process. To invoke all unit tests, run:

    GLOG_logtostderr=1 ./test/unit_tests

