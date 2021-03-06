# Teraslice - Slice and dice your Elasticsearch data

[![Build Status](https://travis-ci.org/terascope/teraslice.svg?branch=master)](https://travis-ci.org/terascope/teraslice)

Teraslice is an open source, distributed computing platform for processing JSON data stored in Elasticsearch. It can be used for many tasks but is particularly adept at migrating and transforming data within and between Elasticsearch clusters and other data stores. It was born and bred in an environment that regularly sees billions of pieces of data per day and is capable of processing millions of records per second.

Here are a few tasks it can help you with:

  * Reindexing data at high volumes
  * Moving data between clusters
  * Moving data out of Elasticsearch into other systems
  * Automatic syncing of data from Elasticsearch to other systems
  * Exporting data to files
  * Importing data from a previous export
  * Periodic processing of data stored in an index

In Teraslice you define jobs that specify a pipeline of work to be applied to a slice of data. That work will execute concurrently across many workers to achieve very high re-processing throughput.

[Watch the overview and getting started video](https://www.youtube.com/watch?v=TG7flPTZeeg)

[![Overview and Getting Started](https://raw.github.com/terascope/teraslice/master/docs/images/reindexing-id-71B.png)](https://www.youtube.com/watch?v=TG7flPTZeeg)

The only requirement that Teraslice makes is that the data is sliceable using date ranges. So as long as your index has a date field that varies across records then you can use it to slice things up and concurrently reprocess the data in the index.

Jobs are specified as simple JSON documents. Here's a simple reindexing example.

```
{
    "name": "Reindex",
    "lifecycle": "once",
    "workers": 1,
    "operations": [
        {
          "_op": "elasticsearch_reader",
          "index": "example-logs",
          "type": "logs",
          "size": 10000,
          "date_field_name": "created",
          "full_response": true
        },
        {
            "_op": "elasticsearch_index_selector",
            "type": "change",
            "index": "example-logs-new",
            "id_field": "_key"
        },
        {
          "_op": "elasticsearch_bulk",
          "size": 10000
        }
    ]
}
```

In this instance all the operations that are performed are provided by Teraslice. The elasticsearch_reader takes a date range and slices up the index so that it can be reprocessed. The result of that is fed to elasticsearch_index_selector that takes the incoming data and bundles it into a bulk request that is then sent to the cluster using elasticsearch_bulk_insert.

Operations are nothing more than Javascript modules and writing your own is easy so custom code can be inserted in the operations flow as needed.

# Status

Teraslice is currently in alpha status. Single node deployment and clustering support are functional and being refined. APIs are usable but will still be evolving as we work toward a production release. See the list of open issues for other limitations.

# Installation

Teraslice is written in Node.js and has been tested on Linux and Mac OS X.

### Dependencies ###
* Node.js 4 or above
* At least one Elasticsearch 5 or above cluster

### Installing with npm ###

```
npm install terascope/teraslice
```

# Configuration Single Node / Cluster Master

Teraslice requires a configuration file in order to run. The configuration file defines your service connections and system level configurations.

This configuration example defines a single connection to Elasticsearch on localhost with 8 workers available to Teraslice. The *teraslice.ops_directory* setting tells Teraslice where it can find custom operation implementations.

The cluster configuration defines this node as a master node. The node will still have workers
available and this configuration is sufficient to do useful work if you don't have multiple
nodes available. The workers will connect to the master on localhost and do work just as if they were in a real cluster.


```
teraslice:
    ops_directory: '/path/to/ops/'
    workers: 8
    cluster:
        master: true
        master_hostname: "127.0.0.1"
        name: "teracluster"

terafoundation:
    environment: 'development'
    log_path: '/path/to/logs'

    connectors:
        elasticsearch:
            default:
                host:
                    - "localhost:9200"
```

# Configuration Cluster Worker Node

Configuration for a worker node is very similar. You just set 'master' to false and provide the IP address where the master node can be located.

```
teraslice:
    ops_directory: '/path/to/ops/'
    workers: 8
    cluster:
        master: false
        master_hostname: "YOUR_MASTER_IP"
        name: "teracluster"

terafoundation:
    environment: 'development'
    log_path: '/path/to/logs'

    connectors:
        elasticsearch:
            default:
                host:
                    - "YOUR_MASTER_IP":9200
```

# Running

Once you have Teraslice installed you need a job specification and a configuration file to do something useful with it. See above for simple examples of each.

Starting the Teraslice service on the master node is simple. Just provide it a path to the configuration file.

```
node service.js -c master-config.yaml
```

Starting a worker on a remote node is basically the same.

```
node service.js -c worker-config.yaml
```

The master publishes a REST style API on port 5678.

To submit a job you just post to the /jobs endpoint.

Assuming your job is in a file called 'job.json' it's as simple as

```
curl -XPOST YOUR_MASTER_IP:5678/jobs -d@job.json
```

This will return the job_id (for access to the original job posted) and the job execution context id (the running instance of a job) which can then be used to manage the job. This will also start the job.
```
{
    "job_id": "5a50580c-4a50-48d9-80f8-ac70a00f3dbd",
}
```
# Job Control

Please check the api docs at the bottom for a comprehensive in-depth list of all api's. What is listed here is just a small brief of only a few api's

### Job status

This will retrieve the job configuration including '\_status' which indicates the execution status of the job.

```
curl YOU_MASTER_IP:5678/jobs/{job_id}/ex
```

### Stopping a job

Stopping a job stops all execution and frees the workers being consumed
by the job on the cluster.

```
curl -XPOST YOU_MASTER_IP:5678/jobs/{job_id}/_stop
```

### Starting a job

Posting a new job will automatically start the job. If the job already exists then using the endpoint below will start a new one.

```
curl -XPOST YOU_MASTER_IP:5678/jobs/{JOB_ID}/_start
```

Starting a job with recover will attempt to replay any failed slices from previous runs and will then pickup where it left off. If there are no failed
slices the job will simply resume from where it was stopped.

```
curl -XPOST YOU_MASTER_IP:5678/jobs/{job_id}/_recover
```

### Pausing a job

Pausing a job will stop execution of the job on the cluster but will not
release the workers being used by the job. It simply pauses the slicer and
stops allocating work to the workers. Workers will complete the work they're doing then just sit idle until the job is resumed.

```
curl -XPOST YOU_MASTER_IP:5678/jobs/{job_id}/_pause
```

### Resuming a job

Resuming a job restarts the slicer and the allocation of slices to workers.

```
curl -XPOST YOU_MASTER_IP:5678/jobs/{job_id}/_resume
```

### Viewing Slicer statistics for a job

This provides information related to the execution of the slicer and can be useful
in monitoring and optimizing the execution of the job.

```
curl YOU_MASTER_IP:5678/jobs/{job_id}/slicer
```

### Viewing cluster state

This will show you all the connected workers and the tasks that are currently assigned to them.

```
curl YOU_MASTER_IP:5678/cluster/state
```

# Additional Documentation

## API

 * [API endpoints reference](./docs/api.md)

## Operations

 * [Job configuration and operations reference](./docs/ops-reference.md)
 * [Custom operations](./docs/custom_operations.md)

## Configuration

 * [Teraslice configuration reference](./docs/configuration.md)

## Services
 * [additional services](./docs/services.md)
