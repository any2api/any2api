# any2api: the better way to create awesome APIs 🚀

**any2api** aims to establish an open ecosystem of composable API bricks, including reusable wrappers and adapters.
Instead of using basic API development frameworks to build specific kinds of APIs from scratch (e.g. REST or messaging APIs), the API bricks simplify the process of creating diverse APIs.
For example, a Python wrapper is used to bring a simple *.py* script into the ecosystem.
Then, a specific adapter such as a REST adapter is utilized to expose the script's functionality through an API.
Intermediate adapters can optionally add more functionality to the provided API such as authentication.
In addition, another adapter can be used to make the same functionality available in a different way, fostering API diversity:

*API diversity is key because there is no 'one-fits-all' kind of API. In certain cases, REST is a good choice, but sometimes messaging, RPC or streaming APIs work much better.*

API bricks are generic, reusable and configurable.
They implement best practices how certain kinds of APIs are built, for example, how to implement long-running tasks with REST or how to emulate a synchronous invocation with a messaging API.
This helps to avoid common and reoccurring issues that happen a lot when developing APIs from scratch.



## This repo: any2api core & CLI

This repository contains the core framework and CLI to create APIs based on the any2api ecosystem.
API bricks such as adapters and wrappers are delivered through separate (public or private) repositories.



## Interfaces: containers & gRPC

Since the any2api ecosystem is about composing API bricks to create awesome APIs, each building block must implement and respect specific interfaces.
To foster tech diversity and allow the implementation of composable API bricks using different technologies, the technical foundation of the ecosystem are containers and gRPC:

* API bricks are packaged as Docker/OCI containers. Therefore, established container deployment/orchestration tools and platforms can be used such as Kubernetes and Docker Compose.

* API bricks communicate via Protocol Buffers (proto3) and gRPC, a high performance HTTP/2-based RPC framework.
  * http://developers.google.com/protocol-buffers/docs/reference/proto3-spec
  * http://developers.google.com/protocol-buffers/docs/proto3
  * http://www.grpc.io/docs/guides/concepts.html

There are different kinds of API bricks, mainly **gRPC apps**, **wrappers** and **adapters**.
In the following, the interface of each kind of API brick is described.



### gRPC app

A gRPC app exposes specific functionality (application/business logic) through a gRPC endpoint, which is specified by a `.proto` file.
To ease the composition of gRPC apps and API adapters, it is highly recommended to run the app inside a Docker/OCI container.
The interface of a gRPC app container:

* A gRPC app *must* provide the following file to specify its interface: `/api/main.proto`

* The `/api` directory *must* be provided as a shared container volume by the gRPC app container. It is used by API adapters.
  * In case of Docker, the `Dockerfile` of the gRPC app *must* include the statement `VOLUME /api`.

* The `/api` directory *can* optionally contain additional files such as metadata, which may be read and considered by certain API adapters.
  * For example, a REST API adapter may require additional information on how to properly map and group RPC operations to resources and their associated CRUD operations.

* Each gRPC app *must* define and expose the port that serves the gRPC endpoint described by the `/api/main.proto` file.
  * In case of Docker, the `Dockerfile` of the gRPC API container *must* include the statements `ENV GRPC_PORT …` and `EXPOSE $GRPC_PORT`.

gRPC apps can be developed from scratch using any programming language that is supported by the gRPC framework.
Alternatively, wrappers can be used as described in the following to provide gRPC endpoints for existing code/executables/APIs/… in a dynamic and automated fashion.



### Wrapper

A wrapper is a special kind of gRPC app container, so its interface follows the previously described constraints.
However, it does not implement specific application/business logic, but provides generic logic to wrap a specific kind of executable/API/… and exposes its functionality through a gRPC endpoint.
For example, a Python wrapper can transparently expose a gRPC endpoint for any given Python script.

As a result, wrappers further reduce API development efforts because you just implement the application logic using a language and technology of your choice.
Then, you utilize an appropriate wrapper to make a gRPC endpoint out of it.
Now you can instantly use and benefit from the entire any2api ecosystem without having to develop an API from scratch.

Technically, a wrapper is a container template to be refined, so that it bundles the wrapping logic with an existing executable/API/…; the resulting bundle is delivered as a gRPC app container.
In case of Docker, the `Dockerfile` of the wrapper includes the statements `ONBUILD ADD . /wrap` and `ONBUILD RUN cp -a /wrap/api /api 2>/dev/null || :` to add all relevant files from the build context directory to the `/wrap` and `/api` directories of the resulting gRPC app container.
Providing an `/api` directory (intended for adapters) as part of the build context is optional because it could be dynamically generated by the wrapper.
Additional `ONBUILD RUN …` statements are typically included in the template container, for example, to generate the `/api/main.proto` file and to wire the wrapping logic with the given executable.
All `ONBUILD` statements are executed when building a gRPC app container based on the wrapper container template.

The build context directory that is added as `/wrap` directory to the resulting gRPC app container includes the code/executables/… to be wrapped and exposed through the gRPC endpoint.
Highly sophisticated wrappers would automatically analyze the provided code to identify and extract metadata such as runtime dependencies and input parameters.
Alternatively, such metadata can be declaratively specified by a `wrapspec` file located in the build context directory.
The schema for a `wrapspec.json` file to wrap an executable such as a Python script or Java JAR executable is outlined in the following:

``` plaintext
{
  "title":   (string), # title of resulting gRPC app
  "wrapper": (string), # name of a specific wrapper to use

  "operations": {
    (operationName): {
      "description": (string),
      "readme_file": (string),     # path to README file
      "longrunning": (boolean),

      "parameters_schema": {
        (parameterName): {
          "description": (string),
          "type":        (string), # JSON or proto3 type
          "proto":       (string), # custom proto3 type def
          "streamable":  (boolean),
          "updatable":   (boolean),
          "default":     (string), # default value
          "map_to":      "env:…" | "stdin" | "file:…" | (any)
        }
      },

      "results_schema": {
        (resultName): {
          "description": (string),
          "type":        (string), # JSON or proto3 type
          "proto":       (string), # custom proto3 type def
          "streamable":  (boolean),
          "map_from":    "stdout" | "file:…" | (any),
        }
      },

      "commands" {
        "prepare": (string),       # shell command
        "invoke":  (string)        # shell command
      }
    }
  }
}
```

The structure and content of `wrapspec` files vary depending on the specific wrapper.
For example, a wrapper that wraps an existing REST API and exposes its functionality through a gRPC endpoint may refer to a Swagger definition, so the corresponding `wrapspec` file with its parameter and result mappings may look different.



### Adapter

As outlined previously, a gRPC app exposes specific functionality (application/business logic) through a gRPC endpoint.
Such an app can either be developed from scratch or existing code/executables/… can be transformed into gRPC apps using wrappers.
Either way, gRPC endpoints are one kind of API that can be used by other applications.
However, in the sense of API diversity, other kinds of APIs such as REST and messaging are more appropriate for certain usage scenarios.
Therefore, API adapters (REST, RabbitMQ, JSON-RPC, etc.) are connected with gRPC apps to serve diverse APIs.

An API adapter does not always translate to different kinds of APIs such as gRPC ↔ REST, gRPC ↔ RabbitMQ, etc.
There are also intermediate adapters that take a gRPC endpoint and expose another gRPC endpoint.
gRPC-to-gRPC adapters can implement diverse middleware functionality such as reshaping gRPC endpoints (e.g. consolidating and filtering operations), authentication, request rate limiting, monitoring, analytics, content transformation, etc.

An intermediate adapter can be used in conjunction with any gRPC app and any other API adapter, including other intermediate adapters.
As a result, multiple adapters can be chained, for example: gRPC app ↔ monitoring adapter ↔ rate limit adapter ↔ authentication adapter ↔ REST API adapter.

To ease the composition of gRPC apps and API adapters, it is highly recommended to run each adapter inside a Docker/OCI container.
The interface of an adapter container:

* An API adapter expects the `/api/main.proto` to specify the underlying gRPC endpoint.
  * The environment variable `MAIN_PROTO_PATH` *can* optionally be provided to the API adapter to load another proto3 file instead of the default `/api/main.proto`. Each API adapter *must* process and respect this variable if it is set.

* The `/api` directory is a shared Docker volume. It is typically provided by the underlying gRPC app container (or intermediate adapter container) and used by the API adapter container.

* The `/api/main.proto` file (or its substitute defined by the `MAIN_PROTO_PATH` variable) is the *only* required file. Therefore each API adapter *must* work properly based on this file without any additional metadata.

* The `/api` directory *can* optionally contain additional files such as metadata, which may be read and considered by certain API adapters.
  * For example, a REST API adapter may require additional information on how to properly map and group RPC operations to resources and their associated CRUD operations.

* Each API adapter *must* understand all proto3 and gRPC features.

* The environment variables `GRPC_HOST` (IP address or hostname) and `GRPC_PORT` (port number) *must* be provided to the API adapter, so the adapter can connect to the underlying gRPC endpoint.

* An adapter *must* delay its start until the underlying gRPC endpoint (provided by a gRPC app or another adapter) is available. This behavior can be implemented using [dockerize](https://github.com/jwilder/dockerize) by the `Dockerfile` statement `CMD dockerize -wait tcp://$GRPC_HOST:$GRPC_PORT …`. Only after the availability of the underlying gRPC endpoint, the adapter can be sure that the required proto3 file exists.

* Intermediate gRPC-to-gRPC adapters *must* also implement the interface of a gRPC app container to enable other adapters to run on top.

* Since intermediate gRPC-to-gRPC adapters potentially modify the originally provided gRPC endpoint's interface, the environment variable `UPDATED_PROTO_PATH` *can* be provided to an intermediate adapter. The updated proto3 file is written to this path, which should point to a file in the `/api` directory. If the variable is not provided, the adapter overrides the `/api/main.proto` file or its substitute defined by the `MAIN_PROTO_PATH` variable.



### Adapter plugin

Adapters should be kept as small and focused as possible.
In case of more complex adapters, it is highly recommended to keep them modular and extensible by plugins.
However, the interface between adapters and plugins depends on the adapter implementation.
