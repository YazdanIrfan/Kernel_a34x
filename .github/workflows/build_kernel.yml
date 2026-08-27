name: Build boot.image

on:
  push:
  pull_request:
  workflow_dispatch:
    inputs:
      release:
          description: '🚀 Make a release?'
          type: boolean
          required: true
          default: false
      telegram:
        description: '📢 Send to Telegram?'
        type: boolean
        required: true
        default: false
      ksu_variant:
        description: "Choose KernelSU variant"
        required: true
        type: choice
        options:
          - NO-ROOT
          - KSUN
          - KSUN-SUSFS-DEV
          - KSUN-SUSFS-v1.5.12
          - KSU-TIAN
          - KSU-TIAN-SUSFS
          - SUKISU-SUSFS
          - SUKISU
        default: NO-ROOT

permissions:
  contents: write

env:
  G_WORKSPACE: ${{ github.workspace }}
  G_EXT: ${{ github.workspace }}/external_deps
  G_KERNEL: ${{ github.workspace }}/Kernel-6.6
  DEFCONFIG: Kernel-6.6/arch/arm64/configs/gki_defconfig
  KSU_VAR: ${{ github.event.inputs.ksu_variant || 'NO-ROOT' }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout repository with submodules (shallow clone)
      - name: Checkout repository
        uses: actions/checkout@v6
        with:
          submodules: true
          fetch-depth: 1

      # Your repo has a broken symlink: kernel/kernel-6.6 -> ../kernel-6.6 (lowercase),
      # but the real directory is top-level "Kernel-6.6" (capital K). On a case-sensitive
      # filesystem that symlink never resolves. build_kernel.sh's own gen_build_config.py
      # pipeline (and the build.config files it generates) rely on this exact relative
      # path resolving from inside kernel/, so fix it here before anything else runs.
      - name: Fix kernel-6.6 symlink case mismatch
        run: |
          rm -f kernel/kernel-6.6
          ln -sfn ../Kernel-6.6 kernel/kernel-6.6
          ls -la kernel/kernel-6.6

      - name: Cleanup & Setup External Deps
        run: |
          sudo apt-get remove --purge -y "php*" "dotnet*" "mysql*" "nodejs*" "clang*" "google*"
          sudo apt-get autoremove -y
          sudo apt-get clean
          sudo rm -rf /usr/local
          mkdir -p ${{ env.G_EXT }}

      - name: Clone Other Dependencies
        working-directory: ${{ env.G_EXT }}
        run: |
          git clone https://github.com/WildKernels/AnyKernel3.git -b "gki-2.0"
          git clone https://gitlab.com/simonpunk/susfs4ksu.git -b gki-android15-6.6
          git clone https://github.com/xnnnsets/kernel_patches.git
          git clone https://github.com/xnnnsets/patch.git
          git clone https://github.com/SukiSU-Ultra/SukiSU_patch.git

      - name: Susfs4ksu (Source Copy)
        run: |
          cd ${{ env.G_EXT }}/susfs4ksu
          case "${{ env.KSU_VAR }}" in
            "KSUN-SUSFS-DEV")
              cp ./kernel_patches/include/linux/* ${{ env.G_KERNEL }}/include/linux/
              cp ./kernel_patches/fs/* ${{ env.G_KERNEL }}/fs/
              ;;
            "KSUN-SUSFS-v1.5.12")
              git checkout "f450ec00bf592d080f59b01ff6f9242456c9a427"
              cp ./kernel_patches/include/linux/* ${{ env.G_KERNEL }}/include/linux/
              cp ./kernel_patches/fs/* ${{ env.G_KERNEL }}/fs/
              ;;
            "KSU-TIAN-SUSFS"|"SUKISU-SUSFS")
              git checkout "gki-android15-6.6"
              cp ./kernel_patches/include/linux/* ${{ env.G_KERNEL }}/include/linux/
              cp ./kernel_patches/fs/* ${{ env.G_KERNEL }}/fs/
              ;;
          esac

      - name: Apply Patch susfs kernel
        run: |
          cd ${{ env.G_KERNEL }}
          patch -p1 < ${{ env.G_EXT }}/patch/6.6/Dont_reduce_TTL.patch || true

          case "${{ env.KSU_VAR }}" in
            "KSUN-SUSFS-v1.5.12")
              patch -p1 < ${{ env.G_EXT }}/kernel_patches/wild/hooks/scope_min_manual_hooks_v1.4.patch || true
              patch -p1 < ${{ env.G_EXT }}/susfs4ksu/kernel_patches/50_add_susfs_in_gki-android15-6.6.patch || true
              patch -p1 < ${{ env.G_EXT }}/kernel_patches/wild/susfs_fix_patches/v1.5.12/1_fix_base.c.patch || true
              ;;
            "KSUN-SUSFS-DEV")
              patch -p1 < ${{ env.G_EXT }}/susfs4ksu/kernel_patches/50_add_susfs_in_gki-android15-6.6.patch || true
              ;;
            "KSU-TIAN-SUSFS"|"SUKISU-SUSFS")
              patch -p1 < ${{ env.G_EXT }}/susfs4ksu/kernel_patches/50_add_susfs_in_gki-android15-6.6.patch || true
              patch -p1 < ${{ env.G_EXT }}/kernel_patches/wild/susfs_fix_patches/v1.5.12/1_fix_base.c.patch || true
              ;;
          esac

      - name: Add KernelSU
        run: |
          cd ${{ env.G_KERNEL }}
          case "${{ env.KSU_VAR }}" in
            "KSUN")
              curl -LSs "https://raw.githubusercontent.com/KernelSU-Next/KernelSU-Next/refs/heads/dev/kernel/setup.sh" | bash -
              KSU_DIR="KernelSU-Next"
              ;;
            "KSUN-SUSFS-DEV")
              curl -LSs "https://raw.githubusercontent.com/pershoot/KernelSU-Next/refs/heads/dev-susfs/kernel/setup.sh" | bash -s dev-susfs
              KSU_DIR="KernelSU-Next"
              ;;
            "KSUN-SUSFS-v1.5.12")
              curl -LSs "https://raw.githubusercontent.com/xnnnsets/KernelSU-Next/refs/heads/next-1/kernel/setup.sh" | bash -s next-1
              KSU_DIR="KernelSU-Next"
              patch -p1 -d ./KernelSU-Next < ${{ env.G_EXT }}/susfs4ksu/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch || true
              patch -p1 -d ./KernelSU-Next < ${{ env.G_EXT }}/kernel_patches/wild/susfs_fix_patches/v1.5.12/fix_core_hook.c.patch || true
              patch -p1 -d ./KernelSU-Next < ${{ env.G_EXT }}/kernel_patches/wild/susfs_fix_patches/v1.5.12/fix_sucompat.c.patch || true
              patch -p1 -d ./KernelSU-Next < ${{ env.G_EXT }}/kernel_patches/wild/susfs_fix_patches/v1.5.12/fix_kernel_compat.c.patch || true
              ;;
            "KSU-TIAN-SUSFS")
              curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
              KSU_DIR="KernelSU"
              patch -p1 -d ./KernelSU < ${{ env.G_EXT }}/susfs4ksu/kernel_patches/KernelSU/10_enable_susfs_for_ksu.patch || true
              ;;
            "KSU-TIAN")
              curl -LSs "https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh" | bash -s main
              KSU_DIR="KernelSU"
              ;;
            "SUKISU-SUSFS")
              curl -LSs "https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh" | bash -s builtin
              KSU_DIR="KernelSU"
              ;;
            "SUKISU")
              curl -LSs "https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh" | bash -
              KSU_DIR="KernelSU"
              ;;
          esac

          if [ -n "$KSU_DIR" ] && [ -d "./$KSU_DIR" ]; then
            cd ./$KSU_DIR
            KSU_GIT_VERSION=$(git rev-list --count HEAD 2>/dev/null || echo "0")
            echo "KSU_GIT_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.9.5")" >> $GITHUB_ENV
            echo "KSU_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")" >> $GITHUB_ENV
            echo "KSU_VERSION=$((30000 + KSU_GIT_VERSION))" >> $GITHUB_ENV
          fi

      - name: Configure KernelSU (Defconfig)
        run: |
          case "${{ env.KSU_VAR }}" in
            "KSUN"|"KSUN-SUSFS-DEV")
              echo "CONFIG_KSU=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBES=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBE_EVENTS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_MODULES=y" >> "${{ env.DEFCONFIG }}"
              ;;
            "KSUN-SUSFS-v1.5.12")
              echo "CONFIG_KSU=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_KPROBES_HOOK=n" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_MODULES=y" >> "${{ env.DEFCONFIG }}"
              ;;
            "KSU-TIAN"*)
              echo "CONFIG_KSU=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBES=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBE_EVENTS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_MODULES=y" >> "${{ env.DEFCONFIG }}"
              ;;
            "SUKISU-SUSFS")
              echo "CONFIG_KSU=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_DEBUG=n" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPM=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_MULTI_MANAGER_SUPPORT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBES=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBE_EVENTS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_MODULES=y" >> "${{ env.DEFCONFIG }}"
              ;;
            "SUKISU")
              echo "CONFIG_KSU=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_DEBUG=n" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPM=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBES=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KPROBE_EVENTS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_MODULES=y" >> "${{ env.DEFCONFIG }}"
              ;;
          esac

      - name: Configure IPSet Support
        run: |
          echo "# IPSet Support" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_MAX=65534" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_BITMAP_IP=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_BITMAP_IPMAC=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_BITMAP_PORT=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IP=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IPMARK=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IPPORT=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IPPORTIP=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IPPORTNET=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_IPMAC=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_MAC=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_NETPORTNET=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_NET=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_NETNET=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_NETPORT=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_HASH_NETIFACE=y" >> "${{ env.DEFCONFIG }}"
          echo "CONFIG_IP_SET_LIST_SET=y" >> "${{ env.DEFCONFIG }}"

      - name: Configure SUSFS
        run: |
          case "${{ env.KSU_VAR }}" in
            "KSUN-SUSFS-v1.5.12")
              echo "# SUSFS Configuration" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_PATH=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_TRY_UMOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_KSTAT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_OVERLAYFS=n" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SPOOF_UNAME=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_OPEN_REDIRECT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_ENABLE_LOG=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_SU=n" >> "${{ env.DEFCONFIG }}"
              ;;
            "KSUN-SUSFS-DEV")
              echo "CONFIG_KSU_SUSFS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SPOOF_UNAME=n" >> "${{ env.DEFCONFIG }}"
              ;;
            "KSU-TIAN-SUSFS")
              echo "# SUSFS Configuration" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_PATH=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_TRY_UMOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_KSTAT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_OVERLAYFS=n" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SPOOF_UNAME=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_OPEN_REDIRECT=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_ENABLE_LOG=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS_SUS_SU=n" >> "${{ env.DEFCONFIG }}"
              ;;
            "SUKISU-SUSFS")
              echo "# SUSFS Configuration" >> "${{ env.DEFCONFIG }}"
              echo "CONFIG_KSU_SUSFS=y" >> "${{ env.DEFCONFIG }}"
              ;;
          esac

      # Your build_kernel.sh already handles repo sync + clang prebuilts + the Bazel build itself,
      # so unlike KSU.zip's workflow there is no separate "Prebuilts and Clang" step here.
      - name: Build Kernel
        run: |
          KBUILD_FILE="${{ env.G_KERNEL }}/drivers/kernelsu/Kbuild"

          if [ -f "$KBUILD_FILE" ]; then
            echo "Applying version patch to $KBUILD_FILE for variant: ${{ env.KSU_VAR }}"

            case "${{ env.KSU_VAR }}" in
              "KSUN" | "KSUN-SUSFS-DEV")
                sed -i "s/KSU_VERSION_FALLBACK := [0-9]\+/KSU_VERSION_FALLBACK := ${{ env.KSU_VERSION }}/" "$KBUILD_FILE"
                sed -i "s/KSU_VERSION_TAG_FALLBACK := .*/KSU_VERSION_TAG_FALLBACK := ${{ env.KSU_GIT_TAG }}/" "$KBUILD_FILE"
                echo "KSU-Next version patched: ${{ env.KSU_VERSION }} (${{ env.KSU_GIT_TAG }})"
                ;;

              "KSU-TIAN" | "KSU-TIAN-SUSFS")
                sed -i "s/-DKSU_VERSION=[0-9]\+/-DKSU_VERSION=${{ env.KSU_VERSION }}/" "$KBUILD_FILE"
                echo "KSU-Tian version patched: ${{ env.KSU_VERSION }}"
                ;;

              *)
                echo "Variant ${{ env.KSU_VAR }} doesn't need Kbuild patching."
                ;;
            esac
          else
            echo "No Kbuild file found at $KBUILD_FILE (expected for NO-ROOT)."
          fi

          chmod +x ./build_kernel.sh
          bash build_kernel.sh

      - name: Copy kernel image
        run: |
          cp out/target/product/a34x/obj/KLEAF_OBJ/dist/kernel_device_modules-6.6/mgk_64_k66_kernel_aarch64.user/Image Image
          sha256sum Image >> Image.sha256sum

      - name: Move out directory
        run: |
          mv out/target/product/a34x/obj/KLEAF_OBJ/dist .
          cd dist; tar czvf ../dist.tar.gz *

      # Note: your repo's top-level "Kernel-6.6" is the real kernel source root.
      # (kernel/kernel-6.6 is a symlink pointing at "../kernel-6.6" — lowercase —
      # which doesn't match the actual "Kernel-6.6" folder on a case-sensitive
      # filesystem, so it's broken on the Linux CI runner.)
      - name: Extract the .config file from a kernel image
        run: |
          bash Kernel-6.6/scripts/extract-ikconfig Image >> kernel_config

          export OSRC_RELEASE="$(cat build_kernel.sh | grep SEC_BUILDNUMBER | tail | sed 's/^export SEC_BUILDNUMBER\=\"//' | sed 's/\"$//' | sed 's/ogki//')"
          echo "Based on $OSRC_RELEASE Samsung Open Source Kernel source" >> versions.txt
          echo "KernelSU variant: ${{ env.KSU_VAR }}" >> versions.txt
          echo "osrc_release=$OSRC_RELEASE" >> $GITHUB_ENV

          export VERSION="$(cat Kernel-6.6/Makefile | grep 'VERSION = ' | head -n 1 | sed 's/VERSION = //')"
          export PATCHLEVEL="$(cat Kernel-6.6/Makefile | grep 'PATCHLEVEL = ' | head -n 1 | sed 's/PATCHLEVEL = //')"
          export SUBLEVEL="$(cat Kernel-6.6/Makefile | grep 'SUBLEVEL = ' | head -n 1 | sed 's/SUBLEVEL = //')"
          export KERNEL_VERSION="$VERSION.$PATCHLEVEL.$SUBLEVEL"
          echo "Linux Kernel ARM64 $KERNEL_VERSION" >> versions.txt
          echo "kernel_release=$KERNEL_VERSION" >> $GITHUB_ENV

          export ASB_LEVEL="$(cat Kernel-6.6/build.config.common | grep ASB_SPL | tail | sed 's/^ASB_SPL\=//')"
          echo "Merged $ASB_LEVEL android15-6.6 ASB Security Patch Level" >> versions.txt
          echo "asb_level=$ASB_LEVEL" >> $GITHUB_ENV

          echo "osrc_release=$OSRC_RELEASE" >> build_info.txt
          echo "asb_level=$ASB_LEVEL" >> build_info.txt
          echo "kernel_release=$KERNEL_VERSION" >> build_info.txt
          echo "ksu_variant=${{ env.KSU_VAR }}" >> build_info.txt
          echo "$(grep 'BRANCH=' Kernel-6.6/build.config.common)" >> build_info.txt
          echo "$(grep 'KMI_GENERATION=' Kernel-6.6/build.config.common)" >> build_info.txt

      - name: Compute release filename
        run: |
          echo "pkg_name=${{ env.KSU_VAR }}_a34x_${{ env.kernel_release }}_${{ env.asb_level }}_${{ env.osrc_release }}" >> $GITHUB_ENV

      # Bonus AnyKernel3 zip (flashable from a custom recovery / KernelSU Manager),
      # using the AnyKernel3 template already cloned in "Clone Other Dependencies".
      - name: Build AnyKernel3 zip
        run: |
          cp Image ${{ env.G_EXT }}/AnyKernel3/Image
          cd ${{ env.G_EXT }}/AnyKernel3
          ZIP_NAME="AnyKernel3_${{ env.pkg_name }}.zip"
          zip -r "${{ env.G_WORKSPACE }}/$ZIP_NAME" ./*
          echo "zip_name=$ZIP_NAME" >> $GITHUB_ENV

      - name: Upload dist artifact
        uses: actions/upload-artifact@v7
        with:
          name: dist-dir-${{ env.KSU_VAR }}
          path: dist

      - name: Upload Image artifact
        uses: actions/upload-artifact@v7
        with:
          name: kernel-image-${{ env.KSU_VAR }}
          path: Image

      - name: Upload AnyKernel3 zip artifact
        uses: actions/upload-artifact@v7
        with:
          name: anykernel3-zip-${{ env.KSU_VAR }}
          path: "${{ env.zip_name }}"

      - name: Upload release assets
        if: github.event.inputs.release == 'true'
        uses: softprops/action-gh-release@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          body_path: versions.txt
          tag_name: "${{ env.pkg_name }}"
          name: "${{ env.KSU_VAR }} - ${{ env.kernel_release }} - ${{ env.asb_level }} - ${{ env.osrc_release }}"
          files: |
            Image
            Image.sha256sum
            dist.tar.gz
            kernel_config
            build_info.txt
            ${{ env.zip_name }}

      - name: Telegram Notification
        if: github.event.inputs.telegram == 'true'
        run: |
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendDocument" \
            -F chat_id="${{ secrets.TELEGRAM_CHAT_ID }}" \
            -F document=@"${{ env.zip_name }}" \
            -F caption="Build success — ${{ env.KSU_VAR }} — ${{ github.sha }}" \
            -F parse_mode="Markdown"
