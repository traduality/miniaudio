<h1 align="center">
    <a href="https://miniaud.io"><img src="https://miniaud.io/img/miniaudio_wide.png" alt="miniaudio" width="1280"></a>
    <br>
</h1>

<h4 align="center">An audio playback and capture library in a single source file.</h4>

<p align="center">
    <a href="https://discord.gg/9vpqbjU"><img src="https://img.shields.io/discord/712952679415939085?label=discord&logo=discord&style=flat-square" alt="discord"></a>
    <a href="https://fosstodon.org/@mackron"><img src="https://img.shields.io/mastodon/follow/109293691403797709?color=blue&domain=https%3A%2F%2Ffosstodon.org&label=mastodon&logo=mastodon&style=flat-square" alt="mastodon"></a>
    <a href="https://www.reddit.com/r/miniaudio"><img src="https://img.shields.io/reddit/subreddit-subscribers/miniaudio?label=r%2Fminiaudio&logo=reddit&style=flat-square" alt="reddit"></a>
</p>

<p align="center">
    <a href="#examples">Examples</a> -
    <a href="#documentation">Documentation</a> -
    <a href="#supported-platforms">Supported Platforms</a> -
    <a href="#backends">Backends</a> -
    <a href="#major-features">Major Features</a> -
    <a href="#building">Building</a>
</p>

miniaudio is a C library (compilable as C++) with a simple API and build system. It has no dependencies and
should compile cleanly on all major compilers. You can add the source file directly to your source tree
without needing to install any additional development packages. See the <a href="#building">Building</a>
section for more details.

All major platforms are supported, and where applicable, each platform has support for multiple backends so
that the best backend can be used when available.

miniaudio is primarily concerned with audio playback and capture. The low-level API is a simple connection
to a physical device with a callback used for delivery of raw audio data. Built on top of the low-level API
is an optional high-level API which is highly flexible and usable for most scenarios.

Included in miniaudio is a suite of functionality not directly related to playback and capture, but highly
useful for a creating a complete audio solution. Such functionality includes a node graph for advanced mixing
and effect processing, resource management, decoding, resampling, format conversion, and more. You can piece
these together to help you develop your audio solution.

Refer to the [Programming Manual](https://miniaud.io/docs/manual/) for a more complete description of
available features in miniaudio.


Examples
========

This example shows one way to play a sound using the high level API.

```c
#define MINIAUDIO_IMPLEMENTATION
#include "../miniaudio.h"

#include <stdio.h>

int main(int argc, char** argv)
{
    ma_result result;
    ma_engine engine;

    if (argc < 2) {
        printf("No input file.");
        return -1;
    }

    result = ma_engine_init(NULL, &engine);
    if (result != MA_SUCCESS) {
        printf("Failed to initialize audio engine.");
        return -1;
    }

    ma_engine_play_sound(&engine, argv[1], NULL);

    printf("Press Enter to quit...");
    getchar();

    ma_engine_uninit(&engine);

    return 0;
}
```

This example shows how to decode and play a sound using the low level API.

```c
#define MINIAUDIO_IMPLEMENTATION
#include "../miniaudio.h"

#include <stdio.h>

void data_callback(ma_device* pDevice, void* pOutput, const void* pInput, ma_uint32 frameCount)
{
    ma_decoder* pDecoder = (ma_decoder*)pDevice->pUserData;
    if (pDecoder == NULL) {
        return;
    }

    ma_decoder_read_pcm_frames(pDecoder, pOutput, frameCount, NULL);

    (void)pInput;
}

int main(int argc, char** argv)
{
    ma_result result;
    ma_decoder decoder;
    ma_device_config deviceConfig;
    ma_device device;

    if (argc < 2) {
        printf("No input file.\n");
        return -1;
    }

    result = ma_decoder_init_file(argv[1], NULL, &decoder);
    if (result != MA_SUCCESS) {
        return -2;
    }

    deviceConfig = ma_device_config_init(ma_device_type_playback);
    deviceConfig.playback.format   = decoder.outputFormat;
    deviceConfig.playback.channels = decoder.outputChannels;
    deviceConfig.sampleRate        = decoder.outputSampleRate;
    deviceConfig.dataCallback      = data_callback;
    deviceConfig.pUserData         = &decoder;

    if (ma_device_init(NULL, &deviceConfig, &device) != MA_SUCCESS) {
        printf("Failed to open playback device.\n");
        ma_decoder_uninit(&decoder);
        return -3;
    }

    if (ma_device_start(&device) != MA_SUCCESS) {
        printf("Failed to start playback device.\n");
        ma_device_uninit(&device);
        ma_decoder_uninit(&decoder);
        return -4;
    }

    printf("Press Enter to quit...");
    getchar();

    ma_device_uninit(&device);
    ma_decoder_uninit(&decoder);

    return 0;
}
```

More examples can be found in the [examples](examples) folder or online here: https://miniaud.io/docs/examples/



Documentation
=============
Online documentation can be found here: https://miniaud.io/docs/

Documentation can also be found at the top of [miniaudio.h](https://raw.githubusercontent.com/mackron/miniaudio/master/miniaudio.h)
which is always the most up-to-date and authoritive source of information on how to use miniaudio. All other
documentation is generated from this in-code documentation.


Supported Platforms
===================
- Windows
- macOS, iOS
- Linux
- FreeBSD / OpenBSD / NetBSD
- Android
- Raspberry Pi
- Emscripten / HTML5

miniaudio should compile clean on other platforms, but it will not include any support for playback or capture
by default. To support that, you would need to implement a custom backend. You can do this without needing to
modify the miniaudio source code. See the [custom_backend](examples/custom_backend.c) example.


Backends
========
- WASAPI
- DirectSound
- WinMM
- Core Audio (Apple)
- ALSA
- PulseAudio
- JACK
- sndio (OpenBSD)
- audio(4) (NetBSD and OpenBSD)
- OSS (FreeBSD)
- AAudio (Android 8.0+)
- OpenSL|ES (Android only)
- Web Audio (Emscripten)
- Null (Silence)
- Custom


Major Features
==============

Low-Level API
-------------
The low-level API is a lightweight API for doing processing of raw audio data straight to/from the underlying
device. You can connect to multiple devices at once, and can choose specific devices to connect to rather than
being forced to use system defaults. You can do playback, capture and full-duplex. The WASAPI backend can also
do loopback.

miniaudio also exposes backend-specific configuration options for when you need finer control over platform-
specific settings.


High-Level API
--------------
The high-level audio engine in miniaudio encapsulates the resource manager and node graph to give you an easy to
use API to manage sounds. The engine is a node graph, and each sound is a node within that graph which means you
can take advantage of miniaudio's graph based effect processing and routing infrastructure.

Sounds support 3D spatialization and can be pluged into effect nodes if you need to apply an effect. You can
also group sounds for when you need to apply volume control or an effect on a group of sounds.


Advanced Mixing and Effect Processing
-------------------------------------
miniaudio includes a node graph system for doing mixing and effect processing. The output of each node is
connected to an input of another node. When the outputs of multiple nodes are connected to the same input node,
they will be mixed before being processed by the input node.

You can implement custom nodes for doing your own effect processing. miniaudio has some basic nodes built-in, but
there are some additional nodes in the extras folder in the miniaudio repository.


Resource Management
-------------------
The resource manager is used for simplifying the hassle of dealing with the loading and management of your audio
resources. It will reference count files so they're only loaded once and handles streaming of large audio sources
to save on memory. It can even load files asynchronously and exposes it's job system so you can handle resource
management jobs from your existing job infrastructure.


Flexible and Modular API
------------------------
miniaudio have a very flexible and highly decoupled API. The high-level API is built on top of the low-level
API but can even be used without it. For example, if you want to use a different library for handling playback,
such as SDL, you can configure miniaudio to bypass it's low-level API completely so you can plug it into SDL
instead.

This modularity extends to all parts of miniaudio, such as the resource manager, node graph, decoders and more.
You can use these as completely independant units for constructing your audio solution.


And More
--------
- Decoding, with built-in support for WAV, FLAC and MP3, in addition to being able to plug in custom decoders.
- Encoding (WAV only).
- Data conversion.
- Resampling, including custom resamplers.
- Basic generation of waveforms and noise.
- Basic effects and filters.

Refer to the [Programming Manual](https://miniaud.io/docs/manual/) for a more complete description of
available features in miniaudio.


Building
========
Do the following in one source file:
```c
#define MINIAUDIO_IMPLEMENTATION
#include "miniaudio.h"
```
Then just compile. There's no need to install any dependencies. On Windows and macOS there's no need to link
to anything. On Linux just link to -lpthread, -lm and -ldl. On BSD just link to -lpthread and -lm. On iOS you
need to compile as Objective-C.

If you get errors about undefined references to `__sync_val_compare_and_swap_8`, `__atomic_load_8`, etc. you
need to link with `-latomic`.

If you prefer separate .h and .c files, you can find a split version of miniaudio in the extras/miniaudio_split
folder. From here you can use miniaudio as a traditional .c and .h library - just add miniaudio.c to your source
tree like any other source file and include miniaudio.h like a normal header. If you prefer compiling as a
single translation unit (AKA unity builds), you can just #include the .c file in your main source file:
```c
#include "miniaudio.c"
```
Note that the split version is auto-generated using a tool and is based on the main file in the root directory.
If you want to contribute, please make the change in the main file.


License
=======
Your choice of either public domain or [MIT No Attribution](https://github.com/aws/mit-0).
