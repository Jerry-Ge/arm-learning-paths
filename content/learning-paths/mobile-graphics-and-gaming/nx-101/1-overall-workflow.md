---
title: Overall workflow
weight: 2

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Overall workflow

- Construct the simple matmul neural network graph with TOSA.
- Convert the TOSA graph to vgf format using Arm ai-ml-sdk-model-converter.
- Prepare the scenario json file.
- Leverage the Arm ai-ml-sdk-scenario-runner to submit the scenario to Vulkan Driver for execution on NX.

## What is Arm Mali GPU NX?
Mali G2-Ultra NX integrates dedicated neural accelerators (NX) into the GPU and shares its memory system, caches, and control structures with the shader cores. This lets neural workloads use graphics data within the same GPU pipeline, reducing data movement and external memory traffic. For developers, it makes techniques such as neural upscaling, frame generation, and denoising more practical within a mobile device’s bandwidth and power limits.

For more details, you can check out this page: [Link](https://newsroom.arm.com/blog/arm-mali-g2-ultra-nx-ai-native-mobile-graphics)
