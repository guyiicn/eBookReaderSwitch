# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

eBookReaderSwitch is a Nintendo Switch homebrew eBook reader application. It supports PDF, EPUB, CBZ, and XPS file formats using the MuPDF library for document rendering and SDL2 for graphics.

## WSL (x86) 编译环境搭建

### 1. 安装 WSL Ubuntu

```powershell
# Windows PowerShell (管理员)
wsl --install -d Ubuntu
```

### 2. 安装 devkitPro 工具链

```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装依赖
sudo apt install -y wget git make build-essential libfreetype6-dev

# 下载并运行 devkitPro 安装脚本
wget https://apt.devkitpro.org/install-devkitpro-pacman
chmod +x ./install-devkitpro-pacman
sudo ./install-devkitpro-pacman

# 安装 Switch 开发工具和库
sudo dkp-pacman -S switch-dev switch-portlibs switch-sdl2 switch-sdl2_ttf switch-sdl2_image switch-libconfig

# 设置环境变量 (添加到 ~/.bashrc)
echo 'export DEVKITPRO=/opt/devkitpro' >> ~/.bashrc
echo 'export DEVKITARM=/opt/devkitpro/devkitARM' >> ~/.bashrc
echo 'export DEVKITA64=/opt/devkitpro/devkitA64' >> ~/.bashrc
echo 'export PATH=$DEVKITPRO/tools/bin:$DEVKITPRO/devkitA64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 3. 克隆并编译项目

```bash
# 克隆仓库
git clone https://github.com/guyiicn/eBookReaderSwitch.git
cd eBookReaderSwitch

# 初始化 mupdf 子模块
git submodule update --init --recursive

# 编译 MuPDF 库 (首次编译必须)
make mupdf

# 编译主程序 (不带调试器)
NODEBUG=true make

# 或者带调试器编译 (需要 twili)
# sudo dkp-pacman -S switch-twili
# make
```

### 4. 编译输出

编译成功后生成 `eBookReaderSwitch.nro`，将其复制到 Switch SD 卡的 `/switch/` 目录即可运行。

### 常见问题

- **找不到 aarch64-none-elf-gcc**: 确保环境变量已正确设置，运行 `source ~/.bashrc`
- **mupdf 编译失败**: 确保已安装 `libfreetype6-dev`
- **链接错误 -ltwili**: 使用 `NODEBUG=true make` 或安装 twili 库

## Arch Linux 编译环境搭建

### 1. 安装 devkitPro

```bash
# 导入 devkitPro GPG 密钥
sudo pacman-key --recv BC26F752D25B92CE272E0F44F7FD5492264BB9D0 --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign BC26F752D25B92CE272E0F44F7FD5492264BB9D0

# 添加 devkitPro 仓库到 /etc/pacman.conf
sudo bash -c 'cat >> /etc/pacman.conf << EOF

[dkp-libs]
Server = https://pkg.devkitpro.org/packages

[dkp-linux]
Server = https://pkg.devkitpro.org/packages/linux/\$arch/
EOF'

# 更新并安装
sudo pacman -Syu
sudo pacman -S switch-dev dkp-toolchain-vars
```

### 2. 安装 Switch 开发库

```bash
# 使用 dkp-pacman 安装 Switch 专用库
sudo dkp-pacman -S switch-portlibs switch-sdl2 switch-sdl2_ttf switch-sdl2_image switch-libconfig

# 安装系统依赖
sudo pacman -S base-devel freetype2 git
```

### 3. 设置环境变量

```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
echo 'export DEVKITPRO=/opt/devkitpro' >> ~/.bashrc
echo 'export DEVKITARM=/opt/devkitpro/devkitARM' >> ~/.bashrc
echo 'export DEVKITA64=/opt/devkitpro/devkitA64' >> ~/.bashrc
echo 'export PATH=$DEVKITPRO/tools/bin:$DEVKITPRO/devkitA64/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### 4. 编译项目

```bash
# 克隆仓库
git clone https://github.com/guyiicn/eBookReaderSwitch.git
cd eBookReaderSwitch

# 初始化子模块
git submodule update --init --recursive

# 编译 MuPDF (首次)
make mupdf

# 编译主程序
NODEBUG=true make
```

### Arch 常见问题

- **pacman-key 错误**: 尝试 `sudo pacman-key --init && sudo pacman-key --populate`
- **找不到 dkp-pacman**: 确保已安装 `switch-dev`，它会自动安装 `dkp-pacman`
- **SSL 证书错误**: 运行 `sudo pacman -S ca-certificates`

## Build Commands (Quick Reference)

```bash
# Build MuPDF library (required before first build)
make mupdf

# Build the application (release)
NODEBUG=true make

# Build with twili debugger
make

# Clean build artifacts
make clean

# Clean MuPDF build
make mupdf-clean
```

The build produces `eBookReaderSwitch.nro` which can be run on Nintendo Switch via homebrew launcher.

## Architecture

### Core Components

- **main.cpp** - Application entry point. Initializes SDL2, libnx services, fonts, and textures. Routes to either direct book opening (if path passed as argument) or the file chooser menu.

- **source/menus/book-chooser/MenuChooser.cpp** - File browser UI. Lists books from `/switch/eBookReader/books`, handles navigation, and displays compatibility warnings for non-PDF formats.

- **source/menus/book/BookReader.cpp** - Main book reading controller. Manages page navigation, zoom, dark/light mode, and layout switching. Persists last-read page positions to `/switch/eBookReader/saved_pages.cfg`.

- **source/menus/book/PageLayout.cpp** - Portrait mode page rendering using MuPDF. Handles page-to-texture conversion, zoom calculations, and panning.

- **source/menus/book/LandscapePageLayout.cpp** - Landscape mode rendering with rotated orientation.

- **source/helpers/** - Utility code: `SDL_helper.c` for drawing primitives and text, `fs.c` for filesystem operations.

### Key Technical Notes

- C++ source files use `extern "C"` blocks to include C headers, maintaining compatibility between the mixed C/C++ codebase
- MuPDF context (`fz_context`) is a global singleton initialized on first book open
- Page rendering uses MuPDF pixmaps converted to SDL textures
- Dark mode inverts the pixmap colors before texture creation
- RomFS (`romfs/resources/`) contains bundled fonts (Roboto) and UI button images

### Build System

- **Makefile** - Main build file using devkitPro's libnx switch_rules
- **Makefile.mupdf** - Cross-compiles MuPDF for ARM64 (aarch64-none-elf)
- MuPDF is built as a static library with most dependencies vendored (freetype and libjpeg are from switch-portlibs)
