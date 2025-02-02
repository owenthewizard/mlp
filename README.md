# mlp

[![ci](https://github.com/domyd/mlp/workflows/ci/badge.svg?branch=master)](https://github.com/domyd/mlp/actions?query=workflow%3Aci)

A Dolby TrueHD demuxer and utility tool, with a focus on accurately and correctly demuxing a TrueHD stream from a decrypted blu-ray disc.

Dual-licensed under MIT and Apache 2.0.

## Install

You can download the latest binaries for **macOS** and **Windows** from the [Releases tab](https://github.com/domyd/mlp/releases) of this repository. Requires at least Windows 7 or Mac OS X 10.10.

## Usage

Demux the TrueHD stream from a given blu-ray playlist file:

```powershell
mlp demux playlist "F:\BDMV\PLAYLIST\00800.mpls" --output "out.thd"
```

Print the available TrueHD streams, the segment map, and any available angles for the given playlist file:

```powershell
mlp demux playlist "F:\BDMV\PLAYLIST\00800.mpls"
```

Demux the TrueHD stream from a list of stream files in the `F:\BDMV\STREAM` directory, and save it to `out.thd`. The files are chosen based either a comma-separated list of numbers or `+`-separated list of file names:

```powershell
mlp demux segments -s "F:\BDMV\STREAM" -o "out.thd" -l "55,56"
mlp demux segments -s "F:\BDMV\STREAM" -o "out.thd" --segment-files "00055.m2ts+00056.m2ts"
```

Show frame count and duration information of a TrueHD stream:

```powershell
mlp info "out.thd"
mlp info "00055.m2ts"
```

### Additional arguments

If your blu-ray has multiple angles, you must select one with `--angle <index>`. The given `<index>` starts at 1. This only applies to the `demux playlist` command.

```powershell
mlp demux playlist "F:\BDMV\PLAYLIST\00800.mpls" --output "out.thd" --angle 2
```

If the selected container format contains multiple TrueHD streams, you must select one with `--stream <index>`. A list of streams is printed at the start, or you can use `ffprobe <m2ts-file>` or `mlp demux playlist <playlist-file>` to see the available streams and their indices. This applies to all commands.

```powershell
mlp demux playlist "F:\BDMV\PLAYLIST\00800.mpls" --output "out.thd" --stream 1
mlp demux segments -s "F:\BDMV\STREAM" -o "out.thd" -l "55,56" --stream 2
mlp info "00055.m2ts" --stream 3
```

Every command supports `-v` or `-vv` for more verbose output.

## FAQ

### Aren't there already other demuxing tools out there?

Absolutely. However, all of them fail in [different](https://www.makemkv.com/forum/viewtopic.php?f=6&t=21513&p=84453#p84453) [ways](http://rationalqm.us/board/viewtopic.php?p=10841#p10841) on TrueHD streams, especially on discs that contain a large number of segments, which has resulted in desync and noticeable audio artifacts. This tool aims to be a perfectly accurate TrueHD demuxer that doesn't produce invalid, broken, or out-of-sync streams.

### What does this tool do differently?

I'm glad you asked! For that, we'll need to inspect the generated TrueHD streams on the level of individual audio samples. The following image is a screenshot from Audacity:

![](docs/explained.png)

This is the TrueHD audio of *Monsters University*, downmixed to one channel for better illustration. We're looking at the boundary between segments 55 and 56.

* Track 1 is the audio demuxed with this tool.
* Track 2 is the (end of the) unmodified TrueHD stream of 00055.m2ts.
* Track 3 is the (start of the) unmodified TrueHD stream of 00056.m2ts.
* Track 4 is the audio demuxed with MakeMKV 1.15.1.

You might already be able to see the problem - the audio *frames* at either ends of the original TrueHD streams are identical! (A TrueHD frame is always 1/1200 seconds, or just under 1ms, long. That's 40 samples in this case. It's the smallest unit of audio we can add or delete.)

It turns out that blu-ray TrueHD tracks can have an *overlapping frame* at the segment boundary.  If we just naïvely append two tracks together, like MakeMKV<sup>[1]</sup> does here, we'll accumulate desync at each boundary and we run the risk of introducing a "pop" caused by the discontinuity of the audio signal<sup>[2]</sup>. Now, nobody can hear 1ms of desync. But *Monsters University* is special in that it consists of **135** segments, or 134 segment boundaries. 129 of those contain duplicate audio frames, which means that by the end of the movie your audio would be **107 ms** late! That's noticeable.

We also know that we can cut the very last TrueHD frame off a stream without consequences. The reason for that is that the start of every stream always has what's called a *major sync*. Major syncs are basically restart points for the decoder, where any issues introduced by previous audio data are no longer relevant.

You might very well say: *"Hold on, maybe that's intended by the film studio? That audio is there on the original blu-ray files, after all."* And that's a very good point. So let's see how long the video is, then. The video stream of 00055.m2ts contains 3,587 frames. At 23.976 fps (really 24,000/1,001), that's precisely 149.607792 seconds. The TrueHD track for that segment contains 179,530 frames, which results in a duration of 149.608333 seconds. That's an audio *overrun* of 26 samples. *The audio stream is longer than the video stream.*

This is the reality with every single UHD blu-ray I've analyzed so far. That includes Disney/Pixar blu-rays, where this issue is particularly pronounced due to their extensive use of seamless branching, as well as *Alita: Battle Angel*.

The good news is, there's a simple solution: Deleting those duplicated audio frames<sup>[3]</sup>. It turns out that if you do that, the resulting demuxed TrueHD stream is exactly the same length as the demuxed video stream (down to within a few audio samples). The audio ends up free of any discontinuities and the A/V sync tracks perfectly.

That's what this tool does in a nutshell. The TL;DR is that it gets rid of duplicated audio frames and as such automatically maintains perfect sync and gets rid of any audio artifacts like popping or cracking.

[1]: MakeMKV 1.15.1 does eventually maintain A/V sync, but it does so by deleting larger groups of frames on fewer occasions. This isn't perfect, but it's good enough. DGDemux deletes a minor frame at *every* segment boundary, which is better still, but not perfect yet 🙂  
[2]: In this particular section of the screenshot you can actually clearly hear a small pop in MakeMKV's track.  
[3]: It's important to *not* delete the frame that contains the major sync, but to delete the *minor frame* duplicate at the end of a stream. Also, the frames don't actually match *exactly*, so you'll need to use some tolerance when comparing them.

## Build

Tested and supported on macOS and Windows.

You'll need to have the Rust programming language installed, as well as the [Git LFS](https://git-lfs.github.com/) extension.

### Windows

Requires Windows 10 version 1803.

```powershell
$env:CFLAGS="-I$(Get-Location)\external\ffmpeg\include"
cargo build
```

### macOS
```sh
export CFLAGS="-I$PWD/external/ffmpeg/include"
cargo build
```

> NOTE: Downloads the ffmpeg 4.2.2 LGPL binaries and library files from the internet during the build phase.

## TODO list

- [ ] `analyze --fix` command for existing streams
- [ ] Allow removing dialog normalization
- [ ] See if we can get rid of the `End of stream indicated.` ffmpeg message when decoding
- [ ] Better console/log output
- [ ] Support Linux and macOS
- [ ] Performance optimization
- [ ] More tests
- [ ] Support Linux

## Special Thanks

* *Rocky* over at the [DGDemux forum](http://rationalqm.us/board/viewforum.php?f=16) for figuring out where and how to cut TrueHD bitstreams without causing audio issues or decoder errors.
