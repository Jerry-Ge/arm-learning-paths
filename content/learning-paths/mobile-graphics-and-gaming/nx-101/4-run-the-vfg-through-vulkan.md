---
title: Run the VGF through VulkanML
weight: 5

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## What is Scenario Runner C++ SDK?

For most of the non-graphics programmers, they don't want to write every Vulkan setup step directly. It can be quite tedious and daunting. Arm provided the Scenario Runner C++ Library to hide a lot of Vulkan complexity. In this demo, we will use it.

## Step by Step walkthrough

You can find the full C++ program for the demo in the very end of the page. In this part, we will go throught each code block chunk by chunk to understand the details.

TODO: for each step, understand it by myself then add notes.
### Create Context, DataManager, and Compute
```cpp {{ line_numbers = "true" line_start = "23" }}
        ScenarioOptions options; // Default Scenario Runner runtime settings.
        Context ctx(options, FamilyQueue::Compute); // Vulkan instance/device and compute queue family.
        DataManager data; // Owns the named tensors and the VGF view.
        Compute graph(ctx); // Records and submits graph commands on ctx's device.
```

### Tensor Construction
```cpp {{ line_numbers = "true" line_start = "28" }}
        // Create A, B, and the result. The VGF expects these three shapes/types.
        const char *names[] = {"a", "b", "result"};
        for (int i = 0; i < 3; ++i) {
            TensorInfo info; // Description used to create a Vulkan tensor.
            info.debugName = names[i]; // Human-readable name for diagnostics.
            info.shape = {1, 2, 2}; // Batch 1, two rows, two columns.
            info.format = i == 2 ? vk::Format::eR32Sint : vk::Format::eR8Sint;
            // i=0,1: signed int8 inputs; i=2: signed int32 result.
            data.createTensor(Guid(names[i]), info); // Register by Scenario Runner ID.
            auto &tensor = data.getTensorMut(Guid(names[i])); // Reference to that tensor.
            tensor.setup(ctx); // Create the VkTensorARM handle and get memory requirements.
            tensor.allocateMemory(ctx); // Allocate/bind device memory and create its tensor view.
            if (i < 2) { // Only A and B have initial data; the graph writes result.
                TensorDesc input(Guid(names[i]), names[i], info.shape, ShaderAccessType::ReadOnly);
                input.src = argv[i + 2]; // i=0 -> argv[2] (A); i=1 -> argv[3] (B).
                tensor.fillFromDescription(ctx, input); // Parse .npy and upload to the tensor.
            }
        }
```

### VGF View Construction
```cpp {{ line_numbers = "true" line_start = "47" }}
        data.createVgfView(Guid("matmul"), argv[1]); // Read the compiled VGF file.
        const auto &vgf = data.getVgfView(Guid("matmul")); // Reference, not a copy.
```

### Bindings
```cpp {{ line_numbers = "true" line_start = "53" }}
        // Map the VGF descriptor set 0, slots 0/1/2, to our named tensors.
        // std::nullopt means no optional level-of-detail value is supplied.
        const std::vector<TypedBinding> tensors = {
            {0, 0, Guid("a"), std::nullopt, vk::DescriptorType::eTensorARM},      // Input A.
            {0, 1, Guid("b"), std::nullopt, vk::DescriptorType::eTensorARM},      // Input B.
            {0, 2, Guid("result"), std::nullopt, vk::DescriptorType::eTensorARM}, // Output.
        };
        auto bindings = vgf.resolveBindings(0, data, tensors); // Validate and complete the mapping.
```

### Create Pipeline and then Submit
```cpp {{ line_numbers = "true" line_start = "61" }}
        // Build the Vulkan data-graph pipeline/session from VGF segment 0.
        // false disables optional neural statistics; {} uses its default mode.
        graph.createPipeline({"matmul", bindings, nullptr}, 0, vgf, data, false, {});
        // Record one graph dispatch. true asks the library for implicit barriers.
        graph.registerPipelineFenced(data, bindings, nullptr, 0, true);
        graph.submitAndWaitOnFence(); // Submit to Vulkan and wait for completion.
```

### Get the result
```cpp {{ line_numbers = "true" line_start = "68" }}
data.getTensor(Guid("result")).store(ctx, argv[4]); // Download and write .npy.
```

## Full Source Code
```c {{ line_numbers = "true" }}
// One int8 MatMul using the same Scenario Runner library objects as QuickSRNet.
// Lines beginning with // are comments: they explain the code and are ignored by C++.
#include "scenario.hpp" // Context, DataManager, Compute, and ScenarioOptions.

#include <iostream>  // std::cout and std::cerr.
#include <optional>  // std::nullopt in the tensor bindings.
#include <stdexcept> // std::runtime_error and std::exception.
#include <vector>    // std::vector of tensor bindings.

// Let us write Context instead of mlsdk::scenariorunner::Context below.
// This changes name lookup only; it does not create an object or run code.
using namespace mlsdk::scenariorunner;

// The OS calls main. argc counts all arguments, including the program name.
// argv[0] is the program; argv[1..4] are VGF, A.npy, B.npy, and output.npy.
int main(int argc, char **argv) {
    if (argc != 5) {
        std::cerr << "usage: magni_101 matmul.vgf input_a.npy input_b.npy output.npy\n";
        return 2; // Invalid command line.
    }

    try { // Report library or Vulkan errors in the catch block below.
        ScenarioOptions options; // Default Scenario Runner runtime settings.
        Context ctx(options, FamilyQueue::Compute); // Vulkan instance/device and compute queue family.
        DataManager data; // Owns the named tensors and the VGF view.
        Compute graph(ctx); // Records and submits graph commands on ctx's device.

        // Create A, B, and the result. The VGF expects these three shapes/types.
        const char *names[] = {"a", "b", "result"};
        for (int i = 0; i < 3; ++i) {
            TensorInfo info; // Description used to create a Vulkan tensor.
            info.debugName = names[i]; // Human-readable name for diagnostics.
            info.shape = {1, 2, 2}; // Batch 1, two rows, two columns.
            info.format = i == 2 ? vk::Format::eR32Sint : vk::Format::eR8Sint;
            // i=0,1: signed int8 inputs; i=2: signed int32 result.
            data.createTensor(Guid(names[i]), info); // Register by Scenario Runner ID.
            auto &tensor = data.getTensorMut(Guid(names[i])); // Reference to that tensor.
            tensor.setup(ctx); // Create the VkTensorARM handle and get memory requirements.
            tensor.allocateMemory(ctx); // Allocate/bind device memory and create its tensor view.
            if (i < 2) { // Only A and B have initial data; the graph writes result.
                TensorDesc input(Guid(names[i]), names[i], info.shape, ShaderAccessType::ReadOnly);
                input.src = argv[i + 2]; // i=0 -> argv[2] (A); i=1 -> argv[3] (B).
                tensor.fillFromDescription(ctx, input); // Parse .npy and upload to the tensor.
            }
        }

        data.createVgfView(Guid("matmul"), argv[1]); // Read the compiled VGF file.
        const auto &vgf = data.getVgfView(Guid("matmul")); // Reference, not a copy.
        // This demo supports exactly one native data-graph segment.
        if (vgf.getNumSegments() != 1 || vgf.getSegmentType(0) != ModuleType::GRAPH)
            throw std::runtime_error("expected one VGF graph segment");

        // Map the VGF descriptor set 0, slots 0/1/2, to our named tensors.
        // std::nullopt means no optional level-of-detail value is supplied.
        const std::vector<TypedBinding> tensors = {
            {0, 0, Guid("a"), std::nullopt, vk::DescriptorType::eTensorARM},      // Input A.
            {0, 1, Guid("b"), std::nullopt, vk::DescriptorType::eTensorARM},      // Input B.
            {0, 2, Guid("result"), std::nullopt, vk::DescriptorType::eTensorARM}, // Output.
        };
        auto bindings = vgf.resolveBindings(0, data, tensors); // Validate and complete the mapping.
        // Build the Vulkan data-graph pipeline/session from VGF segment 0.
        // false disables optional neural statistics; {} uses its default mode.
        graph.createPipeline({"matmul", bindings, nullptr}, 0, vgf, data, false, {});
        // Record one graph dispatch. true asks the library for implicit barriers.
        graph.registerPipelineFenced(data, bindings, nullptr, 0, true);
        graph.submitAndWaitOnFence(); // Submit to Vulkan and wait for completion.

        data.getTensor(Guid("result")).store(ctx, argv[4]); // Download and write .npy.
        std::cout << "Wrote " << argv[4] << '\n';
        return 0; // Success. run.sh also checks the four output values.
    } catch (const std::exception &error) {
        std::cerr << "magni_101: " << error.what() << '\n';
        return 1; // Setup, Vulkan, file, or execution failure.
    }
}

```