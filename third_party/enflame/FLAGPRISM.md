# Enflame debugger and profiler builds

`FLAGTREE_BACKEND=enflame` enables FlagPrism by default. It provides
`flagtree.debugger` and the TOPSPTI-backed `flagtree.profiler`. FlagPrism and
Proton cannot both be enabled.

## Dependencies and build

First prepare the TOPS SDK, matching LLVM toolchain and Python dependencies as
shown in [the Enflame backend guide](README.md). The tools additionally require:

- TOPS headers (`tops/tops_runtime.h`) and `libtopsrt.so`.
- TOPSPTI development headers (`topspti_activity.h`) and the public
  `libtopspti.so` library, not the internal `libtopspti_rt.so`.
- FlagPrism containing the Enflame integration, merged in
  [FlagPrism #12](https://github.com/flagos-ai/FlagPrism/pull/12). The unified
  operator tests were merged in
  [FlagPrism #13](https://github.com/flagos-ai/FlagPrism/pull/13).

The default search locations are `/opt/tops/include`, `/opt/tops/lib`, and
`/opt/tops/extras/TOPSPTI/{include,lib64}`. A fresh build downloads FlagPrism
from its official repository if `third_party/FlagPrism` is absent. An existing
checkout is reused without updating it: update it to a revision containing the
above changes, or use an explicit checkout:

```bash
# Run from the FlagTree repository root after SDK/toolchain setup.
export FLAGTREE_BACKEND=enflame
export FLAGPRISM_SOURCE_DIR=/absolute/path/to/FlagPrism
export TRITON_BUILD_FLAGPRISM=ON
export TRITON_BUILD_PROTON=OFF
python3 -m pip install -e . --no-build-isolation --no-deps
```

For a nonstandard SDK installation, pass the CMake cache paths using
`TRITON_APPEND_CMAKE_ARGS`. Both components need the TOPS paths:

```bash
export TRITON_APPEND_CMAKE_ARGS="-DTOPS_INCLUDE_DIR=/sdk/tops/include -DTOPS_LIBRARY=/sdk/tops/lib/libtopsrt.so -DFLAGTREE_TOPS_INCLUDE_DIR=/sdk/tops/include -DFLAGTREE_TOPS_RUNTIME=/sdk/tops/lib/libtopsrt.so -DTOPSPTI_INCLUDE_DIR=/sdk/tops/extras/TOPSPTI/include -DTOPSPTI_ACTIVITY_LIBRARY=/sdk/tops/extras/TOPSPTI/lib64/libtopspti.so"
```

Keep the SDK runtime libraries discoverable by the system loader, following the
SDK setup instructions. If switching between SDKs or build configurations,
use a fresh build directory to avoid stale CMake dependency paths.

To build only Triton without the debugger/profiler or their TOPSPTI dependency:

```bash
FLAGTREE_BACKEND=enflame TRITON_BUILD_FLAGPRISM=OFF \
  python3 -m pip install -e . --no-build-isolation --no-deps
```

This does not remove Triton's own TOPS SDK/toolchain requirements.

## Validation

With the Enflame backend installed, run the host integration and compiler tests
from the FlagTree root. The compiler tests mock SDK compilation; they do not
claim device execution coverage. Run the two groups separately because their
pytest configurations both register `--device`. For the host build-policy
tests, disable unrelated dependency downloads and backend setup hooks; the
tests select synthetic backends themselves. Keep `FLAGPRISM_SOURCE_DIR` set
when using an external checkout.

```bash
USE_FLAGCX=OFF USE_ENFLAME=OFF python3 -m pytest python/test/unit/test_flagprism.py -q
python3 -m pytest third_party/enflame/python/test/unit/runtime/test_debugger_launcher.py \
  third_party/enflame/python/test/unit/runtime/test_debugger_compiler.py -q
```

For device acceptance, run `python3 test.py` from the selected FlagPrism checkout.
See [FlagPrism's Enflame guide](https://github.com/flagos-ai/FlagPrism/blob/main/docs/enflame.md)
for device setup and debugger limitations. Integration has been validated on
GCU300; GCU400/410/500 require their own validation.

GCU300 has a narrowly scoped SDK workaround for a missing `fabs(float)` symbol
in debugger-generated code. Only the known linker diagnostic triggers a retry
with a compatibility library; unrelated compiler errors propagate normally.
The compiler regression tests cover this boundary, external-library retention,
and i64 code generation for local L2 captures under a global L1 configuration.
