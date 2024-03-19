# wasi-cloud Wasm Components

## Problem

How many typical microservices can move between clouds? And why would you want to?

### Problem 1: business logic portability

The core business logic needs to be adaptable across different platforms without requiring significant modifications

Wasm component & WASI: language neutrality and platform neutrality nature of the CM allow core business logic to be encapsulated within components and reducing the needs to be re-implemented in different languages.

### Problem 2: decoupling from service providers

Cloud providers and platforms offer vendor lock-in features to the business logic

Wasm components & WASI: module linking and language neutrality of the CM allow core business logic to compose and interoperate with cloud providers using a standard interface / WIT.

### Problem 3: maintainability and scalability

Existing solutions like sidecar container decouples the core business logic from service providers by putting the capabilities as sidecar containers. However they require HTTP or gRPC comm and the fact that they are deployed to one pod which can be a bottleneck for scalability and maintainability

Wasm components & WASI: No separate long-live services and communicate over a network and function call in Wasm should be incredibly cheap compared to HTTP or gRPC with overheads from protocol serialization & deserialization

## Concepts

- WASI and Components
- wasi-cloud

### WASI 0.2.0

WASI 0.2.0 launched January 24th 2024 along with the Component Model.

Originally, WASI stood for the **W**eb**A**ssembly **S**ystems **I**nterface, a monolithic ABI that provided necessary API's that host runtimes implemented to allow running WebAssembly modules outside the web runtimes.

Unlike WASI Preview 1, WASI 0.2 is a set of **versioned modular interfaces** defined via the WIT (WebAssembly Interface Types) IDL. Built atop the Component Model spec, WASI 0.2 is designed with these higher level interfaces, with key features including support for async and large worloads via the `streams` type defined in `wasi:io`, UDP and TCP sockets via `wasi:sockets`, and HTTP requests and responses via the incoming and outgoing handler interfaces in `wasi:http`. In addition to `interface`s and `type`s, WASI via the component model supports another concept called a `world`.

A `world` is effectively the type of WebAssembly component. A component build will target a `world` which defines the necessary `import`s the component needs to run and the interfaces the component is expected to `export`. A different way to describe a `world` is that it is like a target with specific configuration for linkable features. WASI 0.2.0 launched with two worlds, `wasi:cli/command` and `wasi:http/proxy`.

### wasi-cloud (3 min)

`wasi-cloud:core` is a proposed world that builds on WASI 0.2 foundations.

We’ve discussed this before:

- [WASI and the Cloud, Jiaxiao (Joe) Zhou and Dan Gohman](https://www.youtube.com/watch?v=5WQRT62V_VU)
- [The Future of Cloud Computing with WebAssembly by Bailey Hayes and Dan Chiarlone](https://www.youtube.com/watch?v=Z7cSjIp7vRg)
- [Taking WASI-Cloud-Core for a Spin, Jiaxiao (Joe) Zhou and Joel Dice](https://www.youtube.com/watch?v=W-ubCAMAJQc)

One of the key design philosophies around `wasi-cloud:core` is the 80/20 rule, where we aim for the interfaces defined in `wasi/cloud:core` to solve ~80% of the use-cases for cloud native applications.

## Demo

### Basics (5 min)

To get a feel for how to build a component that targets a world, let's start simple and build a Go WebAssembly Component starting with targeting `wasi:http/proxy`.

Since `wasi:cloud/core` is an early phase 1 proposal, many runtimes do not yet support `wasi:cloud/core` interfaces or perhaps support different versions of the interfaces in draft. It is important to note that `wasi:http/proxy` is a world that is `include`d into `wasi:cloud/core`. So while we aren't yet targetting the `wasi:cloud/core` world, we are using part of a `wasi:cloud` component is expected to use. `wasi:http/proxy` was part of the launch of WASI 0.2, so we can use either of the reference implementations for `wasi:http/proxy`, JCO (Node.js with shims) or wasmtime.

```bash
cd components/golang/hello

# easiest to build with wash, the "wasmCloud shell"
wash build

# Which runs:
# tinygo build with the wasi flags
# embeds necessary component metadata
# adapts to a component with the wasi .wasm adapter

# run the component
wasmtime serve -Scommon ./build/http_hello_world_s.wasm

# call the component from another terminal
curl localhost:8080
```

```wit
package gopher:hello;

world hello {
  export wasi:http/incoming-handler@0.2.0;
}
```

### Local dev v2 (3 min)

Now that we have a "hello world" out of the way, let's step this up a notch by targeting a different subset of `wasi:cloud/core` as our world.

Taking the hello example, we can add `import`s for `wasi:keyvalue` and `wasi:blobstore` to the wit definition:

```wit
package wasmday:gopher;

world gopher {
  import wasi:keyvalue/eventual@0.2.0-draft;
  import wasi:blobstore/blobstore@0.2.0-draft;
  export wasi:http/incoming-handler@0.2.0;
}
```

Note that this API is in draft and is undergoing active changes is versioned at `0.2.0-draft`.

Then after generating the language bindings, import the bindings and add to our business logic from before.

This time we are going to use CNCF wasmCloud to run our component. wasmCloud is a CNCF project that uses the wasmtime WebAssembly runtime to execute WebAssembly Components. We are using wasmCloud in this case because wasmCloud allows
runtime extensibility for providers and today we wasmtime does not have a builtin keyvalue or blobstore provider.

```bash
cd components/golang/hello-services
wash build
wash app deploy wadm.yaml

curl localhost:8080

```

Let's take a look at what we linked the component to in our manifest.

```yaml
```

Now load some data that our component will use:

```bash
redis-server

# in a separate tab
redis-cli -h 127.0.0.1 -p 6379
HSET user:ricochet accept-language en-US animal dog
HSET user:mossaka accept-language zh-CN animal cat
```

For redis, the source code would typically look like:

```go
err := rdb.Set(ctx, "user:ricochet", "animal dog", 0).Err()
  if err != nil {
    panic(err)
  }

  fmt.Println("OK")

  value, err := rdb.Get(ctx, "user:ricochet").Result()
  if err != nil {
    panic(err)
  }
  fmt.Printf("Favorite animal is %s", value)
```

### Cloud development

```bash
export WASMCLOUD_LATTICE=<lattice id>

# temporary while wadm isn't yet released for 1.0
cd ~/repos/wasmCloud/wadm
cargo run --features cli --release

az aks get-credentials --resource-group wasmday_group --name wasday --overwrite-existing
kubectl port-forward svc/nats-cluster 4222
```

### Diagrams

Mossaka: implemented azure blobstore

## Talk track is 25 min

Why is this so exciting? This flexible solution gives us the ability to design around data locality.

Use-cases for this technology:

- We showed building with Golang and Rust.
- But many more languages are supported including js/ts, python, and more.
- The types of interfaces available are not restricted to WASI.

Participate and give feedback and have you as reference implementation. Zhou created an Azure blob storage implementation for this talk. The ecosystem is ready to grow with new providers, interfaces, and use-cases.
