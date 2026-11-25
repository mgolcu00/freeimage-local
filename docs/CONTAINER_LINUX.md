# Running in a CUDA Container (Fedora Silverblue / Bazzite / Immutable Linux)

On immutable Linux hosts (such as Fedora Silverblue / Bazzite) where `/usr` is read-only and `nvcc` is not installed on the host, you can build and run `stable-diffusion.cpp` (`--sage-attn`) inside a rootless Podman/Docker CUDA 12.8 container.

## 1. Build the CUDA 12.8 Builder/Runtime Image

```bash
podman build -f Containerfile.cuda128 -t freeimage-sdcpp:cu128 .
```

## 2. Compile `stable-diffusion.cpp` for `sm_86` (RTX 3060 Laptop 6 GB)

```bash
podman run --rm --device nvidia.com/gpu=all --security-opt=label=disable \
  -v "$PWD/sdcpp:/workspace/sdcpp" freeimage-sdcpp:cu128 \
  cmake -S /workspace/sdcpp/stable-diffusion.cpp -B /workspace/sdcpp/stable-diffusion.cpp/build \
  -G Ninja -DSD_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=86 -DCMAKE_BUILD_TYPE=Release
```

> **Note for Podman 6.x + NVIDIA CDI:** Avoid combining `--userns=keep-id` with `--device nvidia.com/gpu=all` if `mount_setattr` returns `EPERM`; use `--security-opt=label=disable` instead.

## 3. Reproducible Benchmark (`RTX 3060 Laptop 6 GB`, `sm_86`, `--sage-attn`)

| Engine / Model | Resolution / Steps | Wall (s) | Peak VRAM (MiB) | Peak Temp (°C) |
| :--- | :--- | :---: | :---: | :---: |
| ComfyUI PyTorch (`Qwen 2.1 Q6_K` baseline) | `768×1344 → 1080×1920`, 14 steps | 138.90 | 4,976 | 85.6 |
| `sd-cli` (`z_image_turbo_int8_convrot` + `Qwen3-4B-Q8_0`) | `1024×1024`, 8 steps, CFG 1.0 | **22.51** | 5,748 | 74.3 |
| `sd-server` warm (`z_image_turbo_int8_convrot`) | `2× 1024×1024`, 8 steps, CFG 1.0 | **23.86** | 5,562 | 78.4 |
| `sd-server` warm (`flux-3-image-klein-Q4_K_M` + `Qwen3VL-4B`) | `10× 768×1024`, 16 steps, CFG 1.0 | **16.63 / img** | **5,385** | 85.8 |
