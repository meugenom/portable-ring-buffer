# portable-ring-buffer

> Pure C ring buffer with platform HAL — from bare metal to Linux realtime;
> Kendall's Notification Naming;
> Design by Paradigm from Kleinrock's Queueing Systems, with explicit safety rules and a new $c^2$-based approximation method;
> Alleen-Cunneen aproximaiton as a ingeneering hybrid for embedded systems;

> **Work in Progress** — API and structure are being defined. Not ready.

---

## Status

- [x] [Kendall's Notation and Variability Analysis](docs/Kenndall_Notations.md)
- [x] [Fall 1: D/D/1/K — Deterministisch, konstante Chunks, konstante Frequenzen, kein Jitter](docs/Fall-1_D_D_1_K.md)
- [ ] Fall 2: D/G/1/K — Deterministisch, variable Chunks, konstante Frequenzen, kein Jitter
- [ ] Fall 3: G/D/1/K — Deterministisch, konstante Chunks, variable Frequenzen, kein Jitter
- [ ] Fall 4: G/G/1/K — Deterministisch, konstante Chunks, konstante Frequenzen, variabler Jitter
- [ ] Fall 5: G/G/1/K — Deterministisch, variable Chunks, konstante Frequenzen, variabler Jitter
- [ ] Fall 6: G/G/1/K — Deterministisch, konstante Chunks, variable Frequenzen, variabler Jitter
- [ ] Fall 7: G/G/1/K — Deterministisch, variable Chunks, variable Frequenzen, variabler Jitter

- [ ] Implementation  mathematical logic to octave code (octave/*.m)

- [ ] Theoretical design and queueing analysis

- [ ] project structure and CMake setup

- [ ] core/ring_buffer.h — API definition
- [ ] tests/test_core.c — logic tests (TDD)
- [ ] core/ring_buffer.c — implementation

- [ ] platform/none/platform.c
- [ ] platform/linux/platform.c
- [ ] platform/stm32/platform.c

- [ ] tests/test_threaded.cpp

---

## Table of Contents

- [Status](#status)
- [Why this project exists](#why-this-project-exists)
- [Design constraints](#design-constraints)
- [Planned API](#planned-api)
- [Planned project structure](#planned-project-structure)
- [How to build and test](#how-to-build-and-test)
- [Planned test coverage](#planned-test-coverage)
- [Target platforms](#target-platforms)
- [License](#license)
- [References](#references)

---

## Why this project exists

The challenge was to build a **ring buffer** that works not only as an audio buffer,
but as a general-purpose data transport — for any realtime stream on any platform.

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
│   ├── stm32.cmake           toolchain file for arm-none-eabi-gcc
│   └── posix.cmake           toolchain file for Linux/macOS (optional, uses host compiler by default)
│
├── includes/
│   ├── ring_buffer.h         public API header for users
│   ├── platform.h            platform HAL header for core implementation
│   └── ring_buffer_config.h  user configuration (buffer size, data type, etc.)
│
├── docs/
│   ├── Roadmap_Modells_Table_Info.md  design notes and queueing theory roadmap
│   └── *.md                  future design docs and explanations
│
├── core/
│   └── ring_buffer.c         push / pop / peek / flush / fill_ratio
│
├── platform/
│   ├── platform.h            abstraction: rb_lock(), rb_unlock(), RB_LOG()
│   ├── posix/
│   │   └── platform.c        pthread_mutex + printf (Linux, macOS)  
│   ├── stm32/
│   │   └── platform.c        __disable_irq / __enable_irq + UART log
│   └── none/
│       └── platform.c        no-op (single thread, bare metal no RTOS)
│
├── tests/
│   ├── test_core.c           logic tests — no threads, runs everywhere
│   └── test_threaded.cpp     SPSC stress test — POSIX(Linux, macOS) only
│
├── examples/   
│   └── *.c                  example applications (e.g. audio piper) 
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

---

## Planned test coverage

```text
  test_core.c
  test_threaded.cpp
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

## References

- Kleinrock, L. (1975). *Queueing Systems, Vol. 1: Theory*.
- Allen, A. O. (1990). *Probability, Statistics, and Queueing Theory*.
- Little, J. D. C. (1961). A Proof for the Queuing Formula $L = \lambda W$. *Operations Research*

---

## License

MIT
