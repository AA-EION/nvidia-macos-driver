# Hyper-V GPU partitioning (GPU-P) and this driver

Fork research note, 2026-11-05. It is not an upstream design and is not proposed upstream. It follows [#417](https://github.com/nullmoth/nvidia-macos-driver/issues/417). Sources: this repository at v2.7.2 (`18ba05f`), Microsoft's dxgkrnl (`microsoft/WSL2-Linux-Kernel`, `linux-msft-wsl-6.6.y`, `drivers/hv/dxgkrnl`), ExHyperV (`Justsenger/ExHyperV` `f51e066`) and pome (`AA-EION/pome-hv`, `docs/GPU_P_MACOS.md`).

## What a GPU-P guest receives

GPU-P is WDDM GPU paravirtualization. The guest never sees an NVIDIA PCI function:

| Interface | Identity | Purpose |
|---|---|---|
| Virtual PCI device (VMBus vPCI) | `1414:008E` virtual render device (`1414:008A` compute-only) | Config-space mailbox: guest caps, interface version, vGPU LUID, host caps |
| Global VMBus channel | `{dde9cbc0-5060-4436-9448-ea1254a5d177}` | Adapter enumeration and global commands |
| Per-vGPU VMBus channel | `{6e382d18-3336-4f4b-acc4-2b7703d4df4a}` | About 90 D3DKMT commands: devices, contexts, allocations, sync objects, submits |

The host's WDDM kernel driver (`nvlddmkm.sys`) owns the GPU and executes everything. `CreateAllocation`, `CreateContext`, `SubmitCommand` and `SubmitCommandToHwQueue` carry `PrivateDriverData`, NVIDIA's undocumented UMD↔KMD format. Working Windows and Linux guests run NVIDIA's closed user-mode drivers copied from the host. Windows guests use the DriverStore DLLs. Linux guests use `libnvwgf2umx.so` and `libcuda.so` over Microsoft's `libd3d12.so`/`libdxcore.so` and the GPL `dxgkrnl`. No open-source stack produces that data, and no BSD port exists.

## Why the current stack cannot attach

The driver path is `NVMTLDriver → translator → NVK → NVRM.kext → GSP → PCI GPU`. NVRM claims the PCI device, maps BAR0/BAR1 and boots GSP with `rm_init_adapter()`. Under GPU-P there is no NVIDIA device, BAR, GSP or channel to program. Skipping the GSP boot leaves nothing to drive, and SR-IOV VFs are NVIDIA vGPU, a separate product ([#228](https://github.com/nullmoth/nvidia-macos-driver/issues/228)).

## Routes that keep the existing card paths untouched

1. **Remote Vulkan back end (most practical).** A Venus/gfxstream-style Vulkan ICD beside NVK serializes Vulkan over a VMBus pipe or hv_sock to a Windows host renderer that uses the host's NVIDIA Vulkan driver.
   - The plugin's Metal-on-Vulkan layer (`plugin/nvmtl_vk.c`) and the AIR→SPIR-V translator are reused unchanged.
   - It would be selected only when no NVIDIA PCI device is present and the host channel is offered, so NVRM/NVK on real cards is unaffected.
   - Needed: a VMBus pipe/hv_sock transport (MacHyperVSupport), the ICD, a host service, GPADL shared memory and a present path.
2. **macOS dxgkrnl port (research).** A kext matching `1414:008E` and both channels, with an IOUserClient for D3DKMT. It must be GPL-2.0, and it cannot render without NVIDIA's private ABI. That ABI comes either from NVIDIA's Linux ELF UMDs (not loadable on macOS, licensed for WSL) or from reverse engineering (version-locked and legally uncertain).
3. **DDA passthrough.** The existing driver works on a real PCI GPU, but Hyper-V supports DDA only on Windows Server, with high-MMIO and IOMMU requirements.

## Host side

pome (`AA-EION/pome-hv`) can attach a GPU partition to macOS 26 Tahoe VMs, using the same host settings as ExHyperV: guest-controlled cache types, a 1 GB/32 GB MMIO gap, checkpoints off and `Add-VMGpuPartitionAdapter`. It does this for guest-driver development only; it provides no acceleration. First check in the guest: `ioreg -l | grep -i 1414` to confirm the virtual render device reaches IOPCIFamily.

Per `AGENTS.md`, anything proposed upstream from this note must be written by the contributor personally.
