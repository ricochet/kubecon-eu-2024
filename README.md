# wasi-cloud Wasm Components

## Problem

How many typical microservices can move between clouds? And why would you want to?

1. **Availability and scalability**: By moving microservices to different clouds, you
can ensure that your application is available even if one cloud experiences an
outage or goes down for maintenance. In case of an disaster or unexpected outage, you may want to
move some or all of your microservices to a different cloud to ensure business
continuity. By having a backup plan in place, you can minimize the impact on your
customers and revenue.
1. **Cost optimization**: Different clouds have different pricing models, and some may
be more cost-effective than others for certain workloads.
1. **Security and compliance**: Depending on the sensitivity of your data and the
regulatory requirements you need to meet, you may want to move certain
microservices to a cloud that offers better security and compliance features.
Depending on the regulatory requirements, there may be strict requirements for
data locality and sovereignty.
1. **Ability to leverage the strengths of each cloud**: For example, if you're
using AWS for compute and storage, but Google Cloud for machine learning services,
you can move the relevant microservices to Google Cloud to take advantage of its
specific capabilities.

A decentralized architecture is well-suited for WebAssembly Components.

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
cd golang/hello

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
package wasmcloud:hello;

world hello {
  export wasi:http/incoming-handler@0.2.0;
}
```

### Local dev v2 (3 min)

Now that we have a "hello world" out of the way, let's step this up a notch by targeting a different subset of `wasi:cloud/core` as our world.

Taking the hello example, we can add `import`s for `wasi:keyvalue` and `wasi:blobstore` to the wit definition:

```wit
package wasmcloud:hello;

world hello {
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


### Running LLM on the edge

```bash
Step 2: Download the Gemma-7b-it model GGUF file. Since the size of the model is 5.88G so it
curl -LO https://huggingface.co/second-state/Gemma-7b-it-GGUF/resolve/main/gemma-7```

https://github.com/second-state/llama-utils/tree/main/chat
```

### Cloud development (8 min)

Zhou deployed in Azure against AKS and OpenAI

`wasmcloud/crates/providers`


### LLM

```bash
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- --plugins wasi_nn-ggml
```

### Diagrams

Mossaka: implement azure blobstore in

## Talk track is 25 min

Why is this so exciting? This flexible solution gives us the ability to design around data locality.

Use-cases for this technology:
- We showed building with Golang and Rust.
- But many more languages are supported including js/ts, python, and more.
- The types of interfaces available are not restricted to WASI.

Participate and give feedback and have you as reference implementation. Zhou created an Azure blob storage implementation for this talk. The ecosystem is ready to grow with new providers, interfaces, and use-cases.
