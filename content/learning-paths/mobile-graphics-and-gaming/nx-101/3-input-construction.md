---
title: Inputs Construction
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

### Construct the dummy inputs

Use the following Python code to construct some dummy inputs

```python
"""Write the two signed int8 matrices used by the C++ MatMul example."""
from pathlib import Path

import numpy as np

out = Path(__file__).resolve().parent / "build" / "inputs"
out.mkdir(parents=True, exist_ok=True)
np.save(out / "input_a.npy", np.array([[[1, 2], [-3, 4]]], dtype=np.int8))
np.save(out / "input_b.npy", np.array([[[5, -7], [8, 9]]], dtype=np.int8))
```