# Whisper/Qwen3-ASR Bot - Telegram voice transcription bot

This is a Telegram bot that transcribes voice messages using either [whisper.cpp](https://github.com/ggml-org/whisper.cpp) or [qwen-asr](https://github.com/antirez/qwen-asr). You send a voice message, it replies with the text. It's built on top of botlib and follows the same philosophy: simple C code, one thread per request, minimal dependencies. **We suggest using the Qwen3_ASR model** and the linked Pure C inference implementation. Inference is much faster, and the quality very good.

The bot handles downloading audio, converting it to WAV, running the selected backend as an external process, and streaming the result back to Telegram as it's transcribed.

## How it works

The bot uses botlib's thread-per-request model, but with a twist: ASR is CPU-heavy, so running many transcriptions in parallel usually hurts latency/throughput on small servers. Threads wait their turn using a mutex. The queue length is tracked with a C11 atomic, and if too many requests pile up, the bot tells users to retry later.

There's also a small optimization: when the queue is short, it uses the better model (`MODEL_BEST`). When the queue gets longer, it switches to the faster model (`MODEL_FAST`) to clear backlog faster.

The transcription is streamed back to Telegram by editing the message as new text arrives. If the transcription is very long, it automatically continues in a new message (never tested in practice, so far...).

## Dependencies

* libcurl and libsqlite3 (for botlib)
* ffmpeg (for audio conversion)
* one ASR backend binary:
  * `whisper.cpp` (`whisper-cli`), or
  * `qwen_asr` from antirez

## Installation

1. Build both ASR backends (or at least the one you want to use), and download their models.

2. Edit `whisperbot.c` and select exactly one backend:
```c
// #define USE_WHISPER
#define USE_QWEN3_ASR
```

Then set backend-specific paths in the matching block:

```c
#define ASR_BIN_PATH "..."
#define MODEL_FAST "..."
#define MODEL_BEST "..."
```

3. Create your bot with [@BotFather](https://t.me/botfather) and save the API key in `apikey.txt`.

4. Build and run:
```
make
./whisperbot
```

Use `--verbose` to see what's happening, `--debug` for even more output.

## Configuration

Everything is at the top of `whisperbot.c`:

```c
#define MAX_QUEUE 10            // Max pending requests before rejecting
#define MAX_SECONDS 300         // Max audio duration (5 minutes)
#define MSG_LIMIT 4000          // Telegram message length limit
#define TIMEOUT 600             // Kill ASR process after 10 minutes
#define QUEUE_THRESHOLD_FAST 3  // Use MODEL_FAST when queue >= this
```

Whisper-only knobs in the `USE_WHISPER` block:

```c
#define SHORT_AUDIO_THRESHOLD 1.5
#define DEFAULT_LANG "it"
```

`SHORT_AUDIO_THRESHOLD` and `DEFAULT_LANG` are used only by Whisper.

## Limitations

* Backend and model paths are hardcoded (edit and recompile).
* No persistence: if you restart the bot, queued requests are lost.
