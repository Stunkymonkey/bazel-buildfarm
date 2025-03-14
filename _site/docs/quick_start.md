---
layout: default
title: Quick Start
nav_order: 3
---

# Quick Start

Here we describe how to use bazel remote caching or remote execution with buildfarm. We will create a single client workspace that can be used for both.

## Setup

You can run this quick start on a single computer running nearly any flavor of linux. This computer is the localhost for the rest of the description.

### Backplane

Buildfarm requires a backplane to store information that is shared between cluster members. A [redis](https://redis.io) server can be used to meet this requirement.

Download/Install a redis-server instance and run it on your localhost. The default redis port of 6379 will be used by the default buildfarm configs.

## Workspace

Let's start with a bazel workspace with a single file to compile into an executable:

Create a new directory for our workspace and add the following files:

`main.cc`:
```
#include <iostream>

int main( int argc, char *argv[] )
{
  std::cout << "Hello, World!" << std::endl;
}
```

`BUILD`:
```
cc_binary(
    name = "main",
    srcs = ["main.cc"],
)
```

And an empty WORKSPACE file.

As a test, verify that `bazel run :main` builds your main program and runs it, and prints `Hello, World!`. This will ensure that you have properly installed bazel and a C++ compiler, and have a working target before moving on to remote execution.

Download and extract the buildfarm repository. Each command sequence below will have the intended working directory indicated, between the client (workspace running bazel), and buildfarm.

This tutorial assumes that you have a bazel binary in your path and you are in the root of your buildfarm clone/release, and has been tested to work with bash on linux.

## Remote Caching

A Buildfarm server with an instance can be used strictly as an ActionCache and ContentAddressableStorage to improve build performance. This is an example of running a bazel client that will retrieve results if available, and store them if the cache is missed and the execution needs to run locally.

Download the buildfarm repository and change into its directory, then:

run `bazelisk run src/main/java/build/buildfarm:buildfarm-server $PWD/examples/config.minimal.yml`

This will wait while the server runs, indicating that it is ready for requests.

From another prompt (i.e. a separate terminal) in your newly created workspace directory from above:

run `bazel clean`
run `bazel run --remote_cache=grpc://localhost:8980 :main`

Why do we clean here? Since we're verifying re-execution and caching, this ensures that we will execute any actions in the `run` step and interact with the remote cache. We should be attempting to retrieve cached results, and then when we miss - since we just started this memory resident server - bazel will upload the results of the execution for later use. There will be no change in the output of this bazel run if everything worked, since bazel does not provide output each time it uploads results.

To prove that we have placed something in the action cache, we need to do the following:

run `bazel clean`
run `bazel run --remote_cache=localhost:8980 :main`

This should now print statistics on the `processes` line that indicate that you've retrieved results from the cache for your actions:

```
INFO: 2 processes: 2 remote cache hit.
```

## Remote Execution (and caching)

Now we will use buildfarm for remote execution with a minimal configuration - a single memory instance, with a worker on the localhost that can execute a single process at a time - via a bazel invocation on our workspace.

First, we should restart the buildfarm server to ensure that we get remote execution (this can also be forced from the client by using `--noremote_accept_cached`). From the buildfarm server prompt and directory:

interrupt a running `buildfarm-server`
run `bazelisk run src/main/java/build/buildfarm:buildfarm-server $PWD/examples/config.minimal.yml`

From another prompt in the buildfarm repository directory:

run `bazelisk run src/main/java/build/buildfarm:buildfarm-shard-worker $PWD/examples/config.minimal.yml`

From another prompt, in your client workspace:

run `bazel run --remote_executor=grpc://localhost:8980 :main`

Your build should now print out the following on its `processes` summary line:

```
INFO: 2 processes: 2 remote.
```

That `2 remote` indicates that your compile and link ran remotely. Congratulations, you just build something through remote execution!

## Container Quick Start

To bring up a minimal buildfarm cluster, you can run:
```
./examples/bf-run start
```
This will start all of the necessary containers at the latest version.
Once the containers are up, you can build with `bazel run --remote_executor=grpc://localhost:8980 :main`.

To stop the containers, run:
```
./examples/bf-run stop
```

## Buildfarm Manager

You can now easily launch a new Buildfarm cluster locally or in AWS using an open sourced [Buildfarm Manager](https://github.com/80degreeswest/bfmgr).

```
wget https://github.com/80degreeswest/bfmgr/releases/download/1.0.7/bfmgr-1.0.7.jar
java -jar bfmgr-1.0.7.jar
Navigate to http://localhost
```
