# Roadmap: Migration to `pybind11` / `nanobind` & C++/Python Boundary Caveats

Engineering roadmap for expanding this tutorial from a pure `Boost.Python` reference into a side-by-side modern C++/Python binding guide (`Boost.Python` -> `pybind11` v2/v3, with `nanobind` notes and production boundary caveats), kept in sync across both English (`README.md`) and Polish (`README.pl.md`) editions.

---

## 1. Side-by-Side Migration Roadmap (`Boost.Python` -> `pybind11` & `nanobind`)

### 1.1 Chapter-by-Chapter API Mapping

| Tutorial Section | `Boost.Python` | `pybind11` (v2 / v3) | Migration & Expansion Notes |
| --- | --- | --- | --- |
| **Module entry point** | `BOOST_PYTHON_MODULE(example) { ... }` | `PYBIND11_MODULE(example, m) { ... }` | Explicit `py::module_& m` handle replaces implicit global scope; header-only `<pybind11/pybind11.h>` removes the compiled `Boost::python` binary dependency. |
| **Functions & docstrings** | `def("square", square, "doc")` | `m.def("square", &square, "doc")` | Supports C++11+ lambdas, `std::function`, and overloaded functions via `py::overload_cast<int>(&square)`. |
| **Keywords & defaults** | `(arg("x"), "y", arg("z")=0.0)` | `py::arg("x"), py::arg("y"), py::arg("z") = 0.0` | Add coverage of `py::kw_only()`, `py::pos_only()`, `.none(false)` (disallowing `None`), and `.noconvert()` (disabling implicit C++ conversions). |
| **Template functions** | `def("squareFloat", square<double>)` | `m.def("square", &square<double>)` | Unlike `Boost.Python` examples that needed distinct names or manual wrappers, show true Python-side overload resolution on a single symbol `square` alongside explicit names. |
| **Classes** | `class_<World>("World")` | `py::class_<World>(m, "World")` | In `pybind11` v3, `py::classh<World>(m, "World")` enables `py::smart_holder` out of the box. |
| **Constructors** | `init<std::string>(args("msg"), "doc")` | `.def(py::init<const std::string&>(), py::arg("msg"), "doc")` | `no_init` becomes simply omitting `.def(py::init<...>())`. `boost::noncopyable` is no longer needed for abstract classes because `pybind11` does not eagerly instantiate copy converters. Also cover factory constructors via `py::init(&make_world)`. |
| **Methods & static methods** | `.def("create", &A::create)` + `.staticmethod("create")` | `.def_static("create", &A::create)` | Free functions taking `const Cls&` or `Cls&` as their first parameter still bind directly as methods via `.def("invite", &invite)`. |
| **Data members & properties** | `def_readonly` / `def_readwrite`, `add_property` / `add_static_property` | `def_readonly` / `def_readwrite`, `def_readonly_static` / `def_readwrite_static`, `def_property` / `def_property_readonly` / `def_property_static` | Static fields and properties use explicit `*_static` methods instead of overloaded `def_readonly` on global references. |
| **Inheritance** | `class_<Derived, bases<Base>>("Derived")` | `py::class_<Derived, Base>(m, "Derived")` | For C++ multiple inheritance, pass `py::multiple_inheritance()` tag to `py::class_`. Automatic downcasting works for polymorphic types (`virtual ~Base()`). |
| **Virtual functions** | `struct BaseWrap : Base, wrapper<Base>` + `get_override` + `pure_virtual` + `&BaseWrap::default_f` | Trampoline `struct PyBase : Base { using Base::Base; ... }` with `PYBIND11_OVERRIDE` / `PYBIND11_OVERRIDE_PURE` | Eliminates the error-prone two-pointer `.def("f", &Base::f, &BaseWrap::default_f)` pattern: `py::class_<Base, PyBase>(m, "Base").def("f", &Base::f)` handles both pure and non-pure virtual dispatch cleanly. |
| **Call policies** | `return_internal_reference<>`, `return_value_policy<...>` | `py::return_value_policy::reference_internal`, `reference`, `take_ownership`, `copy`, `move` | Map every `Boost.Python` policy 1:1 and highlight `automatic` / `automatic_reference` defaults. |
| **Lifetime ties** | `with_custodian_and_ward<M, N>` / `_postcall<M, N>` | `py::keep_alive<Nurse, Patient>()` | Unified indexing in `pybind11`: `0` is always the return value, `1` is `self` (or arg 1 for free functions), `2..N` are subsequent arguments. |
| **Smart pointers** | `class_<A, boost::shared_ptr<A>>` | `py::class_<A>` (`std::unique_ptr<A>` default in v2), `py::class_<A, std::shared_ptr<A>>`, or v3 `py::classh<A>` | Returning `std::unique_ptr<T>` from factories works automatically without `manage_new_object`. |
| **Operators & specials** | `self + int()`, `float_(self)`, `str(self)` | `<pybind11/operators.h>`: `py::self + int()`, plus `.def("__repr__", ...)` / `.def("__str__", ...)` | Prefer explicit lambdas for `__repr__`, `__str__`, `__hash__`, and context managers (`__enter__` / `__exit__`). |
| **Tuples, lists, dicts & STL** | `boost::python::tuple`, `list`, `dict` | `py::tuple`, `py::list`, `py::dict` + `<pybind11/stl.h>` | Contrast thin Python object wrappers (`py::list` — zero-copy handle to `PyObject*`) with `<pybind11/stl.h>` automatic C++ container casters. |

### 1.2 `nanobind` Comparison Notes (Dedicated Subchapter)

Add a decision matrix comparing `Boost.Python`, `pybind11`, and `nanobind` (by Wenzel Jakob, original author of `pybind11`):

- **Requirements:** `nanobind` targets C++17+ and Python 3.8+, shedding legacy C++11/Python 2/early-3 compatibility layers.
- **Compile time & binary size:** ~3-5x faster compilation and ~2-4x smaller `.so`/`.pyd` binaries than `pybind11` due to compact type tables and avoiding heavy template metaprogramming per overload.
- **Runtime call overhead:** Lower per-call dispatch latency and compact instance layout (co-locating small C++ objects with the Python object header when possible).
- **Stable ABI (`abi3`):** First-class support for Python 3.12+ Limited API (`Stable ABI`), allowing a single wheel per OS/arch across minor Python versions (`cp312-abi3-*`).
- **Tensors (`nb::ndarray`):** Built-in zero-copy multi-dimensional array exchanges via DLPack and the Python buffer protocol across NumPy, PyTorch, JAX, and CuPy (CPU & GPU) without linking against NumPy C headers.
- **Stricter defaults:** `nanobind` rejects silent dangling-pointer patterns and leaks loudly at interpreter shutdown if bound C++/Python instances remain alive (configurable via `nb::set_leak_warnings`).

### 1.3 Modern Build System & Packaging Replacement

Replace the legacy `bjam`, `SCons`, and CMake 3.5 `find_package(PythonLibs)` + directory-scoped `INCLUDE_DIRECTORIES` section with a modern, reproducible toolchain:

- **CMake 3.25+ target-based workflow:**
  ```cmake
  cmake_minimum_required(VERSION 3.25)
  project(example_bindings LANGUAGES CXX)

  find_package(Python REQUIRED COMPONENTS Interpreter Development.Module)
  find_package(pybind11 CONFIG REQUIRED)

  pybind11_add_module(example MODULE src/bindings.cpp)
  target_compile_features(example PRIVATE cxx_std_20)
  ```
  - Explain why `Development.Module` (instead of full `Development` or deprecated `PythonLibs`) is critical for `manylinux` / Conda / macOS framework builds where `libpython` must **not** be linked into extension modules.
  - Note `Development.SABIModule` for `nanobind` / Python 3.12+ Stable ABI targets.
- **`CMakePresets.json`:** Standardized configure/build/test presets (`dev-debug`, `dev-release`, `asan-ubsan`) with Ninja generator and `CMAKE_EXPORT_COMPILE_COMMANDS=ON`.
- **`pyproject.toml` (`scikit-build-core`):**
  - PEP 517/518/660 compliant packaging (`build-backend = "scikit_build_core.build"`), editable installs (`pip install --no-build-isolation -ve .`), wheel building via `cibuildwheel`, and `.pyi` type stub generation (`pybind11-stubgen` / `nanobind.stubgen`) validated with `mypy` / `pyright`.

---

## 2. Key C++/Python Boundary Caveats (Checklist for New Chapters)

Each item below represents a recurring production bug class at the C++/Python boundary and should be illustrated with a minimal failing snippet + fixed version.

### 2.1 Object Lifetime & Ownership Pitfalls

- [ ] **`reference_internal` & container reallocation UB:** Returning `const Element&` from `Container::at(i)` with `py::return_value_policy::reference_internal` keeps the `Container` alive while the Python `Element` wrapper lives, **but does not prevent `std::vector::push_back` / `clear` from invalidating the underlying C++ pointer** -> instant use-after-free when Python reads the element after mutating the vector.
- [ ] **Chained internal references (`a.b().c()`):** Verify lifetime propagation across multi-hop getters (`a` -> temporary wrapper `b` -> `c`): `reference_internal` ties `c` to `b` and `b` to `a`, whereas accidentally using raw `reference` on `b()` lets `a` die at the semicolon.
- [ ] **Dangling pointers/references across language boundaries:** Passing `T*` or `const T&` stored inside a C++ manager/singleton without `keep_alive` or `shared_ptr` ownership; C++ stack references escaping into stored Python callbacks.
- [ ] **`py::keep_alive<Nurse, Patient>()` traps vs `with_custodian_and_ward`:**
  - Off-by-one index confusion (`0` = return value, `1` = `self`, `2` = first method argument; free functions have no `self`, so `1` = first argument).
  - Fails at runtime if `Nurse` is `None` or a converted value type (e.g., `int`, `float`, `std::string` copied to `str` without weakref support).
  - Mutual `keep_alive<1, 2>` + `keep_alive<2, 1>` creates reference cycles around C++ objects whose native memory footprint is invisible to Python's cyclic GC heuristics.
- [ ] **`std::unique_ptr` vs `std::shared_ptr` holder mismatches (`pybind11` v2 vs v3 `py::smart_holder`):**
  - In `pybind11` v2, default holder is `std::unique_ptr<T>`. Passing/returning `std::shared_ptr<T>` for a class declared without `py::class_<T, std::shared_ptr<T>>` fails or causes double-destruction if raw pointers are wrapped twice; conversely, declaring `std::shared_ptr<T>` holder blocks functions taking `std::unique_ptr<T>` ownership transfer.
  - Inheriting `std::enable_shared_from_this<T>` requires `std::shared_ptr<T>` holder in v2.
  - Document how `pybind11` v3 `py::smart_holder` (`py::classh<T>`) unifies `unique_ptr` and `shared_ptr` semantics, supports safe disowning (`release()`), and fixes Python-derived trampoline slicing/lifetime bugs (`py::trampoline_self_life_support`).

### 2.2 GIL Management & Multi-Threading

- [ ] **Releasing the GIL (`py::gil_scoped_release`):** Mandatory around CPU-heavy algorithms, disk/network I/O, OpenMP parallel loops, and `std::thread::join()` / `std::future::get()`. Emphasize the strict invariant: **zero** access to `py::object`, `py::handle`, `py::list`, `py::dict`, `py::print`, or Python exception throwing while the GIL is released.
- [ ] **Re-acquiring the GIL (`py::gil_scoped_acquire`):** Required in C++ worker threads, ASIO/event-loop handlers, or C callbacks before invoking Python callables (`py::function` / wrapped `std::function`) or constructing Python objects. Guard against acquiring the GIL after `Py_FinalizeEx()` during process teardown (daemon threads / static destructors).
- [ ] **Destructor deadlocks & off-thread refcount crashes:**
  - If a C++ struct holds a `py::object` or `py::function` member and its last `std::shared_ptr` drops on a pure C++ worker thread **without** the GIL, `Py_DECREF` corrupts interpreter state or aborts in debug Python builds.
  - Worse, if a C++ thread holds `std::mutex mtx` and acquires the GIL (`py::gil_scoped_acquire`) inside a destructor/callback while a Python thread holds the GIL and blocks on `mtx.lock()`, the process deadlocks deterministically (classic AB-BA lock inversion between C++ mutexes and the GIL).
- [ ] **Python 3.13+ free-threaded builds (`Py_GIL_DISABLED` / `python3.13t`, `3.14t`):**
  - Declaring free-threading compatibility via `py::mod_gil_not_used()`.
  - `gil_scoped_acquire` no longer provides mutual exclusion between threads in `Py_GIL_DISABLED` builds! Any C++ global/static cache, non-thread-safe bound class method, or shared container previously "accidentally protected by the GIL" becomes an immediate data race requiring explicit C++ synchronization (`std::mutex` / atomics) or `Py_BEGIN_CRITICAL_SECTION`.

### 2.3 ABI, Symbol Visibility & Binary Size

- [ ] **Default hidden symbol visibility (`-fvisibility=hidden` & `-fvisibility-inlines-hidden`):**
  - `pybind11_add_module` sets this automatically, shrinking binaries by 30-60% and preventing accidental ELF symbol interposition. Manual `add_library(foo MODULE ...)` setups that forget `CXX_VISIBILITY_PRESET hidden` risk silent ODR clashes and `pybind11` compiler warnings.
- [ ] **ODR violations from linking static C++ libraries into multiple `.so`/`.pyd` modules:**
  - Linking `libengine.a` statically into both `_core.so` and `_io.so` duplicates static globals, singleton registries, and C++ `type_info` in some configurations. Passing a bound C++ object from `_core` to `_io` either fails type lookup or corrupts separate static states.
  - Solution: factor shared C++ code into a single shared library (`libengine.so` / `engine.dll`) with proper export macros, linked dynamically by all Python extension modules in the package (configured with `$ORIGIN` / `@loader_path` RPATH or `os.add_dll_directory` on Windows).
- [ ] **Cross-DLL / cross-DSO RTTI & exception translation (`Linux` / `macOS` / `Windows`):**
  - Python imports extension modules with `dlopen(..., RTLD_LOCAL)` by default on Linux. If polymorphic base classes or custom C++ exception types (`class MyError : public std::runtime_error`) have hidden visibility, `dynamic_cast` and `catch (const MyError&)` across two `.so` modules can fail because the dynamic loader does not merge hidden `typeinfo` symbols!
  - Mark cross-module polymorphic bases and C++ exception classes with explicit default visibility (`__attribute__((visibility("default")))` on ELF/Mach-O, class export macros on Windows) and register translators properly (`py::register_exception<MyError>` vs `py::register_local_exception_translator`).
  - Keep `PYBIND11_INTERNALS_ID` consistent (compiler, C++ stdlib, debug/release ABI) when sharing bound types between multiple `pybind11` modules.

### 2.4 Container Conversions & Zero-Copy Performance

- [ ] **Hidden deep-copy tax of `<pybind11/stl.h>`:**
  - Including `<pybind11/stl.h>` converts `std::vector<double>` <-> Python `list` of boxed `PyFloatObject`s by **copying all N elements on every call** (both argument input and return output).
  - Silent mutation bug: in Python, `model.weights.append(1.0)` or `model.get_items()[0] = x` mutates a temporary Python `list` copy and discards changes without raising an error!
- [ ] **Opaque STL containers (`PYBIND11_MAKE_OPAQUE` + `py::bind_vector` / `py::bind_map`):**
  - Show how `PYBIND11_MAKE_OPAQUE(std::vector<double>)` disables the `<pybind11/stl.h>` copy caster and exposes the C++ vector by reference with Python sequence protocol (`len`, indexing, slicing, `append`, iteration) plus optional `py::buffer_protocol()`.
  - Warn about the **ODR trap**: if *any* `.cpp` translation unit in the module includes `<pybind11/stl.h>` **without** seeing `PYBIND11_MAKE_OPAQUE(std::vector<double>)` first, the program violates ODR (put all `PYBIND11_MAKE_OPAQUE` declarations in a single common header!).
- [ ] **Zero-copy NumPy / buffer protocol (`py::buffer_info` & `py::array_t<T>`):**
  - Accepting NumPy arrays in C++: difference between `py::array_t<double>` (may have arbitrary strides, including Fortran-contiguous or non-contiguous slices!) vs `py::array_t<double, py::array::c_style | py::array::forcecast>` (guarantees contiguous C layout, **silently copies** when given a strided slice or wrong dtype).
  - Zero-copy C++ -> NumPy ownership transfer using `py::capsule`: heap-allocating `auto* vec = new std::vector<double>(...)` and attaching a `py::capsule(vec, [](void* p) { delete static_cast<std::vector<double>*>(p); })` as the `base` object of `py::array_t<double>` so Python's GC frees the C++ buffer only after all NumPy views expire.
  - Contrast with `nanobind::ndarray` / DLPack for zero-copy device & host tensors.

---

## 3. Planned Execution Order (EN + PL Parity)

1. **Modern Build & Quickstart:** Add CMake 3.25+ + `pyproject.toml` (`scikit-build-core`) minimal working repo template alongside the existing examples (`README.md` + `README.pl.md`).
2. **Side-by-Side Core Syntax:** Add `pybind11` tabs/blocks to Functions, Classes, Constructors, Properties, Inheritance, Virtual Functions (`PYBIND11_OVERRIDE`), and Operators.
3. **Ownership & Smart Holders:** Add deep-dive chapter on `return_value_policy`, `keep_alive`, and `pybind11` v3 `py::smart_holder` vs `Boost.Python` `CallPolicy`.
4. **Concurrency, Containers & ABI:** Add dedicated production chapters covering GIL & Python 3.13+ free-threading, opaque containers + zero-copy NumPy buffers, and multi-module visibility/ODR/RTTI rules, plus the `nanobind` comparison table.
