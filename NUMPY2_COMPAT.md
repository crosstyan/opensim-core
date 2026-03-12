# Building OpenSim 4.5.2 with NumPy 2.x on Jetson Orin

Platform: JetPack 6.x (R36.4.4), Ubuntu 22.04, aarch64, CUDA 12.6.

## Problem

OpenSim 4.5.2 SWIG Python bindings fail at import time with NumPy 2.x:

```
RuntimeError: module compiled against ABI version 0x1000009 but this version of numpy is 0x2000000
ImportError: numpy.core.multiarray failed to import
```

Two issues compound:

1. **ABI mismatch.** System package `python3-numpy` installs NumPy 1.x C headers
   under `/usr/include/python3.10/numpy/`. These define `NPY_ABI_VERSION=0x01000009`.
   The upstream `CMakeLists.txt` lists `Python3_INCLUDE_DIRS` (which resolves to
   `/usr/include/python3.10`) before `Python3_NumPy_INCLUDE_DIRS`, so the compiler
   finds the stale system headers first and compiles against the wrong ABI.

2. **Module path rename.** NumPy 2.x moved `numpy.core._multiarray_umath` to
   `numpy._core._multiarray_umath`. The system headers hardcode the old path in
   `_import_array()`, so even if you bypass the ABI check, `import_array()` fails
   at runtime.

## Fix

One change in `Bindings/Python/CMakeLists.txt` (committed on branch `numpy2-compat`):

```diff
-    target_include_directories(${_libname} PRIVATE
-            "${Python3_INCLUDE_DIRS}"
-            "${Python3_NumPy_INCLUDE_DIRS}")
+    target_include_directories(${_libname} BEFORE PRIVATE
+            "${Python3_NumPy_INCLUDE_DIRS}"
+            "${Python3_INCLUDE_DIRS}")
```

`BEFORE` ensures these directories precede all others on the include path.
Listing `Python3_NumPy_INCLUDE_DIRS` first means `#include <numpy/arrayobject.h>`
resolves from the pip-installed NumPy 2.x headers (correct ABI, correct module path)
rather than the system NumPy 1.x headers.

No system header patching required.

## Reproducible Build

### Prerequisites

```bash
sudo apt install -y build-essential cmake ninja-build \
  liblapack-dev freeglut3-dev libxi-dev libxmu-dev \
  python3-dev python3-pip pkg-config autoconf automake \
  libtool bison byacc pcre2-utils libpcre2-dev

pip install numpy>=2.0
```

### 1. Build SWIG 4.1.1

```bash
cd ~/opensim-workspace
tar xf ~/ThirdParty/swig-4.1.1.tar.gz
cd swig-4.1.1
./configure --prefix=$HOME/swig
make -j8 && make install
```

### 2. Build OpenSim Dependencies

```bash
cd ~/opensim-workspace
mkdir opensim-core-dependencies-build && cd opensim-core-dependencies-build
cmake ~/ThirdParty/opensim-core/dependencies \
  -GNinja \
  -DCMAKE_INSTALL_PREFIX=~/opensim-workspace/opensim-core-dependencies-install \
  -DSUPERBUILD_ezc3d=on
ninja -j8
```

### 3. Build OpenSim

```bash
cd ~/opensim-workspace
mkdir opensim-core-build && cd opensim-core-build
cmake ~/ThirdParty/opensim-core \
  -GNinja \
  -DCMAKE_INSTALL_PREFIX=~/opensim-core \
  -DCMAKE_BUILD_TYPE=RelWithDebInfo \
  -DBUILD_PYTHON_WRAPPING=on \
  -DBUILD_JAVA_WRAPPING=on \
  -DOPENSIM_C3D_PARSER=ezc3d \
  -DOPENSIM_WITH_CASADI=on \
  -DSWIG_EXECUTABLE=~/swig/bin/swig \
  -DPython3_ROOT_DIR=/usr \
  -DOPENSIM_DEPENDENCIES_DIR=~/opensim-workspace/opensim-core-dependencies-install
ninja -j8
cmake --install .
```

### 4. Build Wheel

```bash
cd ~/opensim-core/sdk/Python
python3 setup.py bdist_wheel
pip install dist/opensim-4.5.2-cp310-cp310-linux_aarch64.whl
```

### 5. Runtime Setup

```bash
echo 'export LD_LIBRARY_PATH="$HOME/opensim-core/sdk/Simbody/lib:$HOME/opensim-core/sdk/lib:$HOME/opensim-core/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"' >> ~/.bashrc
source ~/.bashrc
```

### 6. Verify

```bash
python3 -c "
import numpy; print('NumPy', numpy.__version__)
import opensim; print('OpenSim', opensim.GetVersion())
m = opensim.Model(); m.setName('test')
print('Model:', m.getName())
"
```

Expected:

```
NumPy 2.2.6
OpenSim 4.5.2-...
Model: test
```
