# FFmpeg Cheat Sheet: Convert TS/MKV to MP4, Lossless Re-encoding & FFprobe Metadata

A practical FFmpeg reference for converting video files without re-encoding (stream copy / lossless remux), extracting metadata with FFprobe, and fixing common issues like Windows Explorer thumbnail generation. Covers `.ts` to `.mp4`, `.mkv` to `.mp4`, moov atom placement, and `-movflags +faststart`.

## Table of Contents

- [Convert TS to MP4 without re-encoding (lossless stream copy)](#convert-ts-to-mp4-without-re-encoding-lossless-stream-copy)
- [Convert MKV to MP4 without re-encoding (lossless stream copy)](#convert-mkv-to-mp4-without-re-encoding-lossless-stream-copy)
- [Extract video metadata as JSON with FFprobe](#extract-video-metadata-as-json-with-ffprobe)
- [Fix MP4 thumbnail not showing in Windows Explorer](#fix-mp4-thumbnail-not-showing-in-windows-explorer-moov-atom--faststart)
- [Common Flags](#common-flags)
- [Table of flags](#table-of-flags)
- [Footnotes](#footnotes)

## Convert TS to MP4 without re-encoding (lossless stream copy)

```bash
ffmpeg -i "input.ts" -map 0 -c copy -movflags +faststart "output.mp4"
```

_with comments:_
```bash
ffmpeg \
  -i "input.ts" \ # Input file
  -map 0 \ # Select all streams (video, audio, subtitles, etc.)
  -c copy \ # Copy! do not re-encode (fast and lossless)
  -movflags +faststart \ # Move the metadata to the beginning of the file for faster streaming.
  "output.mp4" # Output file
```

For Transport Stream (.ts) files the `-map 0` option is handy,
by default ffmpeg select one "best" stream, however since you have probably recorded the video from a web source, it might contain multiple audio/video tracks.

You almost always want to preserve the original video format in it's entirety,
the only thing you care is that your player can play the file, and .ts files can be problematic especially web players.

The `-movflags +faststart` flags are optional but **strongly** recommended, they ensure the MP4 file is optimized for streaming by moving the metadata to the beginning of the file,
which allows for faster playback start times, especially when streaming over the web.

## Convert MKV to MP4 without re-encoding (lossless stream copy)

> You will lose PGS subtitles and other unsupported streams, but the video and audio will be preserved without re-encoding.

```bash
ffmpeg -i "input.mkv" -map 0:v -map 0:a -c copy -movflags +faststart "output.mp4"
```

_with comments:_

```bash
ffmpeg \
  -i "input.mkv" \ # Input file
  -map 0:v \ # Select all video streams
  -map 0:a \ # Select all video audio
  -c copy \ # Copy! do not re-encode (fast and lossless)
  -movflags +faststart \ # Move the metadata to the beginning of the file for faster streaming.
  "output.mp4" # Output file
```

Here we have a problem, MKV files can contain impossible-to-copy streams, such as PGS subtitles, which are image-based and not supported in MP4 containers.

The command above selects only the video and audio streams, ignoring subtitles and other streams that might not be compatible with MP4.
You will need to extract/convert the subtitles separately, it can be a pain I'll help you with that later.

## Extract video metadata as JSON with FFprobe

```bash
ffprobe -hide_banner -loglevel error -v quiet -bitexact -print_format json -print_filename "" -show_format -show_streams -show_chapters -show_programs -show_error -i "input.mkv"
```

_with comments:_

```bash
ffprobe \
  -hide_banner \ # Hide the initial banner information for cleaner output.
  -loglevel error \ # Only show error messages, suppressing warnings and informational messages.
  -v quiet \ # Suppress all output except for the JSON data, useful in scripts where you do not want to handle stderr output.
  -bitexact \ # Make the output consistent across different versions of ffprobe.
  -print_format json \ # Output the metadata in JSON format for easy parsing.
  -print_filename "" \ # Prevent ffprobe from printing the filename in the output.
  -show_format \ # Include format information in the output.
  -show_streams \ # Include detailed information about each stream (video, audio, subtitles, etc.).
  -show_chapters \ # Include chapter information if available.
  -show_programs \ # Include program information if available.
  -show_error \ # Include any errors encountered during probing in the output, much more useful than stderr.
  -i "input.mkv" # Input file
```

This command outputs a detailed JSON containing all useful metadata about the file, including format information, stream details, chapters, programs, and any errors encountered during probing.

The critical flags are:
- `-show_format`: Prints aggregate info about the file, such as duration, overall bitrate. If you omit this you'll need to extract infos from sterams.
- `-show_streams`: Codec, resolution, bitrate, language, etc are all in this flag
- `-show_chapters`: Useful for movies and TV shows, it contains chapter titles and timestamps.
- `-show_programs`: Useful for live recordings even tho most of the time it will be empty.
- `-show_error`: Handy, when you get an error almost always other fields are non-present in the JSON file.

The main unusual flag here is `-print_filename ""`, which prevents FFprobe from printing the filename in the output. This is useful when saving metadata to a file, if the file is moved later you don't want the filename to be outdated and misleading. Additionally, sometimes you do not want to accidentally leak your filesystem in logs or when sharing the metadata.

`-bitexact` is complitely optional, it makes `ffprobe` output consistent across different versions.

> *Unless you are pinning down a specific version of `ffprobe`, I find it better to have this flag on all the time. If you depend on specific output, you can't just update ffprobe or it will break your code/script. `-bitexact` indicates clearly are not relying on a specific `ffprobe` version*

Some additional notes on the `-bitexact` flag, `ffprobe` do not have a built-in way extract mime-types of the streams, instead you need to parse the `codec_name` or `format` field to infer the mime-type.

In those situations you might want to omit `-bitexact` to get the `codec_long_name` field, which is more human readable and contains the full name of the codec, for example `H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10` instead of just `h264`. This extra bit of information is useful yo update your mime-type inference code when a new codec is released.

Other flags and why they are used:

- `-hide_banner`: Hide the initial banner information for cleaner output.
- `-loglevel error`: Only show error messages, suppressing warnings and informational messages.
- `-v quiet`: Suppress all output except for the JSON data, useful in scripts where you do not want to handle stderr output.
- `-print_format json`: Output the metadata in JSON format for easy parsing.
- `-show_format`: Include format information in the output.
- `-show_streams`: Include detailed information about each stream (video, audio, subtitles, etc.).
- `-show_chapters`: Include chapter information if available.
- `-show_programs`: Include program information if available.
- `-show_error`: Include any errors encountered during probing in the output, much more useful than stderr.

### Tip: why `-i` is at the end of the ffprobe command

You may have noticed that unlike every other FFmpeg command, this `ffprobe` command places `-i` **at the very end**. This is **intentional** and **practical**.

When you copy a long command and paste it into a terminal to run it on a different file, you only need to hit backspace a couple of times. With `-i` at the end, a quick `Ctrl+W` or a few `Delete` presses is all you need to replace the filename.

This is the kind of habit that saves you from catasrophics mistakes when running the same command dozens of times across different files.

## Fix MP4 thumbnail not showing in Windows Explorer (moov atom / faststart)

If your MP4 file shows a blank icon in Windows Explorer instead of a video thumbnail, the cause is almost always the `moov atom` (metadata) being located at the end of the file.
Windows Explorer reads only the beginning of the file to generate thumbnails if the metadata isn't there, it gives up.

The fix is `-movflags +faststart`, which relocates the `moov atom` to the beginning of the file:

```bash
ffmpeg -i "input.mkv" -c copy -movflags +faststart "output.mp4"
```

This also fixes thumbnails in:
 - Media players (VLC, MPC-HC) seeking before the file is fully buffered
 - Web browsers playing the file via `<video>` tag
 - Tools that extract cover art or duration without fully downloading the file

It also fixes VLC freezing for 3 second before playing a video.

Why this happens: Recording software usually write metadata at the end of the file since they are writing as new data comes in,
it would be impossible to know beforehand how long the video will be, so once the steam ends or the recording is stopped,
the software writes the metadata at the end of the file, which is perfectly fine for playback but not for thumbnail generation.
`-movflags +faststart` does a second pass to move it.

`-movflags +faststart` is the secret noone tells you. Once you know it, you'll add it to every single MP4 output command by default. It does not hurt even when the metadata is already at the beginning, it just add a check that moves them when needed.

## Common Flags:

`map 0:v -map 0:a`: Select all video and audio streams, ignore subtitles and other streams.

When the MKV contains subtitles, it might not always be possible to copy them to MP4,
for example MP4 does not support PGS subtitles, which are image-based.

Attempting to remux a MKV with PGS subtitles to MP4 will result in an error.

`-c copy`: Copy the streams without re-encoding.

`-movflags +faststart`: Move the metadata to the beginning of the file for faster streaming.
This is particularly useful for web playback, thumbnail generation and metadata extraction.
MP4 files can have their metadata (moov atom) at the end of the file,
the OS needs to read the first few bytes then move to the end of the file, same with Web players.

## Table of flags:

| Flag | Description |
| --- | --- |
| `-i "input.file"` | Specify the input file |
| `-map 0` | Select all streams (video, audio, subtitles, etc.) |
| `-map 0:v` | Select all video streams |
| `-map 0:a` | Select all audio streams |
| `-c copy` | Copy streams without re-encoding (lossless remux) |
| `-movflags +faststart` | Move metadata to the beginning of the file for faster streaming and thumbnail generation |

## Footnotes

This page collects FFmpeg and FFprobe commands I find myself reusing constantly — lossless video remuxing, metadata extraction, and container fixes. More commands (image manipulation, grid thumbnail generation, etc.) will be added over time.

There are some neat commands I use in large-scale production-grade websites infrastructure that I will also include here later on, they are very specific to my use case but since they are very hard to find online, I think they deserve a place here.
