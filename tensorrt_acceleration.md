# PyTorch Model Acceleration

This will be an overiew on how to take a pre-trained model in PyTorch and accelerate it using TensorRT. Most of this material is from the official NVIDIA Developer guide. The official guide is quite exhaustive and therefore this serves as a tl;dr of PyTorch specific inference.

Original Documentation:

0. [Installation Guide](https://docs.nvidia.com/deeplearning/sdk/tensorrt-install-guide/index.html)
1. [Developer Guide](https://docs.nvidia.com/deeplearning/sdk/tensorrt-developer-guide/index.html)
2. [Samples](https://docs.nvidia.com/deeplearning/sdk/tensorrt-sample-support-guide/index.html)
3. [API](https://docs.nvidia.com/deeplearning/sdk/tensorrt-api/index.html)

### 1. TensorRT

![TensorRT](figs/tensorrt_ov.png)

##### What is TensorRT?
```
The core of TensorRT™ is a C++ library that facilitates high performance inference onNVIDIA graphics processing units (GPUs). It is designed to work in a complementaryfashion with training frameworks such as TensorFlow, Caffe, PyTorch, MXNet, etc. Itfocuses specifically on running an already trained network quickly and efficiently on aGPU for the purpose of generating a result (a process that is referred to in various placesas scoring, detecting, regression, or inference).
```

![Phases](figs/trt_phases.png)

##### Build Phase:
```
To optimize your model for inference, TensorRT takes your network definition,performs optimizations including platform specific optimizations, and generates theinference engine. This process is referred to as the build phase.
```

What's going on?

```
1. Elimination of layers whose outputs are not used
2. Fusion of convolution, bias and ReLU operations
3. Aggregation of operations with sufficiently similar parameters and the same source tensor
4. Merging of concatenation layers by directing layer outputs to the correct eventual destination
5. The builder also modifies the precision of weights if necessary
6. The build phase also runs layers on dummy data to select the fastest from its kernel catalog.
```

##### Execution Engine:

```
The Engine interface provides allow the application to executing inference. Itsupports synchronous and asynchronous execution, profiling, and enumeration andquerying of the bindings for the engine inputs and outputs.
```

#### Supported Platforms

```
C++ Implementation on all platforms and Python on x86
```

#### Important

1. Serialize: To serialize, you are transforming the engine into a format to store and use at a later timefor inference. To use for inference, you would simply deserialize the engine. Serializing and deserializing are optional. Since creating an engine from the Network Definition canbe time consuming, you could avoid rebuilding the engine every time the application reruns by serializing it once and deserializing it while inferencing. Therefore, after theengine is built, users typically want to serialize it for later use. 

### 2. Export a pre-trained PyTorch Model to ONNX

Install Protobuf compiler
```
sudo apt-get install protobuf-compiler libprotoc-dev
```

Install ONNX
```
pip3 install onnx
```

Check if your install is working
```
python3 -c "import onnx"
```

For PyTorch 1.0, there is official support for ONNX in the form of a
torch.onnx library. Check to see if your version of PyTorch contains this
library:
```
import torch.onnx
```

and, 

```
import torch.onnx
help(torch.onnx.export)
```

Exporting your PyTorch model to ONNX is then a matter of calling the
*torch.onnx.export* function:

```
dummy_input = torch.randn(sample_batch_size, channel, height, width)
torch.onnx.export(model, dummy_input, "onnx_model_name.onnx")
```

### 3. Loading an ONNX Model into TensorRT (C++)

```cpp
/* Function: Converting ONNX Model to TensorRT Model */
void onnxToTRTModel(const std::string& modelFile,
                    unsigned int maxBatchSize,
                    IHostMemory*& trtModelStream)
{
    int verbosity = (int) nvinfer1::ILogger::Severity::kWARNING;

    // Create the Builder
    IBuilder* builder = createInferBuilder(gLogger);
    nvinfer1::INetworkDefinition* network = builder->createNetwork();

    // Initialize the Parser (here for ONNX Network Definition)
    auto parser = nvonnxparser::createParser(*network, gLogger);
    if (!parser->parseFromFile(modelFile.c_str(), verbosity))
    {
        string msg("Failed to parse onnx file!");
        gLogger.log(nvinfer1::ILogger::Severity::kERROR, msg.c_str());
        exit(EXIT_FAILURE);
    }

    // Build the Engine
    builder->setMaxBatchSize(maxBatchSize);
    builder->setMaxWorkspaceSize(1 << 20);

    samplesCommon::enableDLA(builder, gUseDLACore);
    ICudaEngine* engine = builder->buildCudaEngine(*network);
    assert(engine);

    // Destroy Parser
    parser->destroy();

    // Serialize the Engine, destroy everything else
    trtModelStream = engine->serialize();
    engine->destroy();
    network->destroy();
    builder->destroy();
}
```
