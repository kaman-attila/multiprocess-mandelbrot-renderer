# Multiprocess Mandelbrot Renderer

A small C program that renders the Mandelbrot set using **multiple Windows processes** that share one pixel buffer through **named shared memory**. It writes the result to a 1024×1024 PNG and reports how long the render took, so you can see how the speedup changes as you add processes.

![Rendered Mandelbrot set](fractal_renderer.png)

## Features

- **Real multiprocessing on Windows:** Windows has no `fork()`, so the parent process launches fresh copies of its own executable with `CreateProcessA`.
- **Shared memory IPC:** the pixel buffer is a named file mapping (`CreateFileMappingA` / `OpenFileMappingA`) that every worker process writes into directly. No pixel data is copied between processes.
- **Row-interleaved load balancing:** worker *i* renders rows `i, i+N, i+2N, …`, so every process gets an even mix of cheap and expensive rows.
- **High-resolution timing** with `QueryPerformanceCounter`. Only the render is timed; writing the PNG is not.
- **Smooth palette** built from Bernstein polynomials, with a square-root remap so the image doesn't come out too dark.
- **PNG output** via the single-header [stb_image_write](https://github.com/nothings/stb) library.

## How it works

1. The **parent** creates a named shared-memory region big enough for `1024 × 1024 × 3` bytes of RGB data.
2. It starts `N` **child processes** running the same executable with `--process <id> <total>`.
3. Each child opens the shared region and, for each of its rows, iterates `z = z² + c` up to `MAX_ITER = 1000` times for every pixel. It stops early once `|z|² ≥ 4`.
4. The parent waits with `WaitForMultipleObjects`, prints the elapsed time, and saves the buffer as `fractal_renderer.png`.

The view covers the complex plane from `-2.0` to `1.0` on the real axis and from `-1.5` to `1.5` on the imaginary axis.

## Building

You need **GCC on Windows**, for example [MinGW-w64](https://www.mingw-w64.org/) or [MSYS2](https://www.msys2.org/). The code uses C99 `<complex.h>`, which MSVC doesn't support the same way.

```bash
gcc -std=gnu11 -O2 main.c -o fractal_renderer.exe
```

> **Note:** keep the output name `fractal_renderer.exe`. The parent launches its workers by that name.

## Usage

```bash
./fractal_renderer.exe 8
```

The argument is the number of worker processes. It defaults to `1` and can be at most `64` (`MAXIMUM_WAIT_OBJECTS`). The program prints the render time and writes `fractal_renderer.png` to the current directory:

```
Time taken: 0.475700 seconds
```

## Benchmarks

Each configuration was run 10 times; the table shows the **median** render time.

| Processes | Contiguous blocks | Speedup | Row-interleaved | Speedup |
|:---------:|------------------:|--------:|----------------:|--------:|
| 1         | 2.793 s           | 1.00×   | 2.649 s         | 1.00×   |
| 2         | 1.487 s           | 1.88×   | 1.272 s         | 2.08×   |
| 4         | 1.431 s           | 1.95×   | 0.703 s         | 3.77×   |
| 8         | 1.027 s           | 2.72×   | 0.476 s         | 5.57×   |
| 16        | 0.633 s           | 4.41×   | 0.405 s         | 6.54×   |

### What the numbers show

- **Load imbalance matters.** In the Mandelbrot set, pixels far outside the set escape after a few iterations, but pixels inside it run the full 1000. When each process got one contiguous block of rows, the processes that got the middle of the image did most of the work. Going from 2 to 4 processes then barely helped. Interleaving the rows spreads that cost evenly, and 4 processes became about 2× faster than with contiguous blocks.
- **Scaling flattens out near the core count.** Above the machine's number of logical processors, extra processes only add scheduling and startup overhead. That's why going from 8 to 16 processes helps much less than going from 4 to 8.

## Project structure

| File                   | Description                                       |
|------------------------|---------------------------------------------------|
| `main.c`               | The renderer: parent/worker logic, Mandelbrot iteration, coloring |
| `stb_image_write.h`    | Single-header PNG writer by Sean Barrett (public domain / MIT) |
| `fractal_renderer.exe` | Prebuilt Windows binary                           |
| `fractal_renderer.png` | Sample output                                     |

## Configuration

The render settings are compile-time constants at the top of `main.c`:

```c
#define WIDTH 1024      // image width in pixels
#define HEIGHT 1024     // image height in pixels
#define MAX_ITER 1000   // iteration limit per pixel
```

The viewport bounds (`x_min`, `x_max`, `y_min`, `y_max`) are set in `render_rows()`.

## Acknowledgements

- [stb](https://github.com/nothings/stb) by Sean Barrett for `stb_image_write.h`.
