---
layout: post
title: Creating a custom AWS Lambda Runtime for LUA
categories: [ aws, lua, lambda]
---
{% raw %}
In this post take a look at a simple aws lambda runtime implementation
for Lua.

AWS Lambda is the serverless platform offered by Amazon, which lets developers
create services without having to worry about provisioning and managing servers.
You can learn more about AWS Lambda in the [official documentation](https://docs.aws.amazon.com/lambda/index.html)

Out-of-the-box AWS Lambda comes with support for a series of languages, which 
includes python, node.js, Go, Rust, Ruby, Java. 
More interestingly, it offers developers the ability to implement a **custom
runtime** for any programming language, so that if your favourite programming
language is not supported you can still use it to create Lambda functions.

## Lua and LuaJIT

Lua is a lightweight programming language, used mostly for embedded use in
applications. For instance, both [Redis](https://redis.io) and 
[Tarantool](https://taratool.io/) use Lua as a scripting language; they however
differ in the version they use: while redis uses the standard Lua interpreter,
Taratool opts to use the more performant LuaJIT. For my runtime I chose to use
LuaJIT, which should have a smaller memory footprint and faster execution times,
with consequent lower aws lambda costs.

## Custom Runtime & AWS Lambda Runtime

There are two runtimes which I am going to talk about: the AWS Lambda
runtime, and the custom runtime. The AWS Lambda runtime provides functions that
the custom runtime can use to get information about the invocation of the
function and to return the result of the function

![](/assets/images/lambda-runtime.png)
*Diagram showing how the custom runtime, the AWS Lambda runtime and the Lambda
function interact*

We have two options when creating a custom runtime: we can implement it as a
[AWS Lambda Layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)
or we can implement it as a library that will then be included in the function
executable. The first option works well with interpreted languages, as you can
include the full interpreter in the layer, so that it can be shared between
all the functions that use the layer. For this reason, the Lua custom runtime
will be implemented as a Lambda Layer.

## Lambda Layer

In this section I discuss the actual code of the custom runtime. You can find
all the code I reference in this repo: 

### Contents of the Layer

For the custom runtime I need the following:

* Lua interpreter (LuaJIT)
* Lua HTTP library
* Boostrap script
* Runtime script

#### Lua interpreter

As I mentioned before, I decided to use the LuaJIT interpreter, which can be
downloaded at [http://luajit.org/download/](http://luajit.org/download/)

#### Lua HTTP library

In order to communicate with the AWS Lambda runtime, my custom runtime needs an
http library. I used `luasocket`; the easier way to install it is using
`luarocks`, so I needed to install it as well. 

`luarocks` is useful to install many additional libraries; I used it to install
the `dkjson` library too, which can be used from the lambda function to decode
JSON events received for instance when the lambda is invoked through the AWS API
Gateway.

#### Bootstrap Script

The bootstrap script sets some configuration environment variables and executes
the LUA interpreter, running the runtime script:

```shell
#!/bin/sh
cd $LAMBDA_TASK_ROOT
export LUA_PATH="/opt/runtime/?.lua;/opt/rocks/share/lua/5.1/?.lua;;"
export LUA_CPATH="/opt/rocks/lib/lua/5.1/?.so;;"
/opt/bin/luajit /opt/runtime/runtime.lua
```

#### Runtime Script

The runtime script does some initialization, loads the user-provided function
and then starts a loop to process the lambda invocations. I created a separate
file to encapsulate the interaction with the AWS Lambda Runtime.

The `main` function takes care of the initialization:

```lua
local json = require "dkjson"
local lambda_runtime = require "lambda"

--- initialize the runtime and start the event loop
function main()
    -- INITIALIZATION
    -- this is needed to save `print` output to cloudwatch
    io.stdout:setvbuf("no")
    local handler_name = os.getenv("_HANDLER")
    local i = string.gmatch(handler_name, "[^.]+")
    local file_name = i()
    local func_name = i()
    -- check for errors
    local loaded, module = pcall(require, file_name)
    if not loaded or module[func_name] == nil then
        local err = {
            errorMessage=module,
            errorType=InvalidFunctionException
        } 
        lambda.init_error(err)
        return
    end
    local handler = module[func_name]
    while true do
        handle_event(handler)
    end
end
```

The actual handler loop only takes a few lines of code:

```lua
-- handle the communication with the lambda runtime interface
function handle_event(handler)
    local event, context = lambda_runtime.next_invocation()
    local success, res = pcall(handler, event, context)
    if success then
        lambda_runtime.send_response(context.request_id, res)
    else
        lambda_runtime.invocation_error(context.request_id, res)
    end
end
```

This function wraps the call to the handler with `pcall`, catching possible
errors and notifying the AWS Lambda runtime when they occur; in the case
everything works and there is no error, it forwards the response.

The `next_invocation` function takes a few more lines, because it needs to
create the `context` object.

```lua
local http = require "socket.http"

--- get the next invocation to process
-- @return the event received by the lambda function
-- @return @{context} 
function lambda_runtime.next_invocation()
    local event, _, headers = http.request(runtime_api .. "/invocation/next")
    local context = {
        request_id=headers["lambda-runtime-aws-request-id"],
        deadline=headers["lambda-runtime-deadline-ms"],
        invoked_function_arn=headers["lambda-runtime-invoked-function-arn"],
        client_context=headers["lambda-runtime-client-context"],
        cognito_identity=headers["lambda-runtime-cognito-identity"]
    }
    return event, context
end

```

The other functions are actually just wrappers around `http.request` calls.

There are a few things that are missing from my implementation:

* The AWS XRay environment variable is not set. This should not be really
  important, since there is no official AWS SDK for Lua
* The context object only includes the info retrieved from the invocation
  headers. To get other information, the environment variables can be used
  directly[^2]

### Building the Layer

When building the layer, one thing to keep in mind is that any executable that
is dynamically linked on your system may not work once uploaded to AWS Lambda,
because there may be different versions of libraries installed. 

To solve this problem, I took advantage of docker to build everything, using the 
official Amazon Linux Image, which is the same OS on top of which lambda 
functions are executed[^1].

The Dockerfile I use to build the layer can be found in the repo I mentioned
above, along with a build script that downloads the LuaJIT and luarocks 
packages.

## Conclusion

In this post I described how I implemented a custom AWS Lambda runtime for Lua;
the concepts I described can be used to create custom runtimes for other 
interpreted languages as well. 

Maybe in a future post I will describe how to create a custom runtime for a
compiled language. Feel free to reach out if you have any question or comment!

[^1]:[Lambda Runtimes](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)
[^2]:[Environment Variables Available to Lambda Functions](https://docs.aws.amazon.com/lambda/latest/dg/lambda-environment-variables.html)
{% endraw %}