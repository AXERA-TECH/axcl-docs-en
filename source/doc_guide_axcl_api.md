# SDK API

## Overview
`AXCL API` is divided into two parts: `Runtime API` and `Native API`. The `Runtime API` is an independent API set that currently includes `Memory` for memory management and `Engine API` for driving the `AXERA NPU`. When `AXCL API` is used in the accelerator card form without codec functionality, only the `Runtime API` is needed to complete all computing tasks. When codec functionality is required, you need to understand the `Native API` and `FFMPEG` module.

## runtime
Using the `Runtime API`, you can invoke the `NPU` on the host system to complete computing tasks. The `Memory API` can allocate and free memory on both the host and the accelerator card, while the `Engine API` can handle model initialization, `IO` configuration, and inference — covering the full range of `NPU` functionality.

### runtime
(axclinit)=
#### axclInit

```c
axclError axclInit(const char *config);
```

**Description**:

System initialization, synchronous interface.

**Parameters**:

- `config [IN]`: Specifies the path to a JSON configuration file.
  - Users can configure system parameters via a JSON configuration file. Currently, log level is supported. Refer to [FAQ](https://github.com/AXERA-TECH/axcl-docs/wiki/0.FAQ#how-to-configure-runtime-log--level) for the format.
  - Passing NULL or a non-existent JSON file path is allowed; the system will use default configuration.


**Restrictions**:

- Must be paired with [`axclFinalize`](#axclfinalize) for system cleanup.
- When developing applications using any AXCL interface, this interface must be called first.
- This interface should only be called once per process.

---
(axclfinalize)=
#### axclFinailze

```c
axclError axclFinalize();
```

**Description**:

System deinitialization, releases AXCL resources within the process, synchronous interface.

**Restrictions**:

- Must be paired with [`axclInit`](#axclinit).
- Before the application process exits, this interface should be explicitly called for deinitialization.
- For C++ applications, it is not recommended to call this in destructors, as the uncertain order of singleton destruction during process exit may cause abnormal process termination.

---
(axclrtgetversion)=
#### axclrtGetVersion

```c
axclError axclrtGetVersion(int32_t *major, int32_t *minor, int32_t *patch);
```

**Description**:

Query system version number, synchronous interface.

**Parameters**:

- `major [OUT]`: Major version number.
- `minor[OUT]`: Minor version number.
- `patch [OUT]`: Patch version number.

**Restrictions**:

No special restrictions.

---
(axclrtgetsocname)=
#### axclrtGetSocName

```c
const char *axclrtGetSocName();
```

**Description**:

Query the current chip SoC name string, synchronous interface.

**Restrictions**:

No special restrictions.

---
(axclrtsetdevice)=
#### axclrtSetDevice

```c
axclError axclrtSetDevice(int32_t deviceId);
```

**Description**:

Specify the device for the current process or thread, while implicitly creating a default Context, synchronous interface.

**Parameters**:

- `deviceId [IN]`: Device ID.

**Restrictions**:

- This interface internally creates a default Context implicitly, which is automatically reclaimed by the system in [`axclrtResetDevice`](#axclrtresetdevice). It cannot be explicitly destroyed by calling [`axclrtDestroyContext`](#axclrtdestroycontext).
- In multiple threads of the same process, if this interface specifies the same deviceId, the implicitly created Context is also the same.
- Must be paired with [`axclrtResetDevice`](#axclrtresetdevice) to release device resources used by the process. Multiple calls are allowed internally via reference counting; resources are only released when the reference count reaches 0.
- In multi-device scenarios, you can switch devices in the process via this interface or [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext).

---
(axclrtresetdevice)=
#### axclrtResetDevice

```c
axclError axclrtResetDevice(int32_t deviceId);
```

**Description**:

Reset the device and release device resources, including implicitly or explicitly created Contexts, synchronous interface.

**Parameters**:

- `deviceId [IN]`: Device ID.

**Restrictions**:

- Contexts explicitly created by [`axclrtCreateContext`](#axclrtcreatecontext) should be explicitly destroyed with [`axclrtDestroyContext`](#axclrtdestroycontext) before calling this interface to release device resources.
- Must be paired with [`axclrtSetDevice`](#axclrtsetdevice); the system will automatically reclaim default Context resources.
- Multiple calls are allowed internally via reference counting; resources are only released when the reference count reaches 0.
- **The application process must ensure `axclrtResetDevice` is called before exiting, especially after handling exception signals, otherwise it will cause C++ to throw a terminated abort exception.**

---
(axclrtgetdevice)=
#### axclrtGetDevice

```c
axclError axclrtGetDevice(int32_t *deviceId);
```

**Description**:

Get the currently active device ID, synchronous interface.

**Parameters**:

- `deviceId [OUT]`: Device ID.

**Restrictions**:

- Returns an error if neither [`axclrtSetDevice`](#axclrtsetdevice) nor [`axclrtCreateContext`](#axclrtcreatecontext) has been called to specify a device.

---
(axclrtgetdevicecount)=
#### axclrtGetDeviceCount

```c
axclError axclrtGetDeviceCount(uint32_t *count);
```

**Description**:

Get the total number of connected devices, synchronous interface.

**Parameters**:

- `count [OUT]`: Number of devices.

**Restrictions**:

No special restrictions.

---
(axclrtgetdevicelist)=
#### axclrtGetDeviceList

```c
axclError axclrtGetDeviceList(axclrtDeviceList *deviceList);
```

**Description**:

Get all connected device IDs, synchronous interface.

**Parameters**:

- `deviceList[OUT]`: Information of all connected device IDs.

**Restrictions**:

No special restrictions.

---
(axclrtsynchronizedevice)=
#### axclrtSynchronizeDevice

```c
axclError axclrtSynchronizeDevice();
```

**Description**:

Synchronously execute all tasks on the current device, synchronous interface.

**Restrictions**:

At least one device must be activated.

---
(axclrtGetDeviceProperties)=
#### axclrtGetDeviceProperties

```c
axclError axclrtGetDeviceProperties(int32_t deviceId, axclrtDeviceProperties *properties);
```

**Description**:

Get device UID, CPU utilization, NPU utilization, memory, and other information, synchronous interface.

**Restrictions**:

---
(axclrtcreatecontext)=
#### axclrtCreateContext

```c
axclError axclrtCreateContext(axclrtContext *context, int32_t deviceId);
```

**Description**:

Explicitly create a Context in the current thread, synchronous interface.

**Parameters**:

- `context [OUT]`: Handle of the created Context.
- `deviceId [IN]`: Device ID.

**Restrictions**:

- User-created child threads that need to call AXCL APIs must call this interface to explicitly create or bind a Context via [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext).
- If the specified device has not been activated, this interface will first activate the device internally.
- Call [`axclrtDestroyContext`](#axclrtdestroycontext) to explicitly release Context resources.
- Multiple threads are allowed to share one Context (bound via [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext)), but task execution depends on the system's thread scheduling order. Users need to manage and maintain the synchronization of task execution between threads. For multi-threading, it is recommended to create a dedicated Context for each thread to improve code maintainability.

---
(axclrtdestroycontext)=
#### axclrtDestroyContext

```c
axclError axclrtDestroyContext(axclrtContext context);
```

**Description**:

Explicitly destroy a Context, synchronous interface.

**Parameters**:

- `context [IN]`: Handle of the created Context.

**Restrictions**:

- Can only destroy Context resources created by [`axclrtCreateContext`](#axclrtcreatecontext).

---
(axclrtsetcurrentcontext)=
#### axclrtSetCurrentContext

```c
axclError axclrtSetCurrentContext(axclrtContext context);
```

**Description**:

Bind a Context to the running thread, synchronous interface.

**Parameters**:

- `context [IN]`: Context handle.

**Restrictions**:

- If this interface is called multiple times to bind a thread, the last Context takes effect.
- If the device corresponding to the bound Context has been reset by [`axclrtResetDevice`](#axclrtresetdevice), that Context cannot be set as the thread's Context; otherwise, it will cause exceptions.
- It is recommended to use a Context in the thread where it was created. If a Context is created in thread A via [`axclrtCreateContext`](#axclrtcreatecontext) and used in thread B, the user must ensure the execution order of tasks under the same Context between the two threads.

---
(axclrtgetcurrentcontext)=
#### axclrtGetCurrentContext

```c
axclError axclrtGetCurrentContext(axclrtContext *context);
```

**Description**:

Get the Context handle bound to the thread, synchronous interface.

**Parameters**:

- `context [OUT]`: Current context handle.

**Restrictions**:

- The calling thread must have bound a Context via [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext) or created one via [`axclrtCreateContext`](#axclrtcreatecontext) before calling this.
- If [`axclrtSetCurrentContext`](#axclrtsetcurrentcontext) is called multiple times, the last set Context is returned.



### memory

(axclrtmalloc)=
#### axclrtMalloc

```c
axclError axclrtMalloc(void **devPtr, size_t size, axclrtMemMallocPolicy policy);
```

**Description**:

Allocate non-CACHED physical memory on the device side, returning a pointer to the allocated memory via **devPtr*, synchronous interface.

**Parameters**:

- `devPtr [OUT]`: Returns a pointer to the allocated device-side physical memory.
- `size [IN]`: Specifies the size of memory to allocate, in bytes.
- `policy[IN]`: Specifies the memory allocation policy, currently unused.

**Restrictions**:

- This interface allocates contiguous physical memory from the device-side CMM memory pool.
- This interface allocates non-CACHED memory, no cache coherency handling is needed.
- Call [`axclrtFree`](#axclrtfree) to free the memory.
- Frequent allocation and deallocation of memory will degrade performance. It is recommended to pre-allocate or implement secondary management to avoid frequent allocation and deallocation.

---
(axclrtmalloccached)=
#### axclrtMallocCached

```c
axclError axclrtMallocCached(void **devPtr, size_t size, axclrtMemMallocPolicy policy);
```

**Description**:

Allocate CACHED physical memory on the device side, returning a pointer to the allocated memory via **devPtr*, synchronous interface.

**Parameters**:

- `devPtr [OUT]`: Returns a pointer to the allocated device-side physical memory.
- `size [IN]`: Specifies the size of memory to allocate, in bytes.
- `policy[IN]`: Specifies the memory allocation policy, currently unused.

**Restrictions**:

- This interface allocates contiguous physical memory from the device-side CMM memory pool.
- This interface allocates CACHED memory, users must handle cache coherency.
- Call [`axclrtFree`](#axclrtfree) to free the memory.
- Frequent allocation and deallocation of memory will degrade performance. It is recommended to pre-allocate or implement secondary management to avoid frequent allocation and deallocation.

---
(axclrtfree)=
#### axclrtFree

```c
axclError axclrtFree(void *devPtr);
```

**Description**:

Free device-side allocated memory, synchronous interface.

**Parameters**:

- `devPtr [IN]`: Device memory to be freed.

**Restrictions**:

- Can only free device-side memory allocated by [`axclrtMalloc`](#axclrtmalloc) or [`axclrtMallocCached`](#axclrtmalloccached).

---
(axclrtmemflush)=
#### axclrtMemFlush

```c
axclError axclrtMemFlush(void *devPtr, size_t size);
```

**Description**:

Flush data from cache to DDR and invalidate the cache contents, synchronous interface.

**Parameters**:

- `devPtr [IN]`: Pointer to the starting address of DDR memory to flush.
- `size [IN]`: Size of DDR memory to flush, in bytes.

**Restrictions**:

No special restrictions.

---
(axclrtmeminvalidate)=
#### axclrtMemInvalidate

```c
axclError axclrtMemInvalidate(void *devPtr, size_t size);
```

**Description**:

Invalidate cache data, synchronous interface.

**Parameters**:

- `devPtr [IN]`: Pointer to the starting address of DDR memory whose cache data should be invalidated.
- `size [IN]`: DDR memory size, in bytes.

**Restrictions**:

- No special restrictions.

---
(axclrtmallochost)=
#### axclrtMallocHost

```c
axclError axclrtMallocHost(void **hostPtr, size_t size);
```

**Description**:

Allocate virtual memory on the HOST, synchronous interface.

**Parameters**:

- `hostPtr [OUT]`: Starting address of the allocated memory.
- `size [IN]`: Size of memory to allocate, in bytes.

**Restrictions**:

- Memory allocated with [`axclrtMallocHost`](#axclrtmallochost) must be freed via [`axclrtFreeHost`](#axclrtfreehost).
- Frequent allocation and deallocation of memory will degrade performance. It is recommended to pre-allocate or implement secondary management to avoid frequent allocation and deallocation.
- Allocating memory on the HOST can also be done directly with malloc, but using [`axclrtMallocHost`](#axclrtmallochost) is recommended.

---
(axclrtfreehost)=
#### axclrtFreeHost

```c
axclError axclrtFreeHost(void *hostPtr);
```

**Description**:

Free memory allocated by [`axclrtMallocHost`](#axclrtmallochost), synchronous interface.

**Parameters**:

- `hostPtr [IN]`: Starting address of the memory to free.

**Restrictions**:

- Can only free HOST memory allocated by [`axclrtMallocHost`](#axclrtmallochost).

---
(axclrtmemset)=
#### axclrtMemset

```c
axclError axclrtMemset(void *devPtr, uint8_t value, size_t count);
```

**Description**:

Can only initialize device-side memory allocated by [`axclrtMalloc`](#axclrtmalloc) or [`axclrtMallocCached`](#axclrtmalloccached), synchronous interface.

**Parameters**:

- `devPtr[IN]`: Starting address of the device-side memory to initialize.
- `value [IN]`: Value to set.
- `count [IN]`: Length of memory to initialize, in bytes.

**Restrictions**:

- Can only initialize device-side memory allocated by [`axclrtMalloc`](#axclrtmalloc) or [`axclrtMallocCached`](#axclrtmalloccached).
- Device-side memory allocated by [`axclrtMallocCached`](#axclrtmalloccached) requires calling [`axclrtMemInvalidate`](#axclrtmeminvalidate) after initialization to maintain coherency.
- HOST memory allocated by [`axclrtMallocHost`](#axclrtmallochost) should use the memset function for initialization.

---
(axclrtmemcpy)=
#### axclrtMemcpy

```c
axclError axclrtMemcpy(void *dstPtr, const void *srcPtr, size_t count, axclrtMemcpyKind kind);
```

**Description**:

Implements synchronous memory copy within HOST, between HOST and DEVICE, and within DEVICE, synchronous interface.

**Parameters**:

- `devPtr [IN]`: Destination memory address pointer.
- `srcPtr [IN]`: Source memory address pointer.
- `count [IN]`: Length of memory copy, in bytes.
- `kind [IN]`: Type of memory copy.
  - [`AXCL_MEMCPY_HOST_TO_HOST`]: Memory copy within HOST.
  - [`AXCL_MEMCPY_HOST_TO_DEVICE`]: Memory copy from HOST virtual memory to DEVICE.
  - [`AXCL_MEMCPY_DEVICE_TO_HOST`]: Memory copy from DEVICE to HOST virtual memory.
  - [`AXCL_MEMCPY_DEVICE_TO_DEVICE`]: Memory copy within DEVICE.
  - [`AXCL_MEMCPY_HOST_PHY_TO_DEVICE`]: Memory copy from HOST contiguous physical memory to DEVICE.
  - [`AXCL_MEMCPY_DEVICE_TO_HOST_PHY`]: Memory copy from DEVICE to HOST contiguous physical memory.


**Restrictions**:

- Source and destination memory must satisfy the requirements of `kind`.

---
(axclrtmemcmp)=
#### axclrtMemcmp

```c
axclError axclrtMemcmp(const void *devPtr1, const void *devPtr2, size_t count);
```

**Description**:

Implements memory comparison within DEVICE, synchronous interface.

**Parameters**:

- `devPtr1 [IN]`: Device-side address 1 pointer.
- `devPtr2 [IN]`: Device-side address 2 pointer.
- `count [IN]`: Comparison length, in bytes.

**Restrictions**:

- Only supports device-side memory comparison, and returns AXCL_SUCC(0) only when the memory contents are identical.



### engine

(axclrtengineinit)=
#### axclrtEngineInit
```c
axclError axclrtEngineInit(axclrtEngineVNpuKind npuKind);
```
**Description**:

This function initializes the `Runtime Engine`. Users must call this function before using the `Runtime Engine`.

**Parameters**:

- `npuKind [IN]`: Specifies the `VNPU` type to initialize.

**Restrictions**:

After finishing with the `Runtime Engine`, users must call [`axclrtEngineFinalize`](#axclrtenginefinalize) to clean up the `Runtime Engine`.

---

(axclrtenginegetvnpukind)=
#### axclrtEngineGetVNpuKind
```c
axclError axclrtEngineGetVNpuKind(axclrtEngineVNpuKind *npuKind);
```
**Description**:

This function retrieves the `VNPU` type used to initialize the `Runtime Engine`.

**Parameters**:
- `npuKind [OUT]`: Returns the `VNPU` type.

**Restrictions**:

Users must call [`axclrtEngineInit`](#axclrtengineinit) to initialize the `Runtime Engine` before calling this function.

---

(axclrtenginefinalize)=
#### axclrtEngineFinalize
```c
axclError axclrtEngineFinalize();
```
**Description**:

This function performs cleanup of the `Runtime Engine`. Users must call this function after completing all operations.

**Restrictions**:

Users must call [`axclrtEngineInit`](#axclrtengineinit) to initialize the `Runtime Engine` before calling this function.

---

(axclrtengineloadfromfile)=
#### axclrtEngineLoadFromFile
```c
axclError axclrtEngineLoadFromFile(const char *modelPath, uint64_t *modelId);
```
**Description**:

This function loads model data from a file and creates a model `ID`.

**Parameters**:
- `modelPath [IN]`: Storage path of the offline model file.
- `modelId [OUT]`: Model `ID` generated after loading the model, used as an identifier for subsequent operations.

**Restrictions**:

Users must call [`axclrtEngineInit`](#axclrtengineinit) to initialize the `Runtime Engine` before calling this function.

---

(axclrtengineloadfrommem)=
#### axclrtEngineLoadFromMem
```c
axclError axclrtEngineLoadFromMem(const void *model, uint64_t modelSize, uint64_t *modelId);
```
**Description**:

This function loads offline model data from memory, with the system internally managing the model's runtime memory.

**Parameters**:
- `model [IN]`: Model data stored in memory.
- `modelSize [IN]`: Size of the model data.
- `modelId [OUT]`: Model `ID` generated after loading the model, used as an identifier for subsequent operations.

**Restrictions**:

The model memory must be device memory; users must manage and free it themselves.

---

(axclrtengineunload)=
#### axclrtEngineUnload
```c
axclError axclrtEngineUnload(uint64_t modelId);
```
**Description**:

This function unloads the model with the specified model `ID`.

**Parameters**:
- `modelId [IN]`: The model `ID` to unload.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetmodelcompilerversion)=
#### axclrtEngineGetModelCompilerVersion
```c
const char* axclrtEngineGetModelCompilerVersion(uint64_t modelId);
```
**Description**:

This function retrieves the model build toolchain version.

**Parameters**:
- `modelId [IN]`: Model `ID`.

**Restrictions**:

No special restrictions.

---
(axclrtenginesetaffinity)=
#### axclrtEngineSetAffinity
```c
axclError axclrtEngineSetAffinity(uint64_t modelId, axclrtEngineSet set);
```
**Description**:

This function sets the NPU affinity of a model.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `set [OUT]`: The affinity set to configure.

**Restrictions**:

Must not be zero; the set mask bits must not exceed the affinity range.

---
(axclrtenginegetaffinity)=
#### axclrtEngineGetAffinity
```c
axclError axclrtEngineGetAffinity(uint64_t modelId, axclrtEngineSet *set);
```
**Description**:

This function retrieves the NPU affinity of a model.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `set [OUT]`: The returned affinity set.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetusage)=
#### axclrtEngineGetUsage
```c
axclError axclrtEngineGetUsage(const char *modelPath, int64_t *sysSize, int64_t *cmmSize);
```
**Description**:

This function retrieves the system memory size and CMM memory size required for model execution based on the model file.

**Parameters**:
- `modelPath [IN]`: Model path for retrieving memory information.
- `sysSize [OUT]`: System memory size required for model execution.
- `cmmSize [OUT]`: CMM memory size required for model execution.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetusagefrommem)=
#### axclrtEngineGetUsageFromMem
```c
axclError axclrtEngineGetUsageFromMem(const void *model, uint64_t modelSize, int64_t *sysSize, int64_t *cmmSize);
```
**Description**:

This function retrieves the system memory size and CMM memory size required for model execution based on model data in memory.

**Parameters**:
- `model [IN]`: User-managed model memory.
- `modelSize [IN]`: Model data size.
- `sysSize [OUT]`: System memory size required for model execution.
- `cmmSize [OUT]`: CMM memory size required for model execution.

**Restrictions**:

The model memory must be device memory; users must manage and free it themselves.

---
(axclrtenginegetusagefrommodelid)=
#### axclrtEngineGetUsageFromModelId
```c
axclError axclrtEngineGetUsageFromModelId(uint64_t modelId, int64_t *sysSize, int64_t *cmmSize);
```
**Description**:

This function retrieves the system memory size and CMM memory size required for model execution based on the model `ID`.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `sysSize [OUT]`: System memory size required for model execution.
- `cmmSize [OUT]`: CMM memory size required for model execution.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetmodeltype)=
#### axclrtEngineGetModelType
```c
axclError axclrtEngineGetModelType(const char *modelPath, axclrtEngineModelKind *modelType);
```
**Description**:

This function retrieves the model type based on the model file.

**Parameters**:
- `modelPath [IN]`: Model path for retrieving the model type.
- `modelType [OUT]`: The returned model type.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetmodeltypefrommem)=
#### axclrtEngineGetModelTypeFromMem
```c
axclError axclrtEngineGetModelTypeFromMem(const void *model, uint64_t modelSize, axclrtEngineModelKind *modelType);
```
**Description**:

This function retrieves the model type based on model data in memory.

**Parameters**:
- `model [IN]`: User-managed model memory.
- `modelSize [IN]`: Model data size.
- `modelType [OUT]`: The returned model type.

**Restrictions**:

The model memory must be device memory; users must manage and free it themselves.

---
(axclrtenginegetmodeltypefrommodelid)=
#### axclrtEngineGetModelTypeFromModelId
```c
axclError axclrtEngineGetModelTypeFromModelId(uint64_t modelId, axclrtEngineModelKind *modelType);
```
**Description**:

This function retrieves the model type based on the model `ID`.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `modelType [OUT]`: The returned model type.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetioinfo)=
#### axclrtEngineGetIOInfo
```c
axclError axclrtEngineGetIOInfo(uint64_t modelId, axclrtEngineIOInfo *ioInfo);
```
**Description**:

This function retrieves the model's IO information based on the model `ID`.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `ioInfo [OUT]`: The returned axclrtEngineIOInfo pointer.

**Restrictions**:

Users should call `axclrtEngineDestroyIOInfo` to release `axclrtEngineIOInfo` before the model `ID` is destroyed.

---
(axclrtenginedestroyioinfo)=
#### axclrtEngineDestroyIOInfo
```c
axclError axclrtEngineDestroyIOInfo(axclrtEngineIOInfo ioInfo);
```
**Description**:

This function destroys data of type `axclrtEngineIOInfo`.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetshapegroupscount)=
#### axclrtEngineGetShapeGroupsCount
```c
axclError axclrtEngineGetShapeGroupsCount(axclrtEngineIOInfo ioInfo, int32_t *count);
```
**Description**:

This function retrieves the number of IO shape groups.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `count [OUT]`: Number of shape groups.

**Restrictions**:

The Pulsar2 toolchain can specify multiple shapes during model conversion. Normal models only have one shape, so calling this function is unnecessary for standard converted models.

---
(axclrtenginegetnuminputs)=
#### axclrtEngineGetNumInputs
```c
uint32_t axclrtEngineGetNumInputs(axclrtEngineIOInfo ioInfo);
```
**Description**:

This function retrieves the number of model inputs based on `axclrtEngineIOInfo` data.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetnumoutputs)=
#### axclrtEngineGetNumOutputs
```c
uint32_t axclrtEngineGetNumOutputs(axclrtEngineIOInfo ioInfo);
```
**Description**:

This function retrieves the number of model outputs based on `axclrtEngineIOInfo` data.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetinputsizebyindex)=
#### axclrtEngineGetInputSizeByIndex
```c
uint64_t axclrtEngineGetInputSizeByIndex(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index);
```
**Description**:

This function retrieves the size of a specified input based on `axclrtEngineIOInfo` data.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `group [IN]`: Input shape group index.
- `index [IN]`: Index value of the input size to retrieve, starting from 0.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetoutputsizebyindex)=
#### axclrtEngineGetOutputSizeByIndex
```c
uint64_t axclrtEngineGetOutputSizeByIndex(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index);
```
**Description**:

This function retrieves the size of a specified output based on `axclrtEngineIOInfo` data.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `group [IN]`: Output shape group index.
- `index [IN]`: Index value of the output size to retrieve, starting from 0.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetinputnamebyindex)=
#### axclrtEngineGetInputNameByIndex
```c
const char *axclrtEngineGetInputNameByIndex(axclrtEngineIOInfo ioInfo, uint32_t index);
```
**Description**:

This function retrieves the name of a specified input.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `index [IN]`: Input IO index.

**Restrictions**:

The returned input tensor name has the same lifecycle as `ioInfo`.

---
(axclrtenginegetoutputnamebyindex)=
#### axclrtEngineGetOutputNameByIndex
```c
const char *axclrtEngineGetOutputNameByIndex(axclrtEngineIOInfo ioInfo, uint32_t index);
```
**Description**:

This function retrieves the name of a specified output.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `index [IN]`: Output IO index.

**Restrictions**:

The returned output tensor name has the same lifecycle as `ioInfo`.

---
(axclrtenginegetinputindexbyname)=
#### axclrtEngineGetInputIndexByName
```c
int32_t axclrtEngineGetInputIndexByName(axclrtEngineIOInfo ioInfo, const char *name);
```
**Description**:

This function retrieves the input index by the input tensor name.

**Parameters**:
- `ioInfo [IN]`: Model description.
- `name [IN]`: Input tensor name.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetoutputindexbyname)=
#### axclrtEngineGetOutputIndexByName
```c
int32_t axclrtEngineGetOutputIndexByName(axclrtEngineIOInfo ioInfo, const char *name);
```
**Description**:

This function retrieves the output index by the output tensor name.

**Parameters**:
- `ioInfo [IN]`: Model description.
- `name [IN]`: Output tensor name.

**Restrictions**:

No special restrictions.

---
(axclrtenginegetinputdims)=
#### axclrtEngineGetInputDims
```c
axclError axclrtEngineGetInputDims(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index, axclrtEngineIODims *dims);
```
**Description**:

This function retrieves the dimension information of a specified input.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `group [IN]`: Input shape group index.
- `index [IN]`: Input tensor index.
- `dims [OUT]`: Returned dimension information.

**Restrictions**:

The storage space for `axclrtEngineIODims` is allocated by the user. The user should release `axclrtEngineIODims` before the model's `axclrtEngineIOInfo` is destroyed.

---
(axclrtenginegetoutputdims)=
#### axclrtEngineGetOutputDims
```c
axclError axclrtEngineGetOutputDims(axclrtEngineIOInfo ioInfo, uint32_t group, uint32_t index, axclrtEngineIODims *dims);
```
**Description**:

This function retrieves the dimension information of a specified output.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `group [IN]`: Output shape group index.
- `index [IN]`: Output tensor index.
- `dims [OUT]`: Returned dimension information.

**Restrictions**:

The storage space for `axclrtEngineIODims` is allocated by the user. The user should release `axclrtEngineIODims` before the model's `axclrtEngineIOInfo` is destroyed.

---
(axclrtenginecreateio)=
#### axclrtEngineCreateIO
```c
axclError axclrtEngineCreateIO(axclrtEngineIOInfo ioInfo, axclrtEngineIO *io);
```
**Description**:

This function creates data of type `axclrtEngineIO`.

**Parameters**:
- `ioInfo [IN]`: axclrtEngineIOInfo pointer.
- `io [OUT]`: The created axclrtEngineIO pointer.

**Restrictions**:

Users should call `axclrtEngineDestroyIO` to release `axclrtEngineIO` before the model `ID` is destroyed.

---
(axclrtenginedestroyio)=
#### axclrtEngineDestroyIO
```c
axclError axclrtEngineDestroyIO(axclrtEngineIO io);
```
**Description**:

This function destroys data of type `axclrtEngineIO`.

**Parameters**:
- `io [IN]`: The axclrtEngineIO pointer to destroy.

**Restrictions**:

No special restrictions.

---
(axclrtenginesetinputbufferbyindex)=
#### axclrtEngineSetInputBufferByIndex
```c
axclError axclrtEngineSetInputBufferByIndex(axclrtEngineIO io, uint32_t index, const void *dataBuffer, uint64_t size);
```
**Description**:

This function sets the input data buffer by IO index.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `index [IN]`: Input tensor index.
- `dataBuffer [IN]`: Address of the data buffer to add.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginesetoutputbufferbyindex)=
#### axclrtEngineSetOutputBufferByIndex
```c
axclError axclrtEngineSetOutputBufferByIndex(axclrtEngineIO io, uint32_t index, const void *dataBuffer, uint64_t size);
```
**Description**:

This function sets the output data buffer by IO index.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `index [IN]`: Output tensor index.
- `dataBuffer [IN]`: Address of the data buffer to add.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginesetinputbufferbyname)=
#### axclrtEngineSetInputBufferByName
```c
axclError axclrtEngineSetInputBufferByName(axclrtEngineIO io, const char *name, const void *dataBuffer, uint64_t size);
```
**Description**:

This function sets the input data buffer by IO name.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `name [IN]`: Input tensor name.
- `dataBuffer [IN]`: Address of the data buffer to add.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginesetoutputbufferbyname)=
#### axclrtEngineSetOutputBufferByName
```c
axclError axclrtEngineSetOutputBufferByName(axclrtEngineIO io, const char *name, const void *dataBuffer, uint64_t size);
```
**Description**:

This function sets the output data buffer by IO name.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `name [IN]`: Output tensor name.
- `dataBuffer [IN]`: Address of the data buffer to add.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginegetinputbufferbyindex)=
#### axclrtEngineGetInputBufferByIndex
```c
axclError axclrtEngineGetInputBufferByIndex(axclrtEngineIO io, uint32_t index, void **dataBuffer, uint64_t *size);
```
**Description**:

This function retrieves the input data buffer by IO index.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `index [IN]`: Input tensor index.
- `dataBuffer [OUT]`: Data buffer address.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginegetoutputbufferbyindex)=
#### axclrtEngineGetOutputBufferByIndex
```c
axclError axclrtEngineGetOutputBufferByIndex(axclrtEngineIO io, uint32_t index, void **dataBuffer, uint64_t *size);
```
**Description**:

This function retrieves the output data buffer by IO index.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `index [IN]`: Output tensor index.
- `dataBuffer [OUT]`: Data buffer address.
- `size [IN]`: Data buffer size.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginegetinputbufferbyname)=
#### axclrtEngineGetInputBufferByName
```c
axclError axclrtEngineGetInputBufferByName(axclrtEngineIO io, const char *name, void **dataBuffer, uint64_t *size);
```
**Description**:

This function retrieves the input data buffer by IO name.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `name [IN]`: Input tensor name.
- `dataBuffer [OUT]`: Data buffer address.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginegetoutputbufferbyname)=
#### axclrtEngineGetOutputBufferByName
```c
axclError axclrtEngineGetOutputBufferByName(axclrtEngineIO io, const char *name, void **dataBuffer, uint64_t *size);
```
**Description**:

This function retrieves the output data buffer by IO name.

**Parameters**:
- `io [IN]`: Address of the axclrtEngineIO data buffer.
- `name [IN]`: Output tensor name.
- `dataBuffer [OUT]`: Data buffer address.

**Restrictions**:

The data buffer must be device memory; users must manage and free it themselves.

---
(axclrtenginesetdynamicbatchsize)=
#### axclrtEngineSetDynamicBatchSize
```c
axclError axclrtEngineSetDynamicBatchSize(axclrtEngineIO io, uint32_t batchSize);
```
**Description**:

This function sets the number of images to process at once in dynamic batch scenarios.

**Parameters**:
- `io [IN]`: IO for model inference.
- `batchSize [IN]`: Number of images to process at once.

**Restrictions**:

No special restrictions.

---
(axclrtenginecreatecontext)=
#### axclrtEngineCreateContext
```c
axclError axclrtEngineCreateContext(uint64_t modelId, uint64_t *contextId);
```
**Description**:

This function creates a model runtime context for the model `ID`.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `contextId [OUT]`: The created context `ID`.

**Restrictions**:

One model `ID` can create multiple runtime contexts, each running only within its own settings and memory space.

---
(axclrtengineexecute)=
#### axclrtEngineExecute
```c
axclError axclrtEngineExecute(uint64_t modelId, uint64_t contextId, uint32_t group, axclrtEngineIO io);
```
**Description**:

This function performs synchronous model inference, blocking until inference results are returned.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `contextId [IN]`: Model inference context.
- `group [IN]`: Model shape group index.
- `io [IN]`: IO for model inference.

**Restrictions**:

No special restrictions.

---
(axclrtengineexecuteasync)=
#### axclrtEngineExecuteAsync
```c
axclError axclrtEngineExecuteAsync(uint64_t modelId, uint64_t contextId, uint32_t group, axclrtEngineIO io, axclrtStream stream);
```
**Description**:

This function performs asynchronous model inference.

**Parameters**:
- `modelId [IN]`: Model `ID`.
- `contextId [IN]`: Model inference context.
- `group [IN]`: Model shape group index.
- `io [IN]`: IO for model inference.
- `stream [IN]`: Stream.

**Restrictions**:

No special restrictions.



## native

- The AXCL NATIVE module supports the SYS, VDEC, VENC, IVPS, DMADIM, ENGINE, and IVE modules.

- AXCL NATIVE API parameters are fully consistent with AX SDK APIs. The difference is that the function naming prefix is changed from AX to AXCL. Example:

  ```c
  AX_S32 AXCL_SYS_Init(AX_VOID);
  AX_S32 AXCL_SYS_Deinit(AX_VOID);

  /* CMM API */
  AX_S32 AXCL_SYS_MemAlloc(AX_U64 *phyaddr, AX_VOID **pviraddr, AX_U32 size, AX_U32 align, const AX_S8 *token);
  AX_S32 AXCL_SYS_MemAllocCached(AX_U64 *phyaddr, AX_VOID **pviraddr, AX_U32 size, AX_U32 align, const AX_S8 *token);
  AX_S32 AXCL_SYS_MemFree(AX_U64 phyaddr, AX_VOID *pviraddr);

  ...

  AX_S32 AXCL_VDEC_Init(const AX_VDEC_MOD_ATTR_T *pstModAttr);
  AX_S32 AXCL_VDEC_Deinit(AX_VOID);

  AX_S32 AXCL_VDEC_ExtractStreamHeaderInfo(const AX_VDEC_STREAM_T *pstStreamBuf, AX_PAYLOAD_TYPE_E enVideoType,
                                           AX_VDEC_BITSTREAM_INFO_T *pstBitStreamInfo);

  AX_S32 AXCL_VDEC_CreateGrp(AX_VDEC_GRP VdGrp, const AX_VDEC_GRP_ATTR_T *pstGrpAttr);
  AX_S32 AXCL_VDEC_CreateGrpEx(AX_VDEC_GRP *VdGrp, const AX_VDEC_GRP_ATTR_T *pstGrpAttr);
  AX_S32 AXCL_VDEC_DestroyGrp(AX_VDEC_GRP VdGrp);

  ...
  ```

- Please refer to the AX SDK API documentation, such as: *AX SYS API Documentation*, *AX VDEC API Documentation*, etc.

- The dynamic library naming has been changed from libax_xxx.so to libaxcl_xxx.so. The mapping table is as follows:

  | Module | AX SDK          | AXCL NATIVE SDK   |
  | ------ | --------------- | ----------------- |
  | SYS    | libax_sys.so    | libaxcl_sys.so    |
  | VDEC   | libax_vdec.so   | libaxcl_vdec.so   |
  | VENC   | libax_venc.so   | libaxcl_venc.so   |
  | IVPS   | libax_ivps.so   | libaxcl_ivps.so   |
  | DMADIM | libax_dmadim.so | libaxcl_dmadim.so |
  | ENGINE | libax_engine.so | libaxcl_engine.so |
  | IVE    | libax_ive.so    | libaxcl_ive.so    |

- Some AX SDK APIs are not supported in AXCL NATIVE. The specific list is as follows:

  | Module | AXCL NATIVE API                 | Note                                                     |
  | ------ | ------------------------------- | -------------------------------------------------------- |
  | SYS    | AXCL_SYS_EnableTimestamp        |                                                          |
  |        | AXCL_SYS_Sleep                  |                                                          |
  |        | AXCL_SYS_WakeLock               |                                                          |
  |        | AXCL_SYS_WakeUnlock             |                                                          |
  |        | AXCL_SYS_RegisterEventCb        |                                                          |
  |        | AXCL_SYS_UnregisterEventCb      |                                                          |
  | VENC   | AXCL_VENC_GetFd                 |                                                          |
  |        | AXCL_VENC_JpegGetThumbnail      |                                                          |
  | IVPS   | AXCL_IVPS_GetChnFd              |                                                          |
  |        | AXCL_IVPS_CloseAllFd            |                                                          |
  | DMADIM | AXCL_DMADIM_Cfg                 | Callback function is not supported, i.e. AX_DMADIM_MSG_T.pfnCallBack |
  | IVE    | AXCL_IVE_NPU_CreateMatMulHandle |                                                          |
  |        | AX_IVE_NPU_DestroyMatMulHandle  |                                                          |
  |        | AX_IVE_NPU_MatMul               |                                                          |

## PPL

### architecture

**libaxcl_ppl.so** is a highly integrated module that implements typical pipeline (PPL). The architecture diagram is as follows:

```bash
|-----------------------------|
|        application          |
|-----------------------------|
|      libaxcl_ppl.so         |
|-----------------------------|
|      libaxcl_lite.so        |
|-----------------------------|
|         axcl sdk            |
|-----------------------------|
|         pcie driver         |
|-----------------------------|
```

:::{Note}
`libaxcl_lite.so` and `libaxcl_ppl.so` are open-source libraries which located in `axcl/sample/axclit` and `axcl/sample/ppl` directories.
:::



#### transcode

![](../res/transcode_ppl.png)

Refer to the source code of **axcl_transcode_sample**  which located in path: `axcl/sample/ppl/transode`.



### API

#### axcl_ppl_init

axcl runtime system and ppl initialization.

##### prototype

axclError axcl_ppl_init(const axcl_ppl_init_param* param);

##### parameter

| parameter | description | in/out |
| --------- | ----------- | ------ |
| param     |             | in     |

```c
typedef enum {
    AXCL_LITE_NONE = 0,
    AXCL_LITE_VDEC = (1 << 0),
    AXCL_LITE_VENC = (1 << 1),
    AXCL_LITE_IVPS = (1 << 2),
    AXCL_LITE_JDEC = (1 << 3),
    AXCL_LITE_JENC = (1 << 4),
    AXCL_LITE_DEFAULT = (AXCL_LITE_VDEC | AXCL_LITE_VENC | AXCL_LITE_IVPS),
    AXCL_LITE_MODULE_BUTT
} AXCLITE_MODULE_E;

typedef struct {
    const char *json; /* axcl.json path */
    AX_S32 device;
    AX_U32 modules;
    AX_U32 max_vdec_grp;
    AX_U32 max_venc_thd;
} axcl_ppl_init_param;
```

| parameter     | description                                  | in/out |
| ------------- | -------------------------------------------- | ------ |
| json          | json file path pass to **axclInit** API.     | in     |
| device        | device id                                    | in     |
| modules       | bitmask of AXCLITE_MODULE_E according to PPL | in     |
| max_vdec_grp  | total VDEC groups of all processes           | in     |
| max_venc_thrd | max VENC threads of each process             | in     |

> [!IMPORTANT]
>
> - AXCL_PPL_TRANSCODE:  modules = AXCL_LITE_DEFAULT
> - **max_vdec_grp** should be equal to or greater than the total number of VDEC groups across all processes. For example, if 16 processes are launched, with each process having one decoder, **max_vdec_grp** should be set to at least 16.
> - **max_venc_thrd** usually be configured as same as the total VENC channels of each process.

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_deinit

axcl runtime system and ppl deinitialization.

##### prototype

axclError axcl_ppl_deinit();

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_create

create ppl.

##### prototype

axclError axcl_ppl_create(axcl_ppl* ppl, const axcl_ppl_param* param);

##### parameter

| parameter | description                         | in/out |
| --------- | ----------------------------------- | ------ |
| ppl       | ppl handle created                  | out    |
| param     | parameters related to specified ppl | in     |

```c
typedef enum {
    AXCL_PPL_TRANSCODE = 0, /* VDEC -> IVPS ->VENC */
    AXCL_PPL_BUTT
} axcl_ppl_type;

typedef struct {
    axcl_ppl_type ppl;
    void *param;
} axcl_ppl_param;
```

**AXCL_PPL_TRANSCODE** parameters as shown as follows:

```C
typedef AX_VENC_STREAM_T axcl_ppl_encoded_stream;
typedef void (*axcl_ppl_encoded_stream_callback_func)(axcl_ppl ppl, const axcl_ppl_encoded_stream *stream, AX_U64 userdata);

typedef struct {
    axcl_ppl_transcode_vdec_attr vdec;
    axcl_ppl_transcode_venc_attr venc;
    axcl_ppl_encoded_stream_callback_func cb;
    AX_U64 userdata;
} axcl_ppl_transcode_param;
```

| parameter | description                                               | in/out |
| --------- | --------------------------------------------------------- | ------ |
| vdec      | decoder attribute                                         | in     |
| venc      | encoder attribute                                         | in     |
| cb        | callback function to receive encoded nalu frame data      | in     |
| userdata  | userdata bypass to  axcl_ppl_encoded_stream_callback_func | in     |

:::{Warning}
Avoid high-latency processing inside the *axcl_ppl_encoded_stream_callback_func* function
:::


```c
typedef struct {
    AX_PAYLOAD_TYPE_E payload;
    AX_U32 width;
    AX_U32 height;
    AX_VDEC_OUTPUT_ORDER_E output_order;
    AX_VDEC_DISPLAY_MODE_E display_mode;
} axcl_ppl_transcode_vdec_attr;
```

| parameter    | description                                                  | in/out |
| ------------ | ------------------------------------------------------------ | ------ |
| payload      | PT_H264 \| PT_H265                                           | in     |
| width        | max. width of input stream                                   | in     |
| height       | max. height of input stream                                  | in     |
| output_order | AX_VDEC_OUTPUT_ORDER_DISP  \| AX_VDEC_OUTPUT_ORDER_DEC       | in     |
| display_mode | AX_VDEC_DISPLAY_MODE_PREVIEW \| AX_VDEC_DISPLAY_MODE_PLAYBACK | in     |

:::{Important}
- **output_order**:
  - If decode sequence is same as display sequence such as IP stream, recommend to AX_VDEC_OUTPUT_ORDER_DEC to save memory.
  - If decode sequence is different to display sequence such as IPB stream, set AX_VDEC_OUTPUT_ORDER_DISP.
- **display_mode**
  - AX_VDEC_DISPLAY_MODE_PREVIEW:  preview mode which frame dropping is allowed typically for RTSP stream... etc.
  - AX_VDEC_DISPLAY_MODE_PLAYBACK: playback mode which frame dropping is not forbidden typically for local stream file.
  :::

```c
typedef struct {
    AX_PAYLOAD_TYPE_E payload;
    AX_U32 width;
    AX_U32 height;
    AX_VENC_PROFILE_E profile;
    AX_VENC_LEVEL_E level;
    AX_VENC_TIER_E tile;
    AX_VENC_RC_ATTR_T rc;
    AX_VENC_GOP_ATTR_T gop;
} axcl_ppl_transcode_venc_attr;
```

| parameter | description                         | in/out |
| --------- | ----------------------------------- | ------ |
| payload   | PT_H264 \| PT_H265                  | in     |
| width     | output width of encoded nalu frame  | in     |
| height    | output height of encoded nalu frame | in     |
| profile   | h264 or h265 profile                | in     |
| level     | h264 or h265 level                  | in     |
| tile      | tile                                | in     |
| rc        | rate control settings               | in     |
| gop       | gop settings                        | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_destroy

destroy ppl.

##### prototype

axclError axcl_ppl_destroy(axcl_ppl ppl)

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

#### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_start

start ppl.

##### prototype

axclError axcl_ppl_start(axcl_ppl ppl);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_stop

stop ppl.

##### prototype

axclError axcl_ppl_stop(axcl_ppl ppl);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_send_stream

send nalu frame to ppl.

##### prototype

axclError axcl_ppl_send_stream(axcl_ppl ppl, const axcl_ppl_input_stream* stream, AX_S32 timeout);

##### parameter

| parameter | description             | in/out |
| --------- | ----------------------- | ------ |
| ppl       | ppl handle created      | in     |
| stream    | nalu frame              | in     |
| timeout   | timeout in milliseconds | in     |

```c
typedef struct {
    AX_U8 *nalu;
    AX_U32 nalu_len;
    AX_U64 pts;
    AX_U64 userdata;
} axcl_ppl_input_stream;
```

| parameter | description                                                  | in/out |
| --------- | ------------------------------------------------------------ | ------ |
| nalu      | pointer to nalu frame data                                   | in     |
| nalu_len  | bytes of nalu frame data                                     | in     |
| pts       | timestamp of nalu frame                                      | in     |
| userdata  | userdata bypass to ***axcl_ppl_encoded_stream.stPack.u64UserData*** | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_get_attr

get attribute of ppl.

##### prototype

axclError axcl_ppl_get_attr(axcl_ppl ppl, const char* name, void* attr);

##### parameter

| parameter | description        | in/out |
| --------- | ------------------ | ------ |
| ppl       | ppl handle created | in     |
| name      | attribute name     | in     |
| attr      | attribute value    | out    |

```c
/**
 *            name                                     attr type        default
 *  axcl.ppl.id                             [R  ]       int32_t                            increment +1 for each axcl_ppl_create
 *
 *  axcl.ppl.transcode.vdec.grp             [R  ]       int32_t                            allocated by ax_vdec.ko
 *  axcl.ppl.transcode.ivps.grp             [R  ]       int32_t                            allocated by ax_ivps.ko
 *  axcl.ppl.transcode.venc.chn             [R  ]       int32_t                            allocated by ax_venc.ko
 *
 *  the following attributes take effect BEFORE the axcl_ppl_create function is called:
 *  axcl.ppl.transcode.vdec.blk.cnt         [R/W]       uint32_t          8                depend on stream DPB size and decode mode
 *  axcl.ppl.transcode.vdec.out.depth       [R/W]       uint32_t          4                out fifo depth
 *  axcl.ppl.transcode.ivps.in.depth        [R/W]       uint32_t          4                in fifo depth
 *  axcl.ppl.transcode.ivps.out.depth       [R  ]       uint32_t          0                out fifo depth
 *  axcl.ppl.transcode.ivps.blk.cnt         [R/W]       uint32_t          4
 *  axcl.ppl.transcode.ivps.engine          [R/W]       uint32_t   AX_IVPS_ENGINE_VPP      AX_IVPS_ENGINE_VPP|AX_IVPS_ENGINE_VGP|AX_IVPS_ENGINE_TDP
 *  axcl.ppl.transcode.venc.in.depth        [R/W]       uint32_t          4                in fifo depth
 *  axcl.ppl.transcode.venc.out.depth       [R/W]       uint32_t          4                out fifo depth
 */
```

:::{Important}
**axcl.ppl.transcode.vdec.blk.cnt**:  blk count is related to the DBP size of input stream, recommend to set dbp + 1.
:::

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



#### axcl_ppl_set_attr

set attribute of ppl.

##### prototype

axclError axcl_ppl_set_attr(axcl_ppl ppl, const char* name, const void* attr);

##### parameter

| parameter | description                                      | in/out |
| --------- | ------------------------------------------------ | ------ |
| ppl       | ppl handle created                               | in     |
| name      | attribute name, refer to ***axcl_ppl_get_attr*** | in     |
| attr      | attribute value                                  | in     |

##### return

0 (AXCL_SUCC)  if success, otherwise failure.



## Error Codes

### Definition

```c
typedef int32_t axclError;

typedef enum {
    AXCL_SUCC                   = 0x00,
    AXCL_FAIL                   = 0x01,
    AXCL_ERR_UNKNOWN            = AXCL_FAIL,
    AXCL_ERR_NULL_POINTER       = 0x02,
    AXCL_ERR_ILLEGAL_PARAM      = 0x03,
    AXCL_ERR_UNSUPPORT          = 0x04,
    AXCL_ERR_TIMEOUT            = 0x05,
    AXCL_ERR_BUSY               = 0x06,
    AXCL_ERR_NO_MEMORY          = 0x07,
    AXCL_ERR_ENCODE             = 0x08,
    AXCL_ERR_DECODE             = 0x09,
    AXCL_ERR_UNEXPECT_RESPONSE  = 0x0A,
    AXCL_ERR_OPEN               = 0x0B,
    AXCL_ERR_EXECUTE_FAIL       = 0x0C,

    AXCL_ERR_BUTT               = 0x7F
} AXCL_ERROR_E;

#define AX_ID_AXCL           (0x30)

/* module */
#define AXCL_RUNTIME         (0x00)
#define AXCL_NATIVE          (0x01)
#define AXCL_LITE            (0x02)

/* runtime sub module */
#define AXCL_RUNTIME_DEVICE  (0x01)
#define AXCL_RUNTIME_CONTEXT (0x02)
#define AXCL_RUNTIME_STREAM  (0x03)
#define AXCL_RUNTIME_TASK    (0x04)
#define AXCL_RUNTIME_MEMORY  (0x05)
#define AXCL_RUNTIME_CONFIG  (0x06)
#define AXCL_RUNTIME_ENGINE  (0x07)
#define AXCL_RUNTIME_SYSTEM  (0x08)

/**
 * |---------------------------------------------------------|
 * | |   MODULE    |  AX_ID_AXCL | SUB_MODULE  |    ERR_ID   |
 * |1|--- 7bits ---|--- 8bits ---|--- 8bits ---|--- 8bits ---|
 **/
#define AXCL_DEF_ERR(module, sub, errid) \
    ((axclError)((0x80000000L) | (((module) & 0x7F) << 24) | ((AX_ID_AXCL) << 16 ) | ((sub) << 8) | (errid)))

#define AXCL_DEF_RUNTIME_ERR(sub, errid)    AXCL_DEF_ERR(AXCL_RUNTIME, (sub), (errid))
#define AXCL_DEF_NATIVE_ERR(sub,  errid)    AXCL_DEF_ERR(AXCL_NATIVE,  (sub), (errid))
#define AXCL_DEF_LITE_ERR(sub,    errid)    AXCL_DEF_ERR(AXCL_LITE,    (sub), (errid))
```

:::{Note}

Error codes are divided into two types: AXCL runtime library error codes and AX NATIVE SDK error codes, distinguished by the third byte of `axclError`. If the third byte equals AX_ID_AXCL (0x30), it indicates an AXCL runtime library error code; otherwise, it indicates a passthrough error code from the device's AX NATIVE SDK module.
- AXCL runtime library error codes: Refer to the `axcl_rt_xxx.h` header files.
- Device NATIVE SDK error codes are passed through to the HOST side. Refer to the *AX Software Error Code Documentation*.
:::

### Parsing

Please visit [AXCL Error Code Parser](https://gakki2019.github.io/axcl/) to parse error codes online.
