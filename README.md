# portable-ring-buffer

> Pure C ring buffer with platform HAL — from bare metal to Linux realtime

> **Work in Progress** — API and structure are being defined. Not ready for production use.

---

## Table of Contents

- [Why this project exists](#why-this-project-exists)
- [Design constraints](#design-constraints)
- [Checklist — before writing any ring buffer](#checklist--before-writing-any-ring-buffer)
- [Planned API](#planned-api)
- [Planned project structure](#planned-project-structure)
- [How to build and test](#how-to-build-and-test)
- [Planned test coverage](#planned-test-coverage)
- [Target platforms](#target-platforms)
- [Status](#status)
- [License](#license)

---

## Why this project exists

The challenge was to build a **universal ring buffer** that works not only as an audio buffer,
but as a general-purpose data transport — for any realtime stream on any platform.

Audio playback from **PiperTTS** is just one example. The same buffer must work equally well
for gyroscope data on **STM32**, UART streams on bare metal, or network packets on Linux —
without changing a single line of core code.

This forced a clean separation between logic and platform.

---

## Design constraints

```
NO   malloc / new          — forbidden in realtime and embedded hot path
NO   C++ STL               — not available on bare metal
NO   pthread               — POSIX only
NO   exceptions            — no -fexceptions on microcontrollers
NO   dynamic sizing        — buffer capacity fixed at compile time

YES  pure C99/C11          — compiles everywhere
YES  static memory         — no leaks possible by design
YES  platform HAL          — thread safety is swappable per target
YES  power-of-two capacity — fast index wrap with bitmask
```

---

## Checklist — before writing any ring buffer

Answer these 8 questions first. The implementation follows directly.

```
1. What is stored?          data type (bytes, frames, structs)
2. Who produces data?       socket, ISR, microphone, file
3. Who consumes data?       playback, network, processing
4. How much to buffer?      size = latency × stream rate
5. How many producers?      SPSC / MPSC / MPMC → pick thread safety strategy
6. On overflow?             DROP_NEW / DROP_OLD / BLOCK
7. On underrun?             SILENCE / BLOCK / error code
8. Target platform?         bare metal / RTOS / Linux / macOS
```

---

## Planned API

```c
RBError  rb_init       (RingBuffer *rb);
RBError  rb_push       (RingBuffer *rb, const void *data);
RBError  rb_pop        (RingBuffer *rb,       void *data);
RBError  rb_peek       (const RingBuffer *rb, void *data);
void     rb_flush      (RingBuffer *rb);
float    rb_fill_ratio (const RingBuffer *rb);   // 0.0 empty → 1.0 full
uint32_t rb_size       (const RingBuffer *rb);
```

Error codes:
```c
typedef enum {
    RB_OK        = 0,
    RB_ERR_FULL  = 1,   // overrun
    RB_ERR_EMPTY = 2,   // underrun
    RB_ERR_NULL  = 3,   // null pointer
} RBError;
```

Errors are never silently ignored — every overrun and underrun is logged via `RB_LOG()`.

---

## Planned project structure

```
portable-ring-buffer/
│
├── CMakeLists.txt
├── cmake/
│   ├── stm32.cmake          toolchain file for arm-none-eabi-gcc
│   └── platform.cmake       platform-specific compile flags
│
├── core/
│   ├── ring_buffer.h       pure C API — no platform includes
│   └── ring_buffer.c       push / pop / peek / flush / fill_ratio
│
├── platform/
│   ├── platform.h          abstraction: rb_lock(), rb_unlock(), RB_LOG()
│   ├── linux/
│   │   └── platform.c      pthread_mutex + printf
│   ├── macos/
│   │   └── platform.c      pthread_mutex + printf  (same as linux)
│   ├── stm32/
│   │   └── platform.c      __disable_irq / __enable_irq + UART log
│   └── none/
│       └── platform.c      no-op (single thread, bare metal no RTOS)
│
├── tests/
│   ├── test_core.c         logic tests — no threads, runs everywhere
│   └── test_threaded.cpp   SPSC stress test — Linux / macOS
│
├── examples/
│   ├── audio_piper/        PiperTTS → Unix socket → ring buffer → playback
│   └── gyroscope_stm32/    gyroscope ISR → ring buffer → processing
│
├── README.md
└── LICENSE
```

---

## How to build and test

> Tests are written before implementation (TDD approach).

```bash
# Clone
git clone https://github.com/meugenom/portable-ring-buffer
cd portable-ring-buffer

# Configure (choose platform: linux / macos / none)
cmake -B build -DPLATFORM=linux
cmake --build build

# Run tests
cd build && ctest --output-on-failure
```

For STM32 (cross-compile):
```bash
cmake -B build-stm32 \
      -DCMAKE_TOOLCHAIN_FILE=cmake/stm32.cmake \
      -DPLATFORM=stm32
cmake --build build-stm32
```

---

## Planned test coverage

```
test_core.c
  ✓ initial state (empty, size=0, fill=0.0)
  ✓ push one element, pop one element
  ✓ FIFO order preserved
  ✓ wraparound across multiple cycles
  ✓ push until full → RB_ERR_FULL
  ✓ pop from empty → RB_ERR_EMPTY
  ✓ peek does not remove element
  ✓ flush resets buffer
  ✓ fill_ratio: 0.0 / 0.5 / 1.0

test_threaded.cpp
  ✓ SPSC: 100,000 frames, producer + consumer threads
  ✓ all frames received in correct order
  ✓ no frames lost or duplicated
  ✓ buffer empty after completion
```

---

## Target platforms

| Platform         | Compiler              | Thread safety   | Log        |
|------------------|-----------------------|-----------------|------------|
| Linux x86_64     | gcc / clang           | pthread_mutex   | printf     |
| Linux ARM64      | gcc aarch64           | pthread_mutex   | printf     |
| macOS ARM64      | clang (Apple Silicon) | pthread_mutex   | printf     |
| macOS x86_64     | clang                 | pthread_mutex   | printf     |
| STM32 bare metal | arm-none-eabi-gcc     | __disable_irq   | UART / ITM |

---

## Status

- [ ] core/ring_buffer.h — API definition
- [ ] tests/test_core.c — logic tests (TDD)
- [ ] core/ring_buffer.c — implementation
- [ ] platform/none/platform.c
- [ ] platform/linux/platform.c
- [ ] platform/stm32/platform.c
- [ ] tests/test_threaded.cpp
- [ ] examples/audio_piper

---

## License

MIT
