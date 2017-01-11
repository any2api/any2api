# any2api: the better way to create awesome APIs 🚀

**any2api** aims to establish an open ecosystem of composable *API bricks* such as reusable wrappers and adapters.
Instead of using basic API development frameworks to build specific kinds of APIs from scratch (e.g. REST or messaging APIs), the API bricks simplify the process of creating diverse APIs.
For example, a Python wrapper is used to bring a simple *.py* script into the ecosystem.
Then, a specific adapter such as a REST adapter is utilized to expose the script's functionality through an API.
Intermediate adapters can optionally add more functionality to the provided API such as authentication.
In addition, another adapter can be used to make the same functionality available in a different way, fostering API diversity:

*API diversity is key because there is no 'one-fits-all' kind of API. In certain cases, REST is a good choice, but sometimes messaging, RPC or streaming APIs work much better.*

API bricks are generic, reusable and configurable.
They implement best practices how certain kinds of APIs are built, for example, how to implement long-running tasks with REST or how to emulate a synchronous invocation with a messaging API.
This helps to avoid common and reoccurring issues that happen a lot when developing APIs from scratch.

APIs are key for many kinds of applications, for example, microservices.
They follow the principle of *smart endpoints (APIs) and dumb pipes*.
However, implementing these smart endpoints isn't trivial.
Furthermore, a microservice can have multiple endpoints such as a messaging API for internal communication and a REST API as external interface.
any2api simplifies the creation of smart endpoints by composing API bricks instead of starting from scratch for each endpoint of each microservice.



## This repo: any2api core & CLI

This repository contains the core framework and CLI to create APIs based on the any2api ecosystem.
API bricks such as adapters and wrappers are delivered through separate (public or private) repositories.



## Ecosystem foundation: containers & gRPC

Since the any2api ecosystem is about composing API bricks to create awesome APIs, each building block must implement and respect specific interfaces.
To foster tech diversity and allow the implementation of composable API bricks using different technologies, the technical foundation of the ecosystem are containers and gRPC:

* API bricks are packaged as Docker/OCI containers. Therefore, established container deployment/orchestration tools and platforms can be used such as Kubernetes and Docker Compose.

* API bricks communicate via Protocol Buffers (proto3) and gRPC, a high performance HTTP/2-based RPC framework.
  * http://developers.google.com/protocol-buffers/docs/reference/proto3-spec
  * http://developers.google.com/protocol-buffers/docs/proto3
  * http://www.grpc.io/docs/guides/concepts.html

There are different kinds of API bricks, mainly **gRPC apps**, **wrappers** and **adapters**.
They are combined and bundled as **API stacks**.



### API stack

API bricks can be combined and stacked like Lego bricks to build and bundle API stacks.
For example, a REST API adapter can be stacked on top of a gRPC app to expose application/business logic provided by the app through an HTTP-based RESTful interface.
Technically, an API stack is specified by a `any2api-stack.yml` file.
See [schema definition](SCHEMA.md#any2api-stackyml-schema).



### gRPC app

A gRPC app exposes specific functionality (application/business logic) through a gRPC endpoint, which is specified by a `.proto` file.
To ease the composition of gRPC apps and API adapters, it is highly recommended to run the app inside a Docker/OCI container.
The interface of a gRPC app container:

* A gRPC app container *must* own the label `org.any2api.kind="app"`.
  * In case of Docker, the `Dockerfile` of the gRPC app container *must* include the statement `LABEL org.any2api.kind="app"`.

* A gRPC app *must* provide the `/api/main.proto` file to specify its gRPC interface.

* The `/api` directory *must* be provided as a shared container volume by the gRPC app container. It is used by API adapters.
  * In case of Docker, the `Dockerfile` of the gRPC app *must* include the statement `VOLUME /api`.

* The `/api` directory *can* optionally contain further files such as `/api/meta.yml` (see [schema](SCHEMA.md#metayml-schema)) to specify additional metadata, which may be read and considered by API adapters.
  * For example, a REST API adapter may require additional information on how to properly map and group RPC operations to resources and their associated CRUD operations.

* By default, a gRPC app exposes the gRPC endpoint described by the `/api/main.proto` file on TCP port `50051`. If the gRPC app exposes its gRPC API on a different port, the container *must* own the `org.any2api.api-port` label with the corresponding port number as value.
  * In case of Docker, the `Dockerfile` of the gRPC app container *must* include the statement `LABEL org.any2api.api-port="…"`.

gRPC apps can be developed from scratch using any programming language that is supported by the gRPC framework.
Alternatively, wrappers can be used as described in the following to provide gRPC endpoints for existing code/executables/APIs/… in a dynamic and automated fashion.



### Wrapper

A wrapper is a special kind of gRPC app container, so its interface follows the previously described constraints.
However, it does not implement specific application/business logic, but provides generic logic to wrap a specific kind of executable/API/… and exposes its functionality through a gRPC endpoint.
For example, a Python wrapper can transparently expose a gRPC endpoint for any given Python script.

As a result, wrappers further reduce API development efforts because you just implement the application logic using a language and technology of your choice.
Then, you utilize an appropriate wrapper to make a gRPC endpoint out of it.
Now you can instantly use and benefit from the entire any2api ecosystem without having to develop an API from scratch.

Technically, a wrapper is a base container to be refined, so that it bundles the wrapping logic with an existing executable/API/…; the resulting bundle is delivered as a gRPC app container.
In case of Docker, the `Dockerfile` of the wrapper includes the statements `ONBUILD ADD . /wrap` and `ONBUILD RUN cp -a /wrap/api /api 2>/dev/null || :` to add all relevant files from the build context directory to the `/wrap` and `/api` directories of the resulting gRPC app container.
Providing an `/api` directory (intended for adapters) as part of the build context is optional because it could be dynamically generated by the wrapper.
Additional `ONBUILD RUN …` statements are typically included in the base container, for example, to generate the `/api/main.proto` file and to wire the wrapping logic with the given executable.
All `ONBUILD` statements are executed when building a gRPC app container based on the wrapper container.
Finally, the statements `LABEL org.any2api.kind="wrapper"` and `ONBUILD LABEL org.any2api.kind="app"` label the base container as wrapper and any container built on top as gRPC app.

The build context directory that is added as `/wrap` directory to the resulting gRPC app container includes the code/executables/… to be wrapped and exposed through the gRPC endpoint.
Highly sophisticated wrappers would automatically analyze the provided code to identify and extract metadata such as runtime dependencies and input parameters.
Alternatively, such metadata can be declaratively specified by a `wrap.yml` file located in the build context directory.
See [schema definition](SCHEMA.md#wrapyml-schema).

The structure and content of `wrap.yml` files vary depending on the specific wrapper.
For example, a wrapper that wraps an existing REST API and exposes its functionality through a gRPC endpoint may refer to a Swagger definition, so the corresponding `wrap.yml` file with its parameter and result mappings may look different.



### Adapter

As described previously, a gRPC app exposes specific functionality (application/business logic) through a gRPC endpoint.
Such an app can either be developed from scratch or existing code/executables/… can be transformed into gRPC apps using wrappers.
Either way, gRPC endpoints are one kind of API that can be used by other applications.
However, in the sense of API diversity, other kinds of APIs such as REST and messaging are more appropriate for certain usage scenarios.
Therefore, API adapters (REST, RabbitMQ, JSON-RPC, etc.) are connected with gRPC apps to serve diverse APIs.

An API adapter does not always translate to different kinds of APIs such as gRPC-REST, gRPC-RabbitMQ, etc.
There are also intermediate adapters that take a gRPC endpoint and expose another gRPC endpoint.
gRPC-to-gRPC adapters can implement diverse middleware functionality such as reshaping gRPC endpoints (e.g. consolidating and filtering operations), authentication, request rate limiting, monitoring, analytics, content transformation, etc.

An intermediate adapter can be used in conjunction with any gRPC app and any other API adapter, including other intermediate adapters.
As a result, multiple adapters can be stacked, for example: gRPC app ↔ monitoring adapter ↔ rate limit adapter ↔ authentication adapter ↔ REST API adapter.

To ease the composition of gRPC apps and API adapters, it is highly recommended to run an adapter inside a Docker/OCI container.
The interface of an adapter container:

* An adapter container *must* own the label `org.any2api.kind="adapter"`; in case of an intermediate adapter, the label *must* be `org.any2api.kind="intermediate-adapter"`. If the adapter itself exposes an API endpoint, the container *must* own the `org.any2api.api-port` label with the corresponding port number as value. Optionally, the label `org.any2api.info-port` *can* be defined with an HTTP port number as value to provide API documentation, status information, etc.
  * In case of Docker, the `Dockerfile` of the API adapter container *must* include the statement `LABEL org.any2api.kind="adapter" …` to specify the required labels.

* An adapter *can* own the `org.any2api.keywords` label to specify properties of the provided API. The label value is a comma-separated list of keywords.
  * Keywords to specify protocols and data formats: `http`, `amqp`, `mqtt`, `grpc`, `xml`, `json`, `json-rpc`, …
  * Keywords to specify capabilities: `sync`, `async`, `rest`, `rpc`, `event-driven`, `messaging`, `streaming`, `brokered`, …

* The `/api` directory is a shared container volume. It is typically provided by the underlying gRPC app container (or intermediate adapter container) and used by the API adapter container.
  * The environment variable `API_DIR` *can* optionally be provided to the adapter to use another directory instead of `/api`. An adapter *must* process and respect this variable if it is set.

* The `/api/main.proto` or `$API_DIR/main.proto` file is the *only* file required by an adapter. Therefore, an adapter *must* work properly based on this file without any additional metadata.

* The `/api` directory (or its substitute `API_DIR`) *can* optionally contain further files such as `/api/meta.yml` (see [schema](SCHEMA.md#metayml-schema)) to specify additional metadata to be considered by the adapter.
  * For example, a REST API adapter may require additional information on how to properly map and group RPC operations to resources and their associated CRUD operations.

* An adapter *must* understand all proto3 and core gRPC features.

* An adapter *must* read the environment variables `GRPC_HOST` (IP address or hostname) and `GRPC_PORT` (port number) to connect to the underlying gRPC endpoint.

* An adapter *must* delay its start until the underlying gRPC endpoint (provided by a gRPC app or another adapter) is available. This behavior can be implemented using [dockerize](https://github.com/jwilder/dockerize) by the `Dockerfile` statement `CMD dockerize -wait tcp://$GRPC_HOST:$GRPC_PORT …`. Only after the availability of the underlying gRPC endpoint, the adapter can be sure that the required `main.proto` file exists.

Furthermore, the interface of an intermediate adapter container must follow additional rules:

* An intermediate gRPC-to-gRPC adapter *must* also implement the interface of a gRPC app container to enable other adapters to run on top.

* Intermediate gRPC-to-gRPC adapters potentially modify the originally provided gRPC endpoint's interface. Therefore, they may change files inside the `/api` directory (or its substitute `API_DIR`) such as `main.proto` and `meta.yml`. By default, existing files are immediately modified and overridden.
  * If the environment variable `UPDATED_API_DIR` is provided to the adapter container with a different path than `/api` or `API_DIR`, the adapter *must* duplicate the existing directory and apply all modifications to the copied directory. The content of the original directory is not modified. The path defined by `UPDATED_API_DIR` *should* be a subdirectory of `/api` or `API_DIR` to place it inside the shared container volume.



### Adapter plugin

Adapters should be kept as small and focused as possible.
In case of more complex adapters, it is highly recommended to keep them modular and extensible using plugins.
The specific interface between adapters and plugins depends on the adapter implementation.
