# Schema definitions



## `any2api-stack.yml` schema

An `any2api-stack.yml` file specifies a stack of API bricks by composing gRPC apps and API adapters.
Its schema is shown in the following:

``` plaintext
{
  "apps": {
    (name): {
      "image": (string),            # see Docker Compose
      "build": (string) | (object), # see Docker Compose
      "environment": (object)       # see Docker Compose
    }
  },

  "adapters": {
    (name): {
      "image": (string),            # see Docker Compose
      "build": (string) | (object), # see Docker Compose
      "environment": (object),      # see Docker Compose
      "stacked_on": (name)
    }
  }
}
```



## `meta.yml` schema

A `meta.yml` file complements `.proto` files with additional metadata.
Its schema is shown in the following:

``` plaintext
{
  "title":       (string),
  "description": (string),

  "messages": {
    (messageName): {
      "description": (string),
      "fields": {
        (fieldName): {
          "description": (string),
          "default":     (string),
          "mime_type":   (string),
          "in": [
            "header" | "body" | "body-exclusive" |
            "query" | (any)
          ]
        }
      }
    }
  },

  "services": {
    (serviceName): {
      "description": (string),
      "metadata": {
        (metadataName): {
          "description": (string),
          "default":     (string), # default value
          "example":     (string)  # example value
        }
      }
    }
  },

  "methods": {
    (methodName): {
      "description": (string),
      "metadata": {
        (metadataName): {
          "description": (string),
          "default":     (string), # default value
          "example":     (string)  # example value
        }
      },
      "rest": {
        "resource_operation":  (string), # CRUD
        "resource_address": {
          "resource_id_field": (fieldName),
          "resource_name":     (string),
          "resource_parent":   (string),
          "resource_path":     (string)
        },
        "created_address": {
          "resource_id_field": (fieldName),
          "resource_name":     (string),
          "resource_parent":   (string),
          "resource_path":     (string)
        }
      }
    }
  }
}
```

The keys `(messageName)`, `(serviceName)` and `(methodName)` are fully qualified names regarding the definitions in the associated `.proto` files, for example, `myService.myMethod` or `myPackage.helloMessage.nestedMessage`.



## `wrap.yml` schema

A `wrap.yml` file specifies how to wrap an executable such as a Python script or Java JAR executable as a gRPC app using a wrapper.
Its schema is shown in the following:

``` plaintext
{
  "title":       (string), # title of resulting gRPC app
  "description": (string),
  "wrapper":     (string), # name of a specific wrapper

  "operations": {
    (operationName): {
      "description":     (string),
      "readme_file":     (string), # path to README file
      "service":         (string), # gRPC service name
      "longrunning":     (boolean),

      "parameters": {
        (parameterName): {
          "description": (string),
          "type":        (string), # JSON or proto3 type
          "proto":       (string), # custom proto3 type def
          "default":     (any),    # default value
          "map_to":      "env:…" | "stdin" | "file:…" | (any),
          "streamable":  (boolean),
          "updatable":   (boolean) # allow change after start
        }
      },

      "results": {
        (resultName): {
          "description": (string),
          "type":        (string), # JSON or proto3 type
          "proto":       (string), # custom proto3 type def
          "map_from":    "stdout" | "file:…" | (any),
          "streamable":  (boolean),
          "chunk_delim": (string)  # complete chunks only
        }
      },

      "commands" {
        "prepare":       (string), # shell command
        "invoke":        (string)  # shell command
      }
    }
  }
}
```
