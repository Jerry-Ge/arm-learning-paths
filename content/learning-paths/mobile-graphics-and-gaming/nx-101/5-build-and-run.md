---
title: Build the Vulkan Program then run on NX phone
weight: 6

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Build the program then run on the Android phone.

- You need a Android phone with NX hardware
    - Mediateck 9600 Pro
    - Xring O3
- Enable Developer Mode

### Fetch Dependencies
Run the following shell script to fetch dependencies.
```bash
#!/usr/bin/env bash
set -euo pipefail
cd "$(dirname "$0")"

# Pin the upstream C++ sources
fetch() {
  local name="$1" url="$2" revision="$3" path="third_party/$1"
  if [[ ! -d "$path/.git" ]]; then
    mkdir -p "$path"
    git -C "$path" init -q
    git -C "$path" remote add origin "$url"
  fi
  if [[ "$(git -C "$path" rev-parse HEAD 2>/dev/null || true)" != "$revision" ]]; then
    git -C "$path" fetch -q --depth 1 origin "$revision"
    git -C "$path" checkout -q --detach "$revision"
  fi
}

fetch Vulkan-Headers https://github.com/KhronosGroup/Vulkan-Headers.git 8864cdc896bbc2a9b6eb36b3218fc9ef57908d77
fetch flatbuffers https://github.com/google/flatbuffers.git 1c514626e83c20fffa8557e75641848e1e15cd5e
fetch vgf-lib https://github.com/arm/ai-ml-sdk-vgf-library.git b21091b9d81efeea582ae9e0c21a4684d7aa8954
fetch scenario-runner https://github.com/arm/ai-ml-sdk-scenario-runner.git e7481c62116f8663b8aafc83ee8fdd563e6253cb
fetch argparse https://github.com/p-ranav/argparse.git 3eda91b2e1ce7d569f84ba295507c4cd8fd96910
fetch json https://github.com/nlohmann/json.git 9cca280a4d0ccf0c08f47a99aa71d1b0e52f8d03
fetch SPIRV-Headers https://github.com/KhronosGroup/SPIRV-Headers.git 8c5559c134abcf432ec59db842404087b9906c1a
fetch SPIRV-Tools https://github.com/KhronosGroup/SPIRV-Tools.git 605a0154c7743ed4239c0eb7f3c61f17ffde4fd0
fetch glslang https://github.com/KhronosGroup/glslang.git 275822a6261ee689aadb1da5f09a0ec2f058685c

```

### Build the Android App
```bash
# Build only the example app, linked with ScenarioRunnerLib, for Android.
ANDROID_SDK_ROOT="${ANDROID_SDK_ROOT:-$HOME/Library/Android/sdk}"
if [[ -z "${ANDROID_NDK_HOME:-}" ]]; then
  ANDROID_NDK_HOME="$(find "$ANDROID_SDK_ROOT/ndk" -mindepth 1 -maxdepth 1 -type d | sort -V | tail -1)"
fi
[[ -f "$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake" ]] || {
  echo "Set ANDROID_NDK_HOME to an installed Android NDK" >&2; exit 1;
}
cmake -S . -B build/android -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-35 -DCMAKE_BUILD_TYPE=Release
cmake --build build/android --target magni_101 -j 8
cp build/android/magni_101 build/magni_101
"$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/darwin-x86_64/bin/llvm-strip" build/magni_101
echo "Built build/magni_101 and matmul.vgf"
```


### Connect to your phone then Submit the program for running
TODO: make this line by line command.
```bash
remote=/data/local/tmp/hello_nx
"$ADB" -s "$serial" shell mkdir -p "$remote"
"$ADB" -s "$serial" push build/hello_nx build/inputs/input_a.npy \
  build/inputs/input_b.npy matmul.vgf "$remote/" >/dev/null
"$ADB" -s "$serial" shell "test ! -e '$remote/output.npy' || rm '$remote/output.npy'"
"$ADB" -s "$serial" shell "cd '$remote' && chmod 755 hello_nx && ./hello_nx matmul.vgf input_a.npy input_b.npy output.npy"
"$ADB" -s "$serial" pull "$remote/output.npy" build/inputs/output.npy >/dev/null
```



