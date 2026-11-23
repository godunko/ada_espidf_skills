---
name: ada-espidf-skeleton
description: "Create starter ESP-IDF firmware projects with Ada integration for supported ESP32 chips. Use when asked to scaffold or initialize a combined Ada + ESP-IDF application, not a standalone Ada project."
---

# ESP-IDF + Ada Skeleton Scaffolding

## Inputs and Destination

Determine the destination, project name, and chip from the request and workspace.
"In this directory" means use that directory directly; do not create another
project folder inside it. Infer the chip from an unambiguous directory name such
as `esp32s3`; otherwise default to `esp32s3`. If no destination is specified, use
`espidf-ada-skeleton`.

Use an Alire-compatible project name (lowercase letters, digits, underscores,
starting with a letter), and use it consistently in `alire.toml`, the GPR filename,
the generated config import, and CMake. Keep the GPR project identifier valid Ada.
Use the user's known maintainer name and email; do not invent an identity.

Inspect existing files and repository instructions before writing. Preserve
unrelated work; only ask before replacing files when that replacement has not
already been authorized. Keep generated files ASCII-only unless requested otherwise.

## Prerequisites and Reference Selection

1. Check `alr --version`, Git, a host C compiler, and Make. Locate ESP-IDF and its
   installed target tools. If `idf.py` is unavailable, locate the existing
   `export.sh` and source it in the same shell that runs the build. Do not assume
   that an empty `IDF_PATH` means ESP-IDF is not installed. If a required tool is
   missing, report it; install it only within the user's authorized scope.
2. Inspect the matching upstream template and dependency manifests before selecting
   versions:
   - ESP32: https://github.com/RREE/esp32_template
   - ESP32-S3: https://github.com/godunko/esp32s3_template
   - ESP32-C3: https://github.com/godunko/esp32c3_template
   Prefer a consistent upstream compiler/runtime combination over stale examples
   here. The ESP32-S3 combination verified with ESP-IDF 6.1 is GNAT 16 with
   `bb-runtimes` branch `gnat-fsf-16`.
   Use an `a0b-tools` revision with `System.Parameters` generation (commit
   `6e92124` or later) for the current embedded descriptor, which no longer
   supplies that package. Rebuild the generator after changing its checkout.
3. Select the runtime backend for the requested chip from the current matching
   template, runtime README, descriptors and component sources. Confirm that
   `svd/<chip>.svd` and the selected descriptor exist. For the new embedded
   integration use `runtime-embedded-<chip>.json` when supplied by the port.
   ESP32-S3 currently provides this descriptor; inspect the current checkout for
   ESP32, ESP32-C3 and other ports as they evolve. Keep the workflow applicable
   to Xtensa and RISC-V; do not restrict it permanently to ESP32-S3.
   A legacy `runtime-<chip>.json` is not evidence that the new embedded C bridge
   is compatible: it may compile its own C bridge through GPR. Do not combine
   those bridges. If the requested port has not migrated, use its matching
   validated template/integration or report the concrete missing port inputs.
   Compiler architecture support alone does not establish runtime support.

## Project Layout

```text
my_project/
|-- CMakeLists.txt
|-- components/
|   `-- espidf_gnat_runtime/     # Same checkout used by ESP-IDF and Alire
|-- crates/                     # Host tools and bb-runtimes sources
|-- main/
|   |-- CMakeLists.txt
|   `-- .gitignore
|-- source/main.adb
|-- my_project.gpr
|-- alire.toml
|-- sdkconfig.defaults
|-- README.md
`-- .gitignore
```

## Setup and Generation

Use the available terminal tool; no particular tool name or asynchronous API is
required. Carry out setup directly rather than leaving the initial checkouts and
tool builds to the user. Observe completion and exit status before dependent steps.

1. Clone the following repositories into the listed directories. Reuse valid
   existing clones without resetting or replacing local changes.

   | Repository | Destination | Version selection |
   | --- | --- | --- |
   | https://github.com/godunko/a0b-tools | `crates/a0b-tools` | Compatible with the reference template |
   | https://github.com/godunko/xtensa-dynconfig | `crates/xtensa-dynconfig` | Xtensa only |
   | https://github.com/godunko/espidf_gnat_runtime | `components/espidf_gnat_runtime` | Compatible with the reference template |
   | https://github.com/alire-project/bb-runtimes.git | `crates/bb-runtimes` | `gnat-fsf-16` for GNAT 16 |

   ESP-IDF discovers `components/espidf_gnat_runtime` automatically. Pin this
   same checkout in Alire; do not install a second runtime through the IDF
   Component Manager. Use public HTTPS URLs for distributable submodules.

   Match the `bb-runtimes` branch to the chosen GNAT major version. This is a
   runtime source checkout consumed by `a0b-runtime`, not an Alire crate to add to
   `[[depends-on]]` or `[[pins]]`.
2. Build the tools, checking each result:

   ```sh
   alr -n -C crates/a0b-tools build
   alr -n -C crates/xtensa-dynconfig build
   ```

   Omit the second command for RISC-V. Its pre-build actions fetch Espressif's
   `xtensa-dynconfig` and `xtensa-overlays` repositories and build the plugins.
   Alire also writes temporary files and shared caches outside the project; use
   the environment's permission workflow if sandbox restrictions block setup.
3. Generate the files below, adapting the matching upstream template to the
   project name and chip. Pin the local Alire crates in `alire.toml`. For Xtensa,
   also set `XTENSA_GNU_CONFIG` to the built chip-specific plugin; adding the
   dependency alone is insufficient.
4. Generate `.gitignore` entries for `.objs/`, `.libs/`, `runtime/`, `config/`,
   `build/`, generated Alire state, `sdkconfig`, and `sdkconfig.old`. The new
   helper keeps ExternalProject bookkeeping under the CMake build directory.
   Explicitly choose how dependency checkouts are preserved: Git submodules or
   ignored local clones with recreation instructions. Do not silently vendor
   nested repositories. For reproducible setup, record the checked-out commit
   SHAs (including the nested Xtensa sources) and reference-template revision in
   the README, with commands to restore them. Branch names alone can change.
   When preserving an Alire lockfile, exempt `alire/alire.lock` from ignored
   generated state as needed; Git revisions alone do not lock index dependencies.
5. Write a concise README with prerequisites, environment activation, dependency
   recreation and tool builds, `idf.py build`, optional flash/monitor commands,
   and expected application output. Use the actual detected ESP-IDF path as a
   local example and identify machine-specific paths as such.

## Verification

Run `idf.py build` from the project root with the ESP-IDF environment active.
Do not verify the application by invoking `alr build` directly at the root:
ESP-IDF must build the SDK-facing C bridge, integrate the unwinder and generate
`components/espidf_gnat_runtime/.gens/s-oscons.ads` from the active SDK headers
and effective `sdkconfig` before the Alire pre-build action reads the descriptor.
The runtime's `espidf_add_ada_application` helper supplies this dependency. Do
not invoke `a0b-runtime` early or create `System.OS_Constants` by hand. If a
standalone Alire build is needed, first run `idf.py gnat_system_os_constants`;
this does not replace the complete firmware build.

When upgrading an existing scaffold to a changed descriptor or generator, clean
its generated runtime with `idf.py clean` before building. Packages can move
between `libgnat` and `libgnarl`; stale generated copies cause duplicate-unit
errors. Preserve application sources and dependency checkouts.

The output is a partially linked Ada object (`.objs/app_main.o`) linked into the ESP-IDF ELF/BIN.

`sdkconfig.defaults` selects the target for a fresh configuration. If an existing
`sdkconfig` selects another target, use `idf.py set-target <chip>` only when
reconfiguration is within the authorized scope; that command resets configuration.
In VS Code, prefer available ESP-IDF extension build/target commands.

If the build fails, diagnose and fix the scaffold or report the concrete missing
prerequisite. Do not claim a successful build from configuration or tool builds
alone. Report the resulting firmware paths and distinguish build verification
from execution on hardware. Flash or monitor only when requested.

Check an incremental rebuild, an Ada source change, rebuilding after the
relocatable object is removed, and a clean/build cycle. For a QEMU smoke test,
check expected application output as well as the emulator exit status.

## Skeleton Templates (New Embedded Integration)

Use these examples with a matching embedded port and component. For a port
still using the legacy integration, adapt its matching validated template
instead. The embedded runtime initializes tasking itself through `System.Init`;
do not carry over the legacy `System.Tasking.Initialization` application import.

### ESP-IDF Root `CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.22)

include($ENV{IDF_PATH}/tools/cmake/project.cmake)

# Minimal build: include only main and dependencies
idf_build_set_property(MINIMAL_BUILD ON)

project(<name>)

# Remove generated Ada artifacts when cleaning
set_property(TARGET "${PROJECT_NAME}.elf" APPEND PROPERTY
    ADDITIONAL_CLEAN_FILES "${CMAKE_CURRENT_SOURCE_DIR}/.objs"
                           "${CMAKE_CURRENT_SOURCE_DIR}/.libs"
                           "${CMAKE_CURRENT_SOURCE_DIR}/runtime")
```

### ESP-IDF `main/CMakeLists.txt`
```cmake
idf_component_register(REQUIRES espidf_gnat_runtime)

idf_component_get_property(runtime_dir espidf_gnat_runtime COMPONENT_DIR)
include("${runtime_dir}/cmake/ada-application.cmake")

espidf_add_ada_application(
    TARGET "${COMPONENT_LIB}"
    PROJECT_DIR "${PROJECT_DIR}"
    OBJECT ".objs/app_main.o")
```

### Ada Project File `<name>.gpr`

Within this GPR template, select `Target` by architecture after confirming runtime
support for the chip:
- Xtensa: `"xtensa-esp32-elf"`
- RISC-V: `"riscv64-elf"`

The `runtime/build_*.gpr` files and `System.Parameters` are generated by the
Alire pre-build action; they are not missing source files to create by hand.

```gpr
with "runtime/build_libgnat.gpr";   --  Base GNAT runtime
with "runtime/build_libgnarl.gpr";  --  Tasking GNAT runtime
--  These runtime project files are generated by a0b-tools

with "config/<name>_config.gpr";    --  Generated by Alire; <name> must match alire.toml name field

project <Name> is

   for Target use "<target-triple>";  --  "xtensa-esp32-elf" or "riscv64-elf"
   for Runtime ("Ada") use "runtime";

   for Source_Dirs use ("source");
   for Object_Dir use ".objs";
   for Main use ("main.adb");

   package Builder is
      --  Compiles to object file, not executable (linked by CMake)
      for Executable ("main.adb") use "app_main.o";
   end Builder;

   package Binder is
      for Switches ("Ada") use
        ("-D4k",
         --  Secondary stack size = nn [kilo|mega] bytes.
         --
         --  Secondary stacks are statically allocated, so their size is
         --  fixed by this switch and cannot grow at run time.
         "-Q<stack-count>",
         --  Generate nnn secondary stacks.
         --
         --  Reserve the minimum required by the selected backend for the
         --  environment task and per-CPU interrupt handlers, plus one stack
         --  per foreign thread needing a secondary stack. Ordinary Ada tasks
         --  are counted separately by the binder.
         "-Mapp_main");
         --  Set the binder generated main function's name to app_main as
         --  this is required by the SDK.
   end Binder;

   package Compiler is
      for Switches ("Ada") use ("-g", "-O2");
   end Compiler;

   package Linker is
      for Default_Switches ("Ada") use ("-Wl,-r", "-nostdlib");
   end Linker;

end <Name>;
```

For the current dual-core ESP32-S3 embedded backend, `-Q3` is the minimum:
one secondary stack for the environment task and one for interrupt handlers on
each CPU.
Add one entry for each foreign thread needing a secondary stack over the whole
application lifetime; these entries are not recycled. Check each other port's
CPU/context accounting rather than copying the ESP32-S3 count blindly.

### Ada `alire.toml`

Select the compiler version with the matching upstream template. For GNAT 16:
- RISC-V: use `gnat_riscv64_elf = "^16"`; omit the Xtensa environment block,
  dependency, and pin.
- Xtensa: use `gnat_xtensa_esp32_elf = "^16"`, add and pin `xtensa_dynconfig`,
  and retain the environment block below, replacing `<chip>` with the supported
  chip name (for example `esp32s3`).

Replace all placeholders, including maintainer details and the selected
runtime descriptor (for example `runtime-embedded-esp32s3.json`), before writing
the file.

```toml
name = "<name>"
description = "ESP-IDF + Ada Integrated Skeleton"
version = "0.1.0-dev"

authors = ["Your Name"]
maintainers = ["Your Name <email@example.com>"]
licenses = "Apache-2.0 WITH LLVM-exception"
tags = ["embedded", "<chip-tag>"]

[configuration]
generate_ada = false
generate_c = false
generate_gpr = true

# Xtensa only; omit this block for RISC-V.
[environment]
XTENSA_GNU_CONFIG.set = "${CRATE_ROOT}/crates/xtensa-dynconfig/xtensa-dynconfig/xtensa_<chip>.so"

[[depends-on]]
a0b_base = "*"
a0b_tools = "*"
espidf_gnat_runtime = "*"
<gnat-toolchain-dependency>  # gnat_riscv64_elf = "^16" or gnat_xtensa_esp32_elf = "^16"
<xtensa-dynconfig-dependency>  # Xtensa only: xtensa_dynconfig = "*"

[[actions]]
type = "pre-build"
command = ["a0b-runtime",
          "--bb-runtimes=crates/bb-runtimes/",
          "--svd=components/espidf_gnat_runtime/svd/<chip>.svd",
          "--runtime-description=components/espidf_gnat_runtime/<runtime-descriptor>",
          "--no-startup"]

[[pins]]
a0b_tools = { path = "crates/a0b-tools" }
<xtensa-dynconfig-pin>  # Xtensa only: xtensa_dynconfig = { path = "crates/xtensa-dynconfig" }
espidf_gnat_runtime = { path = "components/espidf_gnat_runtime" }
```

### Ada `source/main.adb`
```ada
with Ada.Text_IO; use Ada.Text_IO;

procedure Main is
begin
   Put_Line ("Hello, Ada world! Skeleton application.");
end Main;
```

### ESP-IDF `sdkconfig.defaults`

```text
CONFIG_COMPILER_CXX_EXCEPTIONS=y
CONFIG_IDF_TARGET="<chip>"
CONFIG_LIBC_NEWLIB=y
```

Newlib is required by this runtime integration. Keep the matching template's C++
exception setting when using its exception support configuration.

For each chip/backend, inspect its required configuration and ABI generation.
For the current ESP32-S3 embedded port with ESP-IDF 6.1, also use:

```text
CONFIG_FREERTOS_THREAD_LOCAL_STORAGE_POINTERS=2
```

Two TLS slots suffice for the default `CONFIG_GNAT_RUNTIME_ATCB_TLS_SLOT=1`.
Slot 0 belongs to pthread. If another component uses slot 1, select an unused
Ada slot in Component config > GNAT Runtime and make the total FreeRTOS TLS
pointer count greater than that index. The current runtime generates storage
sizes, alignments, tick rate and the selected ATCB slot in `System.OS_Constants`
from the actual SDK configuration; do not impose the old fixed structure sizes,
exactly-two-slot restriction or fixed 100 Hz ABI requirement. Keep the dual-core
configuration used by the ESP32-S3 template unless a changed CPU configuration
has been separately validated. Do not explicitly set SDK defaults.
Keep all nonessential configuration at SDK defaults, including primary stack sizes, watchdog and flash
settings unless the user's board/application requires an override. Do not copy
enlarged stack sizes or other test-harness configuration into the scaffold, or
ESP32-S3 ABI constants to other SoCs. Editing defaults does not override values
in an existing sdkconfig.

Do not enable `-fstack-check` by default. If requested, increase each primary
stack executing instrumented code for its application workload plus 12 KiB
(12288 bytes) for the compiler's checking allowance. Apply this to
the main task, other Ada/native tasks and per-CPU ISR stacks as relevant. These
primary-stack settings are separate from the binder secondary-stack pool.
