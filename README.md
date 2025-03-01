## octoml-profile

octoml-profile is a python library and cloud service
designed to provide the **simplest experience** for assessing and optimizing the performance of PyTorch models on cloud hardware with state-of-the-art ML acceleration technology.

It is suited for benchmarking PyTorch based AI applications before
they are deployed into production.

### Documentation quick links

* [Why octoml-profile?](#why-octoml-profile)
* [Installation](#installation)
* [Getting started](#getting-started)
* [Behind the scenes](#behind-the-scenes)
* [Data privacy](#data-privacy)
* [Known issues](#known-issues)
* [Contact the team](#contact-the-team)

### Example required code change

![Distilbert Example](assets/distilbert-diff.png)

### Example results

```
    Runs discarded because compilation occurred: 1
    Profile 1/1:
    Segment                       Runs  Mean ms  Failures
    =====================================================
    0  Uncompiled                    9    0.051

    1  Graph #1
        r6i.large/onnxrt-cpu        90    0.037         0
        g4dn.xlarge/onnxrt-cuda     90    0.135         0

    2  Uncompiled                    9    0.094
    -----------------------------------------------------
    Total uncompiled code run time: 0.145 ms
    Total times (compiled + uncompiled) per backend, ms:
        r6i.large/onnxrt-cpu     0.182
        g4dn.xlarge/onnxrt-cuda  0.279

```


## Why octoml-profile?

Benchmarking deep learning models for cloud deployment is an intricate and
tedious process. This challenge becomes more significant as the boundary
between model and code continues to blur. As we witness in the
rise of generative models and the increasing popularity of PyTorch, exporting
models from code and selecting the optimal hardware deployment
platform and inference backend becomes a daunting task even for expert ML engineers.

With octoml-profile, you can easily run performance/cost measurements on a wide
variety of different hardware and apply state-of-the-art ML acceleration
techniques, all from your development machine, using the same data and workflow used for
training and experiment tracking, without tracing or exporting the model!

Apply just a few code changes and run the code locally, and
you instantly get performance feedback on your model's compute-intensive
tensor operations

- on each hardware instance type
- with automatic application of state-of-the-art ML acceleration technologies
- your model's data shapes and types can vary

but without the burden of

- exporting the models and stitching them back with pre/post processing code
- provisioning the hardware
- preparing the hardware specific dependencies, i.e. the version of PyTorch, Cuda, TensorRT etc.
- sending the model and data to each hardware and running the benchmarking script

This agile loop enables anyone familiar with PyTorch to rapidly iterate over
their model and the choice of hardware / acceleration technique to meet their
deployment goals without "throwing the model over the fence and back".


## Installation

- You can create a login and generate an API token by going to profiler.app.octoml.ai

- Once you have your token, set it as the env var below.
  ```
  export OCTOML_PROFILE_API_TOKEN=<access token>
  ```
- Create and activate a python virtual environment. Make sure that you are using python3.8.

  using virtualenv:
  ```
  python3.8 -m venv env
  . env/bin/activate
  ```

  or using conda:
  ```
  # Instructions on installing conda on MacOS: https://docs.conda.io/projects/conda/en/latest/user-guide/install/macos.html
  conda create -n octoml python=3.8
  conda activate octoml
  ```
- Install torch2.0 stable version
  ```
  pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
  ```
  Alternatively, install torch-nightly if you want to use the experimental dynamic shape feature.
  ```
  pip install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cpu
  ```
- Install octoml-profile
  ```
  pip install octoml-profile
  ```


## Getting Started

All example code, including applications from `transformers`, can be found at [examples/](examples).

Below is a very simple example that shows how to integrate octoml-profile
into your model code.

```python
import torch
import torch.nn.functional as F
from torch.nn import Linear, ReLU, Sequential
from octoml_profile import (accelerate,
                            remote_profile,
                            RemoteInferenceSession)

model = Sequential(Linear(100, 200), ReLU(), Linear(200, 10))

@accelerate
def predict(x: torch.Tensor):
    y = model(x)
    z = F.softmax(y, dim=-1)
    return z

# Alternatively you can also directly use `accelerate`
# on a model, e.g. `predict = accelerate(model)` which will leave the
# softmax out of remote execution

session = RemoteInferenceSession()
with remote_profile(session):
    for i in range(10):
        x = torch.randn(1, 100)
        predict(x)
```

Running this program results in the following output that shows 
times of the function being executed remotely on each backend.
```
    Runs discarded because compilation occurred: 1
    Profile 1/1:
    Segment                       Runs  Mean ms  Failures
    =====================================================
    0  Uncompiled                    9    0.028

    1  Graph #1
        r6i.large/onnxrt-cpu        90    0.013         0
        g4dn.xlarge/onnxrt-cuda     90    0.088         0

    2  Uncompiled                    9    0.013
    -----------------------------------------------------
    Total uncompiled code run time: 0.042 ms
    Total times (compiled + uncompiled) per backend, ms:
        r6i.large/onnxrt-cpu     0.055
        g4dn.xlarge/onnxrt-cuda  0.130
```
You can think of `Graph #1` as the computation graph which captures
`model` plus `softmax` in the `predict` function. See
the [Uncompiled Segments](#uncompiled-segments) for more information
on the uncompiled blocks.

`Graph #1` shows 90 runs because the `for loop` runs the
`predict` function 10 times. On each loop iteration the model is evaluated
remotely 10 times. However, the result of the first remote run is
discarded because compilation is triggered.

To understand what's happening behind the scenes, read on. To see
more examples, see [examples/](examples).

## Behind the scenes

* [How octoml-profile works](#how-octoml-profile-works)
* [Where `@accelerate` should be applied](#where-accelerate-should-be-applied)
* [The profile report](#the-profile-report)
* [Quota](#quota)
* [Supported backends](#supported-backends)
* [Uncompiled segments](#uncompiled-segments)
* [Experimental features](#experimental-features)

### How octoml-profile works

In the example above, we first decorate the `predict` function with the
`@accelerate` decorator. Whenever the function is executed, the decorator uses
[Torch Dynamo](https://pytorch.org/tutorials/intermediate/dynamo_tutorial.html)
to extract one or more computation graphs, and offload them to the remote
inference worker for execution. These are referred to as `compiled code runs`
in the output.

Code that is not PyTorch computation graphs cannot be offloaded -- such code
runs locally and is shown as `uncompiled code run`. For more details on uncompiled code see
the [uncompiled segments section](#uncompiled-segments) below.

The `RemoteInferenceSession` is used to reserve
exclusive hardware access specified in the `backends` parameter.
If there are multiple backends, they will run in parallel.

The beauty of this example is that the decorator's scope
can be larger than the scope of PyTorch models, whose boundaries
are difficult to carve out exactly.

The `predict` function may contain pre/post processing code, non tensor logic
like control flows, side effects, and multiple models. Only eligible graphs
will be intelligently extracted and offloaded for remote execution.

From a user's perspective the "Total times (compiled + uncompiled) per backend"
is the best estimation of the runtime of decorated function with the chosen 
hardware platform and acceleration library.

### Where `@accelerate` should be applied
In general, `@accelerate` is a drop-in replacement for `@torch.compile`
and should be applied to function which contains PyTorch Model that performs inference.
When the function is called under the context manager of `with remote_profile()`,
the remote execution and profiling activated. When called without `remote_profile()`
it behaves just as TorchDynamo.

If you expect the input shape to change especially for generative models,
see [Experimental features](#experimental-features).

By default, `torch.no_grad()` is set in the remote_profile context to minimize usage of non
PyTorch code in the decorated function. This minimizes the chance of hitting
`TorchDynamoInternalError`.

Last but not least, `@accelerate` should not be used to decorate a function
that has already been decorated with `@accelerate` or `@torch.compile`.

### The profile report

By default, the `Profile` report table will show the linear sequence of subgraph segment runs.

However, when too many subgraphs are run, subgraph exeuction segments are aggregated per
subgraph, and a few subgraphs that have the highest aggregate runtimes are shown.
For example, a generative encoder-decoder based model that produces a large number of
run segments will display an abridged report displaying 
**runtime by subgraph** instead of **runtime by segment** by default.
In cases like this, you'll see:

```
Profile 1/1:
   Segment                Runs  Mean ms  Failures
=================================================

0  Graph #9             
     ryzen9/onnxrt-cpu     190    3.929         0
     rtx3060/onnxrt-cuda   190    1.752         0


1  Graph #4             
     ryzen9/onnxrt-cpu      10    5.011         0
     rtx3060/onnxrt-cuda    10    1.524         0


2  Graph #7             
     ryzen9/onnxrt-cpu      10    3.715         0
     rtx3060/onnxrt-cuda    10    1.437         0

-------------------------------------------------

9 total graphs were compiled
66 total compiled segments were run (a graph can run in multiple segments)
More than 3 compiled segments were run, so only graphs with the highest aggregate runtimes are shown.
```

Other graphs are hidden. If your output has been abridged in this way
but you want to see the full, sequential results of your profiling run, you can print
the report with `verbose`:

```python
# print_results_to=None silences the default output profile report.
with remote_profile(print_results_to=None) as prof:
    ...
prof.report().print(verbose=True)
```

### Quota

Each user has a limit on the number of concurrent backends held by the user's sessions.
If you find yourself hitting quota limits, please ensure you are closing previously held sessions
with `session.close()`. Otherwise, your session will automatically be closed at script exit.

### Supported backends

To programmatically access a list of supported backends, please invoke:

```python
RemoteInferenceSession().print_supported_backends()
```

**Supported Cloud Hardware**

AWS
- g4dn.xlarge (Nvidia T4 GPU)
- g5.xlarge  (Nvidia A10g GPU)
- r6i.large (Intel Xeon IceLake CPU)
- r6g.large (Arm based Graviton2 CPU)

**Supported Acceleration Libraries**

ONNXRuntime
- onnxrt-cpu
- onnxrt-cuda
- onnxrt-tensorrt

If no backends are specified while creating the `RemoteInferenceSession(backends: List[str])`, the default backends `g4dn.xlarge/onnxrt-cuda` and `r6i.large/onnxrt-cpu` are used for the session.

### Uncompiled segments

You will likely see `Uncompiled segments` in the profiling report. It is caused
by [graph breaks](https://pytorch.org/docs/master/dynamo/troubleshooting.html#graph-breaks)
in the decorated function.

It's easier to illustrate using an example
```python
import torch.nn.functional as F
import time

@accelerate
def function_with_graph_breaks(x):
    time.sleep(1) # Uncompiled segment 0
    x = F.relu(x) # graph #1, segment 1
    torch._dynamo.graph_break() # Uncompiled segment 2
    time.sleep(0.1) # continue segment 2
    x = F.relu(x) # graph #2, segment 3
    time.sleep(1) # Uncompiled segment 4
    return x

sess = RemoteInferenceSession('r6i.large/onnxrt-cpu')
with remote_profile(sess):
    for _ in range(2):
        function_with_graph_breaks(torch.tensor(1.))
```

```
Profile 1/1:
   Segment             Runs   Mean ms  Failures
===============================================
0  Uncompiled             1  1001.051

1  Graph #1
     r6i.large/onnxrt-cpu 10     0.007         0

2  Uncompiled             1   100.150

3  Graph #2
     r6i.large/onnxrt-cpu 10     0.007         0

4  Uncompiled             1  1000.766
-----------------------------------------------
```
In the example above, we explicitly insert `graph_break`.
In real case, TorchDynamo will automatically
insert graph break when encoutering code that cannot be captured
as part of the computation graph. As a side effect of
us relying on TorchDynamo graph capturing, even a simple function as the one
below will always have Uncompiled segment in the beginning and the end because 
we cannot differentiate if there is non graph user code before or after `relu`.

```python
@accelerate
def simple_function(x):
    # implicit uncompiled segment 0
    x = F.relu(x) # graph #1, segment 1
    # implicit uncompiled segment 2
    return x
```


Uncompiled segment is run locally once for every invocation. The total
uncompiled time is summed up and add to the estimated total time (compiled and
uncompiled).

To print graph breaks and understand more of what TorchDynamo is doing under the hood, see
the [dynamo.explain](https://pytorch.org/tutorials/intermediate/dynamo_tutorial.html#torchdynamo-and-fx-graphs)
and
[PyTorch
Troubleshooting](https://pytorch.org/docs/master/dynamo/troubleshooting.html#torchdynamo-troubleshooting)
pages.

### Experimental features

**dynamic shapes**

This feature is still under active development, so your results may vary. There are known bugs with using
dynamic shapes with the torch eager and inductor backends. To use this experimental feature, please install
a version of nightly torch more recent than 20230307.

```
pip install --pre "torch>=2.0.0dev" "torchvision>=0.15.0.dev" "torchaudio>=2.0.0dev" --extra-index-url https://download.pytorch.org/whl/nightly/cpu
```

Set `@accelerate(dynamic=True)` on any `accelerate` usage.

## Data privacy

We know that keeping your model data private is important to you. We guarantee that no other
user has access to your data. We do not scrape any model information internally, and we do not use
your uploaded data to try to improve our system -- we rely on you filing github issues or otherwise
contacting us to understand your use case, and anything else about your model.
Here's how our system currently works.

We leverage TorchDynamo's subgraph capture to identify Pytorch-only code, serialize those subgraphs,
and upload them to our system for benchmark.

On model upload, we cache your model in AWS S3. This helps with your development iteration speed --
every subsequent time you want profiling results, you won't have to wait for model re-upload on every
minor tweak to your model or update to your requested backends list. When untouched for four weeks,
any model subgraphs and constants are automatically removed from S3.

Your model subgraphs are loaded onto our remote workers and are cleaned up on the creation of
every subsequent session. Between your session's closure and another session's startup,
serialized subgraphs may lie around idle. No users can access these subgraphs in this interval.

If you still have concerns around data privacy, please [contact the team](#contact-the-team).

## Known issues

### Waiting for Session
When you create a session, you get exclusive access to the requested hardware.
When there are no hardware available, new session requests will
be queued.

### OOM for large models
When a function contains too many graph breaks,
the remote inference worker may run out of GPU memory.
When it happens, you may get an "Error on loading model component". 
This is known to happen with models like Stable Diffusion.
We are actively working on optimizing the memory allocation of 
many subgraphs.

### Limitations of TorchDynamo 
TorchDynamo is under active development. You may encounter
errors that are TorchDynamo related. For instance,
we found that TorchDynamo does not support `Ultralytics/Yolov5`.
These should not be fundamental problems as we believe TorchDynamo
will continue to improve its coverage.
If you find a broken model, please [file an issue](https://github.com/octoml/octoml-profile/issues).

## Contact the team
- Discord: [OctoML community Discord](https://discord.gg/Quc8hSxpMe)
- Github issues: https://github.com/octoml/octoml-profile/issues
- Email: dynamite@octoml.ai
