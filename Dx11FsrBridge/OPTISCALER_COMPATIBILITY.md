# OptiScaler compatibility contract

> **2026-11-09 改写**：本文件此前描述的是 v1.1.5 的"六个 FSR2 导出 + GetProcAddress shim"契约。
> 那套实现已于 2026-08-29（`e2b812c`）**删除**，旧文本要求核对的两行日志
> （`ffxFsr2_get_proc_address_shim_queries mask=0x3F` / `fsr2_translation_context_created ... detoured=1`）
> 在**当前二进制里永远不会出现**。旧文本已整体替换为下面这份"真实存在的契约"。

## 当前真实契约

The production Bridge does **not** import, link, locate, or call `OptiScaler.dll` directly, and it does
**not** export any `ffxFsr2*` symbol. There is no `GetProcAddress` shim for FSR2 any more
(`Dx11FsrBridge.cpp:15138` records the removal; `hooked_get_proc_address` only redirects
`D3D11CreateDevice*` and `DisplayConfig*`).

Runtime integration is **entirely one-directional and passive**, through exactly two surfaces:

1. **Two independent rendering chains that coexist.**
   The Bridge runs its own `ffx-api` chain (`Ffx12Backend`); OptiScaler runs its own input→output chain.
   Neither hands frames to the other. `passthrough` was removed on purpose because it made OptiScaler
   lose FFX input detection (`Ffx12Backend.h:116-119`).

2. **One read-only parameter coupling.**
   The Bridge reads `[Upscalers] Dx12Upscaler` from `OptiScaler.ini` (throttled to once per 500 ms,
   `Dx11FsrBridge.cpp:596-630`) purely to decide whether to clamp the jitter it feeds
   **its own** FSR chain. No value is ever written back.

## Load order: Bridge first, OptiScaler second

Three code-level reasons:

1. OptiScaler performs its own executable-scan/initialisation **once at startup**.
2. The Bridge's `GetProcAddress` IAT hook can only cover modules that are loaded before it runs
   (`install_loader_hooks_for_loaded_modules`, `Dx11FsrBridge.cpp:5689-5729`).
3. `ffx12::preload()` must claim the FFX provider **before** OptiScaler installs its `LoadLibrary` hook,
   because that hook intercepts any `amd_fidelityfx_*` filename (`Ffx12Backend.cpp:3353-3365`,
   called at `Dx11FsrBridge.cpp:15062-15064`).

## DLSS input path (`DlssInput`, off by default)

`Dx11FsrBridge/DlssInput.{h,cpp}` adds an **optional second** input contract: instead of pretending to be
an FSR2 game, the Bridge can pretend to be an **NVSDK_NGX client** and feed OptiScaler through its
native `DLSS_DX11` input. See `DLSS_INPUT.md` for the design and the six NVIDIA-specific root causes it fixes.

This path is also passive: it only calls `LoadLibraryW("nvngx.dll")` + `GetProcAddress("NVSDK_NGX_D3D11_*")`.
OptiScaler's `LdrLoadDll` hook recognises that module name and answers with its own module handle
(`LibraryLoad_Hooks.cpp:66-75`), so the Bridge never links against OptiScaler or NVIDIA.

Compatibility is confirmed when `Dx11FsrBridge.log` contains, in this order:

```text
dlss_input_route path=dlss reason=nvidia_with_ngx_runtime ...
dlss_input_init_ok impl=optiscaler module=...\OptiScaler.dll
dlss_input_frame n=1 jitter_px=...,... mv_scale=...,... flags=0x... reset=1(first_frame) ...
```

If `reason=` reports `no_dlss_runtime`, OptiScaler is not loaded under a name OptiScaler recognises —
note that **`nvngx_dlss.dll` is not a DLSS input** (`DllNames.h:179-198` requires a path separator before
the matched suffix), only `nvngx.dll` / `_nvngx.dll` are.

Both paths (FSR2-input and DLSS-input) can be swapped without rebuilding OptiScaler; the choice lives
entirely in `Dx11FsrBridge.ini` (`DlssInput=0|1|2`). The external loader remains responsible for the
order: `Dx11FsrBridge.dll` first, `OptiScaler.dll` second.
