# HV

A graphical Hyper-V management tool that exposes the advanced side of Hyper-V through a modern desktop UI, so users don't have to hand-type PowerShell for every knob.

Built on WPF + [WPF-UI](https://github.com/lepoco/wpfui), targeting `net8.0-windows`, works on both `win-x64` and `win-arm64`.

## Highlights

- **CPU (vCPU)** — cores / reserve / limit / weight, host resource protection, nested virtualization, migration compatibility, legacy-system compatibility, VM SMT, PMU / LBR / PEBS / IPT exposure, custom CPU name, CPU pinning, APIC mode, L3 ways & distribution, SLAT page shattering, EPP / frequency cap-floor-target, CCX-per-socket & processors-per-L3, Intel TDX / AMD SEV-SNP confidential-compute.
- **Memory** — startup / weight / dynamic memory, page size (4K / 2M / 1G), memory encryption (SEV / TDX), mapping mode, access-monitoring granularity, Intel SGX enclaves, NUMA block limits, dynamic-adjust stride, CXL support and GPA pinning.
- **Storage** — VHD / VHDX / ISO plus offline physical disk/optical pass-through, auto slot allocation across IDE / SCSI controllers, dynamic / fixed / differencing disks with 512n / 512e / 4kn sector formats, on-the-fly ISO packaging via IMAPI2.
- **GPU-PV** — one-click GPU partitioning, MMIO auto-tuning + write-combining, Windows driver injection (with minimal NVIDIA `nvlddmkm` service tree) and Linux SSH-driven driver install for Ubuntu 22.04 / 24.04 kernels. Per-partition 3D / Copy / VideoEncode / VideoDecode utilization monitor.
- **Console** — MsRdpEx-based RDP console over `127.0.0.1`, both Basic Session (VMBus, pre-boot) and Enhanced Session (RDP, clipboard / audio / resolution), fullscreen, `Ctrl+Alt+Del` injection.
- **Network** — bridged / NAT (with ICS) / no-upstream switch modes, VLAN access / trunk / private (isolated / community / promiscuous), QoS min/max bandwidth, SR-IOV / VMQ / IPsec offload, MAC-spoof / DHCP-guard / router-guard / port-mirroring / storm threshold.
- **Boot** — reorderable boot entries for Gen1 and UEFI-backed Gen2 (SCSI HDD / DVD / PXE / Windows Boot Manager), persisted through the `.vmgs` NVRAM image.
- **Spacetime (Checkpoints)** — tree view of the checkpoint timeline; continuous (standard) and static (production) snapshots; jump / merge / delete; wormhole to mount a snapshot's disk into the live VM without switching.
- **PCIe pass-through (DDA)** — device inventory with host / dismounted / virtual state, non-Server "ServerNT" identifier switch for supported Windows SKUs, GPU + FLR + physical-display-output compatibility notes baked in.
- **Virtual Switch** — topology view of external / internal / private switches, remapped as Bridged / NAT / No-Upstream, with Default Switch treated separately.
- **USB pass-through** — VMBus + USBIP forwarding (via `usbipd-win` on the host and `usbip-win2` inside the guest), no network setup required.
- **Hyper-V on ARM** — Snapdragon 8cx-and-up support, with MMIO layout tuned for 36-bit physical addressing (Gen3). Known limits: ARM64 guests only, no nested virtualization, no DDA, Gen2 only.

Bilingual UI (English + Simplified Chinese), auto light / dark theme.

## Interface

Screenshots live under [`img/`](img/) — `01.png` is the main window, `02.png`–`12.png` cover the per-feature panels.

## Quick start

### Download

Grab the latest build from this repository's Releases page and run `HV.exe` (or the equivalent binary) directly. No installer, no service.

### Build from source

Requirements:

- Visual Studio 2022 (or later) with the **.NET desktop development** workload — or a standalone [.NET 8 SDK](https://dotnet.microsoft.com/download).
- Windows 10 (Build 17134+) / 11 / Server 2019 / 2022 / 2025 as the host.
- Hyper-V role enabled:
  ```powershell
  Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
  ```

Build:

```powershell
cd src
dotnet build
```

Publish a single-file binary for both architectures:

```powershell
cd src
msbuild ExHyperV.csproj /t:PublishAllArchitectures
```

Output lands in `src/bin/publish-all/{win-x64,win-arm64}/`.

## Documentation

The `doc/` folder contains focused deep-dives:

- [`Linux.md`](doc/Linux.md) — Linux guest GPU-PV bring-up (Ubuntu 22.04 / 24.04, dxgkrnl, CUDA, Mesa/Vulkan).
- [`cpu-migration-compatibility.md`](doc/cpu-migration-compatibility.md) — which CPU features get masked when the "Migration Compatibility" bit is toggled.
- [`drivermapping.md`](doc/drivermapping.md) — how the Windows GPU-PV driver-file symlinking works inside the guest's `System32`.

## Compatibility notes

### GPU-PV (partial list, tested with GPU Caps Viewer + DXVA Checker)

| Vendor | Model | Notes |
| :--- | :--- | :--- |
| Nvidia | RTX 40 series, RTX 20 series, GTX 10 series, Tesla P100 | Full DX12 / OpenGL / Vulkan / Codec / CUDA. |
| Nvidia | GT 210 (Tesla), Tesla V100 | Not supported / crashes host. |
| Intel | Xe / Xe-HPG, UHD 730 | DX12 / OpenGL / Vulkan / Codec OK; hardware-ID identification incomplete; no CUDA path. |
| Intel | HD Graphics 5xx / older | Not supported. |
| AMD | RX 5500 XT (RDNA1) | Full stack. |
| AMD | Vega 3, Radeon 8060S | Works but incomplete hardware ID. |
| AMD | Radeon 890M (RDNA 3.5) | Boot crashes host. |
| Qualcomm | Adreno 8cx Gen 3 / X1-85 / X2-90 | Works; incomplete hardware ID. |

### PCIe pass-through (DDA)

Works on Server 2019 / 2022 / 2025 (and on select non-Server SKUs via the built-in `ServerNT` toggle). Requires IOMMU / VT-d enabled in firmware. Nvidia consumer + workstation cards generally boot and reset cleanly; Intel Arc / AMD RDNA hit `Code 43` at driver load in a VM; on-board integrated graphics generally cannot be passed through.

### WDDM ↔ Hyper-V feature matrix

| Windows build | WDDM | Notable change |
| :--- | :--- | :--- |
| 17134 | 2.4 | GPU-PV introduced. |
| 19041 | 2.7 | Guest sees the correct GPU model. |
| 20348 | 2.9 | Linux / WSL2 GPU-PV. |
| 22000 | 3.0 | DMA remap, larger GPU memory ceiling. |
| 22621 | 3.1 | UMD/KMD memory sharing. |
| 26100 | 3.2 | GPU perf counters in guest Task Manager, GPU live-migration. |

## Guest video output for GPU-PV

GPU-PV assigns a *render* device; a *display* device still needs to exist. Three options:

1. **Microsoft Hyper-V Video (default)** — max 1080p @ ~60 Hz.
2. **Virtual Display Driver + streaming (recommended)** — install [Virtual-Display-Driver](https://github.com/VirtualDrivers/Virtual-Display-Driver), pair Sunshine/Moonlight (or Parsec) inside the guest and connect with all local remote desktops closed. High resolution + high refresh + audio.
3. **USB display adapter + GPU-PV** — pass through a USB controller, plug in a DisplayLink / SM768-based adapter as the physical display device.

## USB pass-through setup

- Host: install [`usbipd-win`](https://github.com/dorssel/usbipd-win).
- Guest: install [`usbip-win2`](https://github.com/vadimgrn/usbip-win2), then run the `USBProxy.exe` guest agent bundled in the release.

Currently the tool needs to stay open to keep the USB link alive; a background service is on the roadmap.

## Requirements

- .NET 8.0 desktop runtime (bundled in single-file publishes).
- Hyper-V role enabled on the host.
- Windows 10 Pro / Enterprise / Education (Build 17134+), Windows 11 Pro / Enterprise, or Windows Server 2019 / 2022 / 2025. Home editions do not ship the Hyper-V role.
- For confidential-compute, GPU-PV and PCIe pass-through features: matching CPU + firmware support (VT-x/AMD-V, IOMMU/VT-d, SEV-SNP / TDX / SGX).

## License

Released under the [GNU General Public License v3.0](LICENSE).
