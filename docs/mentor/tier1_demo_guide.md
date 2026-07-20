# Tier 1 Progress Summary & Demo Guide (CVE-2026-46242)

## 1. Executive Summary of Progress
We have successfully built the **Tier 1 QEMU VM** running the target **Kernel 6.12.67 LTS**. The environment was compiled from scratch on a Fedora host, including the `bzImage` and a custom `initramfs` (busybox-based). 

The original kernelCTF exploit (written in C++) was rigorously patched to run in our local environment:
- Fixed modern GCC 16 C++ standard library compilation errors (e.g., `std::reference_wrapper` strictness).
- Hardcoded the target database matching to recognize our custom kernel build string (`kernelctf` / `lts-6.12.67`).
- Bypassed the `rdtscp` KASLR leak due to QEMU `kvm64` SIGILL limitations (we boot QEMU with `nokaslr` and hardcode the base address).
- Dynamically calibrated the timing thresholds to win the `epoll` Use-After-Free race condition despite virtualization overhead.
- Mapped and patched custom structural offsets for `task_struct` (e.g., `comm=1840`, `files=1896`) that shifted during our local kernel compilation.

**Current Status:**
The exploit **successfully wins the race**, triggers the UAF, allocates the cross-cache pipe, and achieves an Arbitrary Address Read (AAR). The pipeline panics at the final step (payload execution) due to a misinterpretation of a ROP stack pivot gadget (`#PF: supervisor write access`). The memory corruption primitives are 100% stable; we are currently blocked purely on finding the correct stack pivot in `vmlinux` to finish the root ROP chain.

---

## 2. Manual Run Steps (Demo Execution)

Use the following exact commands to demonstrate the current working state of the exploit. **Do not run these out of order.**

### Step A: Navigate to the Tier 1 Directory
Ensure you are in the correct directory where the exploit artifacts and boot scripts reside:
```bash
cd /mnt/work/company/cyphermatrix/repos/bad-epoll-lab/exploit/tier1
```

### Step B: Verify the Packed Initramfs
Ensure that the pre-packaged initramfs (which already contains the statically compiled exploit) is present in the directory. You can list the files to confirm:
```bash
ls -lh initramfs_exploit.cpio
```
*(If you need to re-pack it manually for any reason, the `rootfs_build/` directory contains the uncompressed filesystem structure.)*

### Step C: Boot the QEMU VM
Run the provided boot script which automatically launches QEMU with the correct kernel path (`../../third_party/...`) and CPU configurations (including `nokaslr`):
```bash
./boot.sh
```

### Step D: Execute the Exploit (Inside QEMU)
Once QEMU boots, you will see a root shell `#` inside the VM. Run the injected exploit binary:
```bash
/exploit
# (or /bin/exploit depending on the active init script)
```

*Note: The exploit will run, successfully win the race, complete the Arbitrary Address Read, and then kernel panic during the pivot execution. This is the expected and correct "current progress" state showing the UAF vulnerability is active and working.*
