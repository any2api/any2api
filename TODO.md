# any2api TODO

* Feature: support adapters that consume multiple gRPC endpoints, e.g. to consolidate their interfaces

* Refine README.md
  * Add optional label `org.any2api.api-volume` (instead of `org.any2api.api-port`) to adapters to specify a volume exposed by the adapter for filesystem-based APIs
  * Draw stack of lego bricks to visualize idea of stacked API bricks: http://publicdomainvectors.org/photos/Daily-Boxes-8.png
  * Describe how to install/run any2api CLI using npm and Docker

* Refine SCHEMA.md
  * Additional (alternative) wrap.yml schema for wrapping existing non-gRPC APIs (hosted services or containers); point to existing non-gRPC app containers with port info etc.; extend stack.yml schema to cover these non-gRPC apps
