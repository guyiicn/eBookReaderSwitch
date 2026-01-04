# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

eBookReaderSwitch is a Nintendo Switch homebrew eBook reader application. It supports PDF, EPUB, CBZ, and XPS file formats using the MuPDF library for document rendering and SDL2 for graphics.

## Build Commands

**Prerequisites:** devkitPro toolchain with libnx and switch-portlibs installed.

```bash
# Install dependencies (via devkitPro pacman)
pacman -S libnx switch-portlibs

# Build MuPDF library (required before first build)
make mupdf

# Build the application
make

# Build without twili debugger (for release/CI)
NODEBUG=true make

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
