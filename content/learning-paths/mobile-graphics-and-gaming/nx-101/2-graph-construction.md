---
title: Graph Construction
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Construct the simple neural network using TOSA
The Mali G2 NX is a TOSA 1.0 compatible AI accelerator with INT profile (TOSA-1.0+INT).

For simplicity, we can simply construct neural network graph using textual TOSA MLIR.

For more complicated ML contents, it's recommended to use ML Frameworks like Pytorch, Tensorflow to construct the graph then legalize to TOSA.

```text { line_numbers = "true" }
// A and B are 1-batch 2x2 signed int8 matrices; MatMul accumulates to int32.
module {
  func.func @main(%a: tensor<1x2x2xi8>, %b: tensor<1x2x2xi8>)-> tensor<1x2x2xi32> {
    %zero = "tosa.const"() {values = dense<0> : tensor<1xi8>} : () -> tensor<1xi8>
    %product = tosa.matmul %a, %b, %zero, %zero :
        (tensor<1x2x2xi8>, tensor<1x2x2xi8>, tensor<1xi8>, tensor<1xi8>) -> tensor<1x2x2xi32>
    return %product : tensor<1x2x2xi32>
  }
}
```

## Create a directory for your experiment
```
mkdir hello-nx
```

## Copy the TOSA MLIR text above into a file
```bash
cd hello-nx
touch matmul.mlir

# Copy the text into matmul.mlir
```

## Create the venv
```
python3 -m venv .venv
```

## Install the model-converter

What's model-converter?
The model converter is Arm’s host-side tool that turns a model expressed in TOSA into a VGF file that the Vulkan ML runtime can load. It runs during model preparation, not during inference on the phone. [Source](https://github.com/arm/ai-ml-sdk-model-converter)

```bash
.venv/bin/python -m pip install 'ai-ml-sdk-model-converter==0.10.0'
```

## Convert TOSA MLIR to TOSA flatbuffer using model-converter
```bash
.venv/bin/model-converter --tosa-flatbuffer --input matmul.mlir --output build/matmul.tosa
```

## Convert TOSA flatbuffer to vgf using model-converter
```bash
.venv/bin/model-converter --input build/matmul.tosa --output matmul.vgf --require-static-shape
```

Now you should have the `matmul.vgf` model file under your `hello-nx` directory

### What's VGF format?
VGF stands Vulkan Graph Format. The file packages what the Vulkan data graph runtime needs: the compiled graph module, tensor shapes and types, input/output bindings, and any constants. Scenario Runner reads the VGF, creates a graph pipeline from it, and binds your input and output tensors.

