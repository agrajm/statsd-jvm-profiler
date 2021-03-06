# statsd-jvm-profiler - a foreword by Ihor Bobak

Hi, everyone. My name is Ihor Bobak, I created a mod of a famous statsd-jvm-profiler which you can find in this repository. Below are first my comments, then you may read original text of the authors of StatsD JVM Profiler.

Here http://ihorbobak.com/index.php/2015/08/05/cluster-profiling/ you may find my article that describes how to use it in details.

## List of My Changes

1. Profiler adds the jvmName tag to each stacktrace;
2. Optimized performance;
3. Added catching mechanizm for OutOfMemoryException which should not allow the profiler to fail;
4. added statistics which shows how many lines and characters we are passing to the backend (to track if OOM appears);
5. Fixed the influxdb_dump.py: now it extracts data into a set of distinct files - one file for each JVM. 

# ORIGINAL TEXT - statsd-jvm-profiler [![Build Status](https://travis-ci.org/etsy/statsd-jvm-profiler.svg)](https://travis-ci.org/etsy/statsd-jvm-profiler)

statsd-jvm-profiler is a JVM agent profiler that sends profiling data to StatsD.  Inspired by [riemann-jvm-profiler](https://github.com/riemann/riemann-jvm-profiler), it was primarily built for profiling Hadoop jobs, but can be used with any JVM process.

Read [the blog post](https://codeascraft.com/2015/01/14/introducing-statsd-jvm-profiler-a-jvm-profiler-for-hadoop/) that introduced statsd-jvm-profiler on [Code as Craft](https://codeascraft.com/), Etsy's engineering blog.

Also check out [the blog post](https://codeascraft.com/2015/05/12/four-months-of-statsd-jvm-profiler-a-retrospective/) reflecting on the experience of open-sourcing the project.

## Mailing List
There is a mailing list for this project at https://groups.google.com/forum/#!forum/statsd-jvm-profiler.  If you have questions or suggestions for the project send them here!

## Installation

You will need the statsd-jvm-profiler JAR on the machine where the JVM will be running.  If you are profiling Hadoop jobs, that means the JAR will need to be on all of the datanodes.

The JAR can be built with `mvn package`.  You will need a relatively recent Maven (at least Maven 3).

## Usage

The profiler is enabled using the JVM's `-javaagent` argument.  You are required to specify at least the StatsD host and port number to use.  You can also specify the prefix for metrics and a whitelist of packages to be included in the CPU profiling.  Arguments can be specified like so:
```
-javaagent:/usr/etsy/statsd-jvm-profiler/statsd-jvm-profiler.jar=server=hostname,port=num
```

An example of setting up Cascading/Scalding jobs to use the profiler can be found in the `example` directory.

### Global Options

Name             | Meaning
---------------- | -------
server           | The hostname to which the reporter should send data (required)
port             | The port number for the server to which the reporter should send data (required)
prefix           | The prefix for metrics (optional, defaults to statsd-jvm-profiler)
packageWhitelist | Colon-delimited whitelist for packages to include (optional, defaults to include everything)
packageBlacklist | Colon-delimited whitelist for packages to exclude (optional, defaults to exclude nothing)
profilers        | Colon-delimited list of profiler class names (optional, defaults to CPUProfiler and MemoryProfiler)
reporter         | Class name of the reporter to use (optional, defaults to StatsDReporter)
httpPort         | The port on which to bind the embedded HTTP server (optional, defaults to 5005)

### Embedded HTTP Server
statsd-jvm-profiler embeds an HTTP server to support simple interactions with the profiler while it is in operation.  You can configure the port on which this server runs with the `httpPort` option.
 
Endpoint            | Usage
---------------     | -----
/profilers          | List the currently enabled profilers
/disable/:profiler  | Disable the profiler specified by `:profiler`. The name must match what is returned by `/profilers`.

### Reporters
statsd-jvm-profiler supports multiple backends.  StatsD is the default, but InfluxDB is also supported.  You can select the backend to use by passing the `reporter` argument to the profiler; `StatsDReporter` and `InfluxDBReporter` are the supported values.

Some reporters may require additional arguments.

#### StatsDReporter
This reporter does not have any additional arguments.

#### InfluxDBReporter

Name        | Meaning
----------- | -------
username    | The username with which to connect to InfluxDB (required)
password    | The password with which to connect to InfluxDB (required)
database    | The database to which to write metrics (required)
tagMapping  | A mapping of tag names from the metric prefix (optional, defaults to no mapping)

##### Tag Mapping
InfluxDB 0.9 supports tagging measurements and querying based on those tags.  statsd-jvm-profilers uses these tags to support richer querying of the produced data.  For compatibility with other metric backends, the tags are extracted from the metric prefix.

If the `tagMapping` argument is not defined, only the `prefix` tag will be added, with the value of the entire prefix.

`tagMapping` should be a period-delimited set of tag names.  It must have the same number of components as `prefix`, or else an exception would be thrown.  Each component of `tagMapping` is the name of the tag.  The component in the corresponding position of `prefix` will be the value.

If you do not want to include a component of `prefix` as a tag, use the special name `SKIP` in `tagMapping` for that position.

## Metrics

`statsd-jvm-profiler` will profile the following:

1. Heap and non-heap memory usage
2. Number of GC pauses and GC time
3. Time spent in each function

Assuming you use the default prefix of `statsd-jvm-profiler`, the memory usage metrics will be under `statsd-jvm-profiler.heap` and `statsd-jvm-profiler.nonheap`, the GC metrics will be under `statsd-jvm-profiler.gc`, and the CPU time metrics will be under `statsd-jvm-profiler.cpu.trace`.

Memory and GC metrics are reported once every 10 seconds.  The CPU time is sampled every millisecond, but only reported every 10 seconds.  The CPU time metrics represent the total time spent in that function.

Profiling a long-running process or a lot of processes simultaneously will produce a lot of data, so be careful with the capacity of your StatsD instance.  The `packageWhitelist` and `packageBlacklist` arguments can be used to limit the number of functions that are reported.  Any function whose stack trace contains a function in one of the whitelisted packages will be included.

You can disable either the memory or CPU metrics using the `profilers` argument:

1. Memory metrics only: `profilers=MemoryProfiler`
2. CPU metrics only: `profilers=CPUProfiler`

## Visualization

The `visualization` directory contains some utilities for visualizing the output of the profiler.

## Contributing
Contributions are highly encouraged!  Check out [the contribution guidlines](https://github.com/etsy/statsd-jvm-profiler/blob/master/CONTRIBUTING.md).

Any ideas you have are welcome,  but check out [some ideas](https://github.com/etsy/statsd-jvm-profiler/wiki/Contribution-Ideas) for contributions.
