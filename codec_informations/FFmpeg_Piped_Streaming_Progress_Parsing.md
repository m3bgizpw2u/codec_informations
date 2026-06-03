# FFmpeg Piped Streaming and Progress Parsing — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg API reference)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** FFmpeg source code (`ffmpeg_opt.c`, `ffmpeg.h`, `progress.c`)
> **Patent Status:** N/A
> **License:** LGPL 2.1+
> **Current Version:** FFmpeg 7.x (ongoing development)
> **Active Development:** Yes — ongoing

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation

- **Creator(s):** FFmpeg Project
- **Year Created:** FFmpeg piping and progress mechanisms evolved from early 2000s
- **Original Purpose:** Enable programmatic access to FFmpeg audio/video processing, real-time progress tracking, and integration with external tools
- **Problem with Predecessors:** FFmpeg historically wrote all output to stderr, making programmatic integration difficult; the pipe protocol and progress mechanism solve this

### 1.2 Version History

| Version | Year | Key Changes |
|---------|------|-------------|
| FFmpeg 0.6 | 2010 | Basic pipe protocol support |
| FFmpeg 2.0 | 2013 | Improved progress reporting |
| FFmpeg 4.0 | 2018 | `-progress` flag stable, structured output |
| FFmpeg 5.0 | 2022 | Enhanced progress fields |
| FFmpeg 6.0 | 2023 | Better pipe buffering |
| FFmpeg 7.0 | 2024 | Continued improvements |

### 1.3 Current Adoption

- **Primary use cases today:** Audio processing pipelines, batch converters, streaming servers, monitoring systems, CI/CD pipelines
- **Platforms with native support:** All platforms with FFmpeg
- **Major tools using this:** Audio normalization tools, video transcoding services, music library management
- **Status:** Widely adopted for automation and integration

---

## 2. FFMPEG PIPE PROTOCOL

### 2.1 Pipe Protocol Fundamentals

FFmpeg supports reading from and writing to stdin/stdout using the pipe protocol:

```bash
# Read from stdin
ffmpeg -i - [options] output

# Write to stdout
ffmpeg [options] -i input -

# Chain two FFmpeg instances via pipe
ffmpeg -i input.wav -f f32le - | ffmpeg -f f32le -i - output.mp3
```

### 2.2 Pipe Protocol Syntax

```bash
# Basic pipe operations
ffmpeg -i input.wav -f wav -          # Write WAV to stdout
ffmpeg -i - -c:a flac output.flac    # Read from stdin

# Specify format for pipe
ffmpeg -i input.wav -f s16le -       # Raw PCM, signed 16-bit little-endian
ffmpeg -f s16le -i - output.wav     # Read raw PCM from stdin
```

### 2.3 Supported Formats for Piping

| Format | CLI Name | Description | Use Case |
|--------|----------|-------------|----------|
| WAV | `wav` | RIFF WAV with header | Standard audio files |
| Raw PCM | `s16le` | Signed 16-bit little-endian | Direct sample access |
| Raw PCM | `s24le` | Signed 24-bit little-endian | Hi-res audio |
| Raw PCM | `s32le` | Signed 32-bit little-endian | Hi-res audio |
| Raw PCM | `f32le` | 32-bit float little-endian | Float processing |
| AIFF | `aiff` | AIFF with header | macOS compatible |
| FFmpeg format | `ffmpeg` | Auto-detect | Mixed scenarios |

### 2.4 Raw PCM Pipe Format Options

```bash
# Signed 8-bit PCM
ffmpeg -i input.wav -f s8 - output.raw

# Signed 16-bit PCM little-endian (CD quality)
ffmpeg -i input.wav -f s16le - output.raw

# Signed 24-bit PCM little-endian (hi-res)
ffmpeg -i input.wav -f s24le - output.raw

# Signed 32-bit PCM little-endian
ffmpeg -i input.wav -f s32le - output.raw

# 32-bit float little-endian
ffmpeg -i input.wav -f f32le - output.raw

# 64-bit float little-endian
ffmpeg -i input.wav -f f64le - output.raw
```

---

## 3. WAV PIPING

### 3.1 WAV vs Raw PCM Piping

| Aspect | WAV Pipe | Raw PCM Pipe |
|--------|----------|--------------|
| Header | Yes (44 bytes typical) | No |
| Self-contained | Yes | No |
| Metadata | Yes (in header) | No |
| Sample rate | Encoded in header | Must be specified separately |
| Channel count | Encoded in header | Must be specified separately |
| Bit depth | Encoded in header | Must be specified separately |

### 3.2 WAV Pipe Example

```bash
# Read WAV from pipe
ffmpeg -i - -c:a flac output.flac < input.wav

# Write WAV to pipe
ffmpeg -i input.flac -f wav - > output.wav

# Chain with WAV header preservation
ffmpeg -i input.wav -f wav - | ffmpeg -i - -c:a alac output.m4a
```

### 3.3 WAV Header Structure

```
Offset  Size  Field           Description
------  ----  -----           -----------
0x00    4B    "RIFF"         RIFF magic
0x04    4B    File size      Total file size minus 8
0x08    4B    "WAVE"         WAVE format
0x0C    4B    "fmt "         fmt chunk marker
0x10    4B    fmt chunk size  16 for PCM
0x14    2B    Audio format    1 = PCM, 3 = IEEE float
0x16    2B    Channels        1 = mono, 2 = stereo
0x18    4B    Sample rate     44100, 48000, etc.
0x1C    4B    Byte rate       sample_rate × channels × bits/8
0x20    2B    Block align     channels × bits/8
0x22    2B    Bits per sample  8, 16, 24, 32
0x24    4B    "data"          data chunk marker
0x28    4B    Data size       Audio data size
```

---

## 4. PROGRESS OUTPUT

### 4.1 Progress Output Basics

The `-progress` flag enables structured progress reporting:

```bash
# Basic progress output to stdout
ffmpeg -i input.wav -c:a flac output.flac -progress pipe:1

# Progress to file
ffmpeg -i input.wav -c:a flac output.flac -progress progress.txt

# Progress to custom URL
ffmpeg -i input.wav -c:a flac output.flac -progress tcp://localhost:9000
```

### 4.2 Progress Fields

Each progress update contains key-value pairs:

```
frame=1234
fps=25.00
stream_0_0_q=-0.0
bitrate=1280.0kbits/s
total_size=5242880
out_time_ms=50000000
out_time=00:00:50.000000
dup_frames=0
drop_frames=0
speed=2.45x
progress=continue
```

### 4.3 Progress Field Descriptions

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `frame` | integer | Total frames processed | `1234` |
| `fps` | float | Current encoding FPS | `25.00` |
| `stream_*_q` | float | Stream quality (QP) | `-0.0` |
| `bitrate` | string | Current bitrate | `1280.0kbits/s` |
| `total_size` | integer | Total output size in bytes | `5242880` |
| `out_time_ms` | integer | Output time in milliseconds | `50000000` |
| `out_time` | string | Output time as HH:MM:SS.MS | `00:00:50.000000` |
| `dup_frames` | integer | Duplicated frames | `0` |
| `drop_frames` | integer | Dropped frames | `0` |
| `speed` | string | Processing speed | `2.45x` |
| `progress` | string | Progress state | `continue` or `end` |

### 4.4 Progress State Values

| State | Meaning | When Received |
|-------|---------|---------------|
| `continue` | Processing continues | Every progress update (typically every 1 second) |
| `end` | Encoding complete | Final progress update |

### 4.5 Progress Update Frequency

By default, progress updates occur:
- Every 1 second during encoding
- At the end of encoding (with `progress=end`)

---

## 5. PIPED STREAMING EXAMPLES

### 5.1 Simple Audio Conversion via Pipe

```bash
# Convert WAV to FLAC via pipe
ffmpeg -i input.wav -f wav - | ffmpeg -i - -c:a flac output.flac

# Convert to multiple formats via pipe
ffmpeg -i input.wav -f wav - | tee \
    >(ffmpeg -i - -c:a alac output1.m4a) \
    >(ffmpeg -i - -c:a flac output2.flac) \
    >(ffmpeg -i - -c:a libmp3lame -b:a 320k output3.mp3) \
    > /dev/null
```

### 5.2 Cross-Format Pipe Conversion

```bash
# FLAC to MP3 via raw PCM
ffmpeg -i input.flac -f s16le - | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - \
    -c:a libmp3lame -b:a 320k output.mp3

# DSD to FLAC via PCM conversion
ffmpeg -i input.dsf -c:a dsd_msbf -f f32le - | \
    ffmpeg -f f32le -ar 96000 -ac 2 -i - \
    -c:a flac -compression_level 8 output.flac
```

### 5.3 Two-Pass Encoding with Pipes

```bash
# Two-pass MP3 encoding with pipe
# Pass 1: Analyze
ffmpeg -i input.wav -f s16le - 2>/dev/null | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - \
    -pass 1 -c:a libmp3lame -b:a 320k -f mp3 /dev/null

# Pass 2: Encode
ffmpeg -i input.wav -f s16le - 2>/dev/null | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - \
    -pass 2 -c:a libmp3lame -b:a 320k output.mp3
```

### 5.4 Seeking in Pipe Mode

Since pipes are sequential, seeking requires pre-processing:

```bash
# Process specific time range via pipe
ffmpeg -i input.wav -ss 00:01:00 -t 00:01:00 -f s16le - | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - \
    -c:a flac output_section.flac

# Process multiple ranges (requires multiple pipes)
ffmpeg -i input.wav -ss 00:00:00 -t 00:01:00 -f s16le - | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - -c:a copy part1.flac

ffmpeg -i input.wav -ss 00:01:00 -t 00:01:00 -f s16le - | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - -c:a copy part2.flac

# Concatenate
ffmpeg -i "concat:part1.flac|part2.flac" -c:a flac output_combined.flac
```

---

## 6. PROGRESS PARSING IN PYTHON

### 6.1 Basic Progress Parser

```python
import subprocess
import threading
import re
import sys

def parse_ffmpeg_progress(pipe, progress_callback=None):
    """
    Parse FFmpeg progress output from a pipe.
    
    Args:
        pipe: File-like object to read from
        progress_callback: Optional callback(frame, fps, bitrate, total_size, out_time)
    """
    progress = {}
    
    for line in pipe:
        line = line.strip()
        if not line:
            continue
        
        if '=' in line:
            key, value = line.split('=', 1)
            progress[key] = value
            
            if progress_callback:
                # Call callback with current progress
                progress_callback(
                    frame=int(progress.get('frame', 0)),
                    fps=float(progress.get('fps', 0.0)),
                    bitrate=progress.get('bitrate', ''),
                    total_size=int(progress.get('total_size', 0)),
                    out_time=progress.get('out_time', ''),
                    speed=progress.get('speed', ''),
                    progress_state=progress.get('progress', '')
                )
        
        if progress.get('progress') == 'end':
            break
    
    return progress

def run_ffmpeg_with_progress(input_file, output_file, progress_callback=None):
    """
    Run FFmpeg with progress tracking.
    """
    cmd = [
        'ffmpeg', '-i', input_file,
        '-c:a', 'flac', '-compression_level', '8',
        '-progress', 'pipe:1',  # Progress to stdout
        '-y',  # Overwrite output
        output_file
    ]
    
    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Read progress in separate thread
    def read_progress():
        parse_ffmpeg_progress(process.stdout, progress_callback)
    
    progress_thread = threading.Thread(target=read_progress)
    progress_thread.start()
    
    # Wait for FFmpeg to complete
    return_code = process.wait()
    progress_thread.join()
    
    return return_code
```

### 6.2 Advanced Progress Parser with Percentage

```python
import subprocess
import threading
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class FFmpegProgress:
    """Progress information from FFmpeg."""
    frame: int = 0
    fps: float = 0.0
    bitrate: str = ''
    total_size: int = 0
    out_time_ms: int = 0
    out_time: str = ''
    dup_frames: int = 0
    drop_frames: int = 0
    speed: str = ''
    progress_state: str = ''
    
    @property
    def out_time_seconds(self) -> float:
        """Get output time in seconds."""
        return self.out_time_ms / 1_000_000.0

class FFmpegProgressParser:
    """Parser for FFmpeg progress output."""
    
    def __init__(self, total_duration: Optional[float] = None):
        self.total_duration = total_duration
        self.progress = FFmpegProgress()
        self._is_complete = False
    
    def parse_line(self, line: str) -> bool:
        """Parse a single progress line. Returns True if parsing should continue."""
        line = line.strip()
        if not line:
            return True
        
        if '=' not in line:
            return True
        
        key, value = line.split('=', 1)
        
        # Parse based on key type
        if key == 'frame':
            self.progress.frame = int(value)
        elif key == 'fps':
            self.progress.fps = float(value)
        elif key == 'bitrate':
            self.progress.bitrate = value
        elif key == 'total_size':
            self.progress.total_size = int(value)
        elif key == 'out_time_ms':
            self.progress.out_time_ms = int(value)
        elif key == 'out_time':
            self.progress.out_time = value
        elif key == 'dup_frames':
            self.progress.dup_frames = int(value)
        elif key == 'drop_frames':
            self.progress.drop_frames = int(value)
        elif key == 'speed':
            self.progress.speed = value
        elif key == 'progress':
            self.progress.progress_state = value
            if value == 'end':
                self._is_complete = True
                return False
        
        return True
    
    def calculate_percentage(self) -> float:
        """Calculate progress percentage if total duration is known."""
        if self.total_duration and self.total_duration > 0:
            return min(100.0, (self.progress.out_time_seconds / self.total_duration) * 100.0)
        return 0.0
    
    @property
    def is_complete(self) -> bool:
        return self._is_complete

def run_ffmpeg_with_percentage(
    input_file: str,
    output_file: str,
    total_duration: float,
    on_progress: callable
) -> int:
    """Run FFmpeg with percentage progress reporting."""
    
    parser = FFmpegProgressParser(total_duration)
    
    cmd = [
        'ffmpeg', '-i', input_file,
        '-c:a', 'flac', '-compression_level', '8',
        '-progress', 'pipe:1',
        '-y', output_file
    ]
    
    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL
    )
    
    for line in process.stdout:
        if not parser.parse_line(line.decode('utf-8', errors='replace')):
            break
        
        percentage = parser.calculate_percentage()
        on_progress(percentage, parser.progress)
    
    return process.wait()
```

### 6.3 Asynchronous Progress Parser

```python
import asyncio
import subprocess
from typing import Optional, Callable

async def run_ffmpeg_async(
    input_file: str,
    output_file: str,
    progress_callback: Optional[Callable[[float, dict], None]] = None
) -> tuple[int, dict]:
    """Run FFmpeg asynchronously with progress support."""
    
    cmd = [
        'ffmpeg', '-i', input_file,
        '-c:a', 'flac', '-compression_level', '8',
        '-progress', 'pipe:1',
        '-y', output_file
    ]
    
    process = await asyncio.create_subprocess_exec(
        *cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.DEVNULL
    )
    
    progress_data = {}
    
    async def read_progress():
        if process.stdout:
            while True:
                line = await process.stdout.readline()
                if not line:
                    break
                
                line_text = line.decode('utf-8', errors='replace').strip()
                if '=' in line_text:
                    key, value = line_text.split('=', 1)
                    progress_data[key] = value
    
    # Run progress reader concurrently
    progress_task = asyncio.create_task(read_progress())
    
    # Wait for process to complete
    return_code = await process.wait()
    
    # Wait for progress reader to finish
    await progress_task
    
    return return_code, progress_data

# Usage with progress callback
async def main():
    def on_progress(percentage, data):
        print(f"Progress: {percentage:.1f}% - {data.get('frame', 0)} frames")
    
    return_code, progress = await run_ffmpeg_async(
        'input.wav',
        'output.flac',
        progress_callback=on_progress
    )
    
    print(f"FFmpeg exit code: {return_code}")
    print(f"Final progress: {progress}")
```

---

## 7. PROGRESS PARSING IN OTHER LANGUAGES

### 7.1 JavaScript/Node.js Progress Parser

```javascript
const { spawn } = require('child_process');

function runFFmpegWithProgress(inputFile, outputFile, onProgress) {
    return new Promise((resolve, reject) => {
        const ffmpeg = spawn('ffmpeg', [
            '-i', inputFile,
            '-c:a', 'flac', '-compression_level', '8',
            '-progress', 'pipe:1',
            '-y', outputFile
        ]);
        
        let progress = {};
        
        ffmpeg.stdout.on('data', (data) => {
            const lines = data.toString().split('\n');
            
            for (const line of lines) {
                const trimmed = line.trim();
                if (!trimmed) continue;
                
                if (trimmed.includes('=')) {
                    const [key, value] = trimmed.split('=');
                    progress[key] = value;
                    
                    if (onProgress) {
                        onProgress({
                            frame: parseInt(progress.frame || 0),
                            fps: parseFloat(progress.fps || 0),
                            bitrate: progress.bitrate || '',
                            totalSize: parseInt(progress.total_size || 0),
                            outTime: progress.out_time || '',
                            speed: progress.speed || '',
                            progressState: progress.progress || ''
                        });
                    }
                }
                
                if (progress.progress === 'end') {
                    ffmpeg.kill();
                    break;
                }
            }
        });
        
        ffmpeg.stderr.on('data', (data) => {
            // FFmpeg info/warnings go to stderr
            console.error(data.toString());
        });
        
        ffmpeg.on('close', (code) => {
            resolve({ exitCode: code, progress });
        });
        
        ffmpeg.on('error', (err) => {
            reject(err);
        });
    });
}

// Usage
runFFmpegWithProgress('input.wav', 'output.flac', (p) => {
    const percentage = p.totalSize ? 
        Math.round((p.totalSize / expectedSize) * 100) : 0;
    console.log(`Progress: ${percentage}%`);
}).then(result => {
    console.log(`Done with exit code: ${result.exitCode}`);
}).catch(err => {
    console.error('Error:', err);
});
```

### 7.2 C Progress Parser

```c
#include <libavformat/avformat.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int frame;
    float fps;
    int64_t total_size;
    int64_t out_time_ms;
    float speed;
    char progress_state[32];
} FFmpegProgress;

int parse_progress_line(const char *line, FFmpegProgress *prog) {
    char key[64], value[256];
    
    if (sscanf(line, "%63[^=]=%255[^\n]", key, value) == 2) {
        if (strcmp(key, "frame") == 0) {
            prog->frame = atoi(value);
        } else if (strcmp(key, "fps") == 0) {
            prog->fps = atof(value);
        } else if (strcmp(key, "total_size") == 0) {
            prog->total_size = atoll(value);
        } else if (strcmp(key, "out_time_ms") == 0) {
            prog->out_time_ms = atoll(value);
        } else if (strcmp(key, "speed") == 0) {
            // Remove 'x' suffix
            prog->speed = atof(value);
        } else if (strcmp(key, "progress") == 0) {
            strncpy(prog->progress_state, value, sizeof(prog->progress_state) - 1);
        }
        return 1;
    }
    return 0;
}

int run_ffmpeg_with_progress(const char *input, const char *output,
                            void (*progress_cb)(FFmpegProgress*)) {
    AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
    AVPacket *pkt = NULL;
    int ret;
    FFmpegProgress progress = {0};
    FILE *progress_file = NULL;
    
    // Open input
    ret = avformat_open_input(&ifmt_ctx, input, NULL, NULL);
    if (ret < 0) return ret;
    
    ret = avformat_find_stream_info(ifmt_ctx, NULL);
    if (ret < 0) goto end;
    
    // Create output context
    avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, output);
    if (!ofmt_ctx) { ret = AVERROR(ENOMEM); goto end; }
    
    // Setup output stream (simplified - add proper encoding)
    AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
    
    // Open output file
    if (!(ofmt_ctx->oformat->flags & AVFMT_NOFILE)) {
        ret = avio_open(&ofmt_ctx->pb, output, AVIO_FLAG_WRITE);
        if (ret < 0) goto end;
    }
    
    // Open progress file (pipe:1 equivalent)
    progress_file = popen("cat", "r");  // Simplified for example
    
    avformat_write_header(ofmt_ctx, NULL);
    
    // Read/encode loop
    pkt = av_packet_alloc();
    while (av_read_frame(ifmt_ctx, pkt) >= 0) {
        // Simplified processing
        
        // Call progress callback
        if (progress_cb) {
            progress_cb(&progress);
        }
        
        av_packet_unref(pkt);
    }
    
    av_write_trailer(ofmt_ctx);
    
end:
    if (progress_file) pclose(progress_file);
    av_packet_free(&pkt);
    avformat_close_input(&ifmt_ctx);
    if (ofmt_ctx && !(ofmt_ctx->oformat->flags & AVFMT_NOFILE))
        avio_closep(&ofmt_ctx->pb);
    avformat_free_context(ofmt_ctx);
    
    return ret < 0 ? ret : 0;
}
```

### 7.3 Bash Progress Monitor

```bash
#!/bin/bash

# FFmpeg progress monitor script
# Usage: monitor_progress.sh <pid>

PID=$1
PROGRESS_FILE="/tmp/ffmpeg_progress_$$.log"

# Cleanup on exit
trap "rm -f $PROGRESS_FILE" EXIT

# Monitor loop
while kill -0 $PID 2>/dev/null; do
    if [ -f "$PROGRESS_FILE" ]; then
        FRAME=$(grep "^frame=" "$PROGRESS_FILE" 2>/dev/null | tail -1 | cut -d= -f2)
        FPS=$(grep "^fps=" "$PROGRESS_FILE" 2>/dev/null | tail -1 | cut -d= -f2)
        SPEED=$(grep "^speed=" "$PROGRESS_FILE" 2>/dev/null | tail -1 | cut -d= -f2)
        TIME=$(grep "^out_time=" "$PROGRESS_FILE" 2>/dev/null | tail -1 | cut -d= -f2)
        
        echo -ne "Frame: $FRAME | FPS: $FPS | Speed: $SPEED | Time: $TIME\033[0K\r"
    fi
    sleep 1
done

echo ""  # Newline after progress line
```

---

## 8. FFMPEG EXIT CODES

### 8.1 Exit Code Reference

| Code | Name | Meaning | Common Cause |
|------|------|---------|--------------|
| 0 | Success | Operation completed successfully | Normal completion |
| 1 | General Error | Generic error occurred | Invalid input, unsupported codec |
| 2 | — | Reserved | — |
| ... | Various | Specific errors | Format-specific issues |
| 255 | Interrupted | Process was interrupted | Ctrl+C, signal received |

### 8.2 Handling Exit Codes in Scripts

```bash
#!/bin/bash

# Run FFmpeg and check exit code
ffmpeg -i input.wav -c:a flac output.flac

EXIT_CODE=$?

case $EXIT_CODE in
    0)
        echo "Success: Output created"
        ;;
    1)
        echo "Error: FFmpeg encountered an error"
        # Check stderr for details
        ;;
    255)
        echo "Interrupted: Process was cancelled"
        # Clean up partial output
        ;;
    *)
        echo "Unknown exit code: $EXIT_CODE"
        ;;
esac
```

### 8.3 Using -xerror for Strict Mode

```bash
# Exit on first error
ffmpeg -i input.wav -c:a flac output.flac -xerror

# In a pipeline - fail if any FFmpeg fails
set -o pipefail
ffmpeg -i input.wav -f s16le - 2>/dev/null | \
    ffmpeg -f s16le -i - -c:a flac output.flac -xerror
```

---

## 9. CANCELING PIPED CONVERSIONS

### 9.1 Signal Handling

```python
import subprocess
import signal
import os

class FFmpegProcess:
    """FFmpeg process with cancellation support."""
    
    def __init__(self):
        self.process = None
        self.should_cancel = False
    
    def run(self, input_file, output_file):
        cmd = [
            'ffmpeg', '-i', input_file,
            '-c:a', 'flac', '-compression_level', '8',
            '-progress', 'pipe:1',
            '-y', output_file
        ]
        
        self.process = subprocess.Popen(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE
        )
        
        try:
            return_code = self.process.wait()
            return return_code
        except KeyboardInterrupt:
            self.cancel()
            return -1
    
    def cancel(self):
        """Cancel the running FFmpeg process."""
        if self.process and self.process.poll() is None:
            self.process.terminate()
            try:
                self.process.wait(timeout=5)
            except subprocess.TimeoutExpired:
                self.process.kill()
                self.process.wait()

# Usage
ffmpeg_proc = FFmpegProcess()

# In a signal handler
def signal_handler(signum, frame):
    print("Cancellation requested...")
    ffmpeg_proc.cancel()

signal.signal(signal.SIGINT, signal_handler)
signal.signal(signal.SIGTERM, signal_handler)

return_code = ffmpeg_proc.run('input.wav', 'output.flac')
```

### 9.2 Graceful Cancellation

```bash
#!/bin/bash

# Graceful FFmpeg cancellation script
PID=0

# Cleanup function
cleanup() {
    if [ $PID -gt 0 ]; then
        echo "Cleaning up..."
        kill -TERM $PID 2>/dev/null
        wait $PID 2>/dev/null
        
        # Remove partial output
        if [ -f "output.flac" ]; then
            rm -f "output.flac"
        fi
    fi
    exit 130  # Standard exit code for SIGINT
}

trap cleanup EXIT INT TERM

# Run FFmpeg in background
ffmpeg -i input.wav -c:a flac output.flac -progress pipe:1 &
PID=$!

# Monitor progress
while kill -0 $PID 2>/dev/null; do
    sleep 1
done

wait $PID
EXIT_CODE=$?

echo "FFmpeg exited with code: $EXIT_CODE"
```

---

## 10. MEMORY CONSIDERATIONS

### 10.1 Buffer Sizing

```bash
# Increase buffer size for large files
ffmpeg -i input.wav -f s16le -buffer_size 10M - | \
    ffmpeg -f s16le -buffer_size 10M -i - -c:a flac output.flac

# Use larger packets for better throughput
ffmpeg -i input.wav -f s16le -packet_size 65536 - | \
    ffmpeg -f s16le -packet_size 65536 -i - -c:a flac output.flac
```

### 10.2 Memory-Efficient Piping

```python
import subprocess

def pipe_conversion_efficient(input_file, output_file, chunk_size=65536):
    """
    Memory-efficient audio conversion via pipe.
    """
    # Reader process
    reader = subprocess.Popen(
        ['ffmpeg', '-i', input_file, '-f', 's16le', '-'],
        stdout=subprocess.PIPE,
        stderr=subprocess.DEVNULL
    )
    
    # Writer process
    writer = subprocess.Popen(
        ['ffmpeg', '-f', 's16le', '-ar', '44100', '-ac', '2', '-i', '-',
         '-c:a', 'flac', '-compression_level', '8', output_file],
        stdin=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Stream data in chunks
    while True:
        chunk = reader.stdout.read(chunk_size)
        if not chunk:
            break
        writer.stdin.write(chunk)
    
    # Close writer's stdin to signal EOF
    writer.stdin.close()
    reader.stdout.close()
    
    # Wait for processes
    reader.wait()
    writer.wait()
    
    return reader.returncode, writer.returncode
```

---

## 11. CODEC TRANSCODING VIA PIPES

### 11.1 When to Use Pipes vs Direct Transcoding

| Aspect | Pipe Mode | Direct Mode |
|--------|-----------|-------------|
| Memory usage | Lower (streaming) | Higher (full decode) |
| Seeking | Limited | Full support |
| Complexity | Higher | Lower |
| Flexibility | Higher (insert processing) | Lower |
| Speed | Similar | Similar |
| Metadata handling | Manual | Automatic |

### 11.2 Pipe vs Direct: Decision Guide

```bash
# Use DIRECT transcode when:
# - Input format is supported by both encoder and decoder
# - No intermediate processing needed
# - Want automatic metadata handling
ffmpeg -i input.flac -c:a alac output.m4a

# Use PIPE when:
# - Need custom processing between decode/encode
# - Processing in chunks
# - Integrating with external tools
ffmpeg -i input.flac -f s16le - | \
    some_processor | \
    ffmpeg -f s16le -i - -c:a flac output.flac
```

### 11.3 Inserting Processing in Pipe Chain

```bash
# Add normalization between decode and encode
ffmpeg -i input.wav -f s16le - | \
    ffmpeg -f s16le -i - -af "loudnorm=I=-16" -c:a flac output.flac

# Add filtering in the chain
ffmpeg -i input.wav -f f32le - | \
    ffmpeg -f f32le -ar 48000 -ac 2 -i - \
    -af "highpass=f=80,lowpass=f=16000" \
    -c:a alac output.m4a

# Multiple processing steps
ffmpeg -i input.wav -f s16le - | \
    tee \
        >(ffmpeg -f s16le -i - -af "volume=2.0" -c:a pcm_s16le backup.raw) \
        | ffmpeg -f s16le -i - -c:a flac output.flac
```

---

## 12. FAST PIPE MODE

### 12.1 Optimizing Pipe Throughput

```bash
# Use larger buffer for faster throughput
ffmpeg -i input.wav -f s16le -buffer_size 20M - | \
    ffmpeg -f s16le -buffer_size 20M -i - -c:a flac output.flac

# Use faster codec for intermediate format
ffmpeg -i input.flac -f f32le - | \
    ffmpeg -f f32le -i - -c:a flac output.flac

# Avoid unnecessary processing
ffmpeg -i input.wav -f s16le - | \
    ffmpeg -f s16le -i - -c:a pcm_s16le - | \
    ffmpeg -f s16le -i - -c:a flac output.flac
```

### 12.2 Parallel Pipe Processing

```bash
#!/bin/bash

# Parallel conversion of multiple files via pipes

INPUTS=("file1.wav" "file2.wav" "file3.wav")
OUTPUTS=("out1.flac" "out2.flac" "out3.flac")

# Start all conversions in parallel
PIDS=()
for i in "${!INPUTS[@]}"; do
    ffmpeg -i "${INPUTS[$i]}" -f s16le - \
        | ffmpeg -f s16le -i - -c:a flac -compression_level 8 \
            "${OUTPUTS[$i]}" &
    PIDS+=($!)
done

# Wait for all to complete
FAILED=0
for pid in "${PIDS[@]}"; do
    wait $pid || ((FAILED++))
done

echo "Completed with $FAILED failures"
```

---

## 13. REAL-WORLD EXAMPLES

### 13.1 Audio Normalization Pipeline

```python
#!/usr/bin/env python3
"""
Audio normalization pipeline with FFmpeg pipe and progress.
"""

import subprocess
import sys
import re

def run_normalization(input_file, output_file, target_loudness=-16.0):
    """Normalize audio using loudnorm filter via pipe."""
    
    # First pass: analyze
    print("Analyzing input...")
    cmd_analyze = [
        'ffmpeg', '-i', input_file,
        '-af', f'loudnorm=I={target_loudness}:print_format=json',
        '-f', 'null', '-'
    ]
    
    result = subprocess.run(
        cmd_analyze,
        capture_output=True,
        text=True
    )
    
    # Parse measured values from stderr
    measured = {}
    for line in result.stderr.split('\n'):
        if 'Input Integrated' in line:
            match = re.search(r'(-?\d+\.?\d*) LUFS', line)
            if match:
                measured['input_i'] = float(match.group(1))
        elif 'Input True Peak' in line:
            match = re.search(r'(-?\d+\.?\d*) dBFS', line)
            if match:
                measured['input_tp'] = float(match.group(1))
        elif 'Input LRA' in line:
            match = re.search(r'(-?\d+\.?\d*) LU', line)
            if match:
                measured['input_lra'] = float(match.group(1))
    
    if not measured:
        print("Error: Could not analyze input", file=sys.stderr)
        return 1
    
    print(f"Measured: {measured}")
    
    # Second pass: apply normalization
    print("Normalizing...")
    filter_str = (
        f"loudnorm="
        f"I={target_loudness}:"
        f"TP={measured.get('input_tp', -1.0):.1f}:"
        f"LRA={measured.get('input_lra', 0.0):.1f}"
    )
    
    cmd_encode = [
        'ffmpeg', '-i', input_file,
        '-af', filter_str,
        '-c:a', 'flac', '-compression_level', '8',
        '-progress', 'pipe:1',
        '-y', output_file
    ]
    
    process = subprocess.Popen(cmd_encode, stdout=subprocess.PIPE, text=True)
    
    for line in process.stdout:
        line = line.strip()
        if '=' in line:
            key, value = line.split('=', 1)
            if key == 'out_time':
                print(f"Progress: {value}", end='\r')
    
    print()  # Newline after progress
    return process.wait()

if __name__ == '__main__':
    sys.exit(run_normalization(sys.argv[1], sys.argv[2]))
```

### 13.2 Batch Conversion with Progress

```python
#!/usr/bin/env python3
"""
Batch audio conversion with overall progress tracking.
"""

import subprocess
import os
from pathlib import Path

class BatchConverter:
    """Batch audio converter with progress reporting."""
    
    def __init__(self, input_dir, output_dir, output_format='flac'):
        self.input_dir = Path(input_dir)
        self.output_dir = Path(output_dir)
        self.output_format = output_format
        self.converted = 0
        self.failed = 0
        
        self.output_dir.mkdir(parents=True, exist_ok=True)
    
    def get_audio_files(self):
        """Get list of audio files to convert."""
        extensions = {'.wav', '.mp3', '.aac', '.ogg', '.flac', '.m4a'}
        return list(self.input_dir.glob('*.*'))
    
    def convert_file(self, input_file):
        """Convert a single file."""
        output_file = self.output_dir / f'{input_file.stem}.{self.output_format}'
        
        cmd = [
            'ffmpeg', '-i', str(input_file),
            '-c:a', self.output_format,
            '-compression_level', '8',
            '-y', str(output_file)
        ]
        
        result = subprocess.run(cmd, capture_output=True)
        return result.returncode == 0
    
    def run(self):
        """Run batch conversion."""
        files = self.get_audio_files()
        total = len(files)
        
        print(f"Converting {total} files...")
        
        for i, input_file in enumerate(files, 1):
            print(f"[{i}/{total}] {input_file.name}... ", end='', flush=True)
            
            if self.convert_file(input_file):
                print("OK")
                self.converted += 1
            else:
                print("FAILED")
                self.failed += 1
        
        print(f"\nComplete: {self.converted} succeeded, {self.failed} failed")
        return self.failed == 0

if __name__ == '__main__':
    import sys
    converter = BatchConverter(sys.argv[1], sys.argv[2])
    success = converter.run()
    exit(0 if success else 1)
```

---

## 14. ERROR HANDLING IN PIPES

### 14.1 Detecting Pipe Errors

```python
import subprocess

def check_pipe_health(process, stderr_captured):
    """Check if pipe has errors."""
    
    # Check stderr for errors
    error_patterns = [
        'Error',
        'Invalid',
        'failed',
        'cannot',
        'does not support'
    ]
    
    stderr_lower = stderr_captured.lower()
    for pattern in error_patterns:
        if pattern.lower() in stderr_lower:
            return False, f"Found error pattern: {pattern}"
    
    # Check exit code
    if process.returncode != 0:
        return False, f"Non-zero exit code: {process.returncode}"
    
    return True, "OK"

def run_safe_pipe_conversion(input_file, output_file):
    """Run conversion with error checking."""
    
    # Reader
    reader = subprocess.Popen(
        ['ffmpeg', '-i', input_file, '-f', 's16le', '-'],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Writer
    writer = subprocess.Popen(
        ['ffmpeg', '-f', 's16le', '-i', '-', '-c:a', 'flac', output_file],
        stdin=reader.stdout,
        stderr=subprocess.PIPE
    )
    
    # Close reader's stdout to signal EOF to writer
    reader.stdout.close()
    
    # Wait and capture
    writer_stderr = writer.communicate()
    reader.wait()
    
    # Check for errors
    writer_ok, writer_msg = check_pipe_health(writer, writer_stderr[0].decode())
    reader_ok, reader_msg = check_pipe_health(reader, writer_stderr[1].decode())
    
    if not writer_ok:
        print(f"Writer error: {writer_msg}")
    if not reader_ok:
        print(f"Reader error: {reader_msg}")
    
    return writer_ok and reader_ok
```

### 14.2 Timeout Handling

```python
import subprocess
import signal
import time

class TimeoutError(Exception):
    pass

def run_with_timeout(cmd, timeout_seconds):
    """Run FFmpeg with timeout."""
    
    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    start_time = time.time()
    
    try:
        while process.poll() is None:
            elapsed = time.time() - start_time
            if elapsed > timeout_seconds:
                process.kill()
                process.wait()
                raise TimeoutError(f"Process exceeded {timeout_seconds} seconds")
            time.sleep(0.1)
        
        return process.returncode
    
    finally:
        if process.poll() is None:
            process.kill()
            process.wait()
```

---

## 15. COMMAND REFERENCE

### 15.1 Basic Pipe Commands

```bash
# Read from pipe
ffmpeg -i - output.wav

# Write to pipe
ffmpeg -i input.wav -f wav -

# Pipe with format specification
ffmpeg -i input.wav -f s16le - | ffmpeg -f s16le -i - output.mp3

# Suppress stderr for cleaner piping
ffmpeg -i input.wav -f wav - 2>/dev/null | ffmpeg -i - -c:a flac output.flac
```

### 15.2 Progress Commands

```bash
# Progress to stdout
ffmpeg -i input.wav -c:a flac output.flac -progress pipe:1

# Progress to file
ffmpeg -i input.wav -c:a flac output.flac -progress progress.log

# Progress with timestamp
ffmpeg -i input.wav -c:a flac output.flac \
    -progress "progress_$(date +%Y%m%d_%H%M%S).log"

# Progress with quiet mode
ffmpeg -i input.wav -c:a flac output.flac \
    -progress pipe:1 \
    -loglevel error
```

### 15.3 Advanced Pipe Commands

```bash
# Chain multiple conversions
ffmpeg -i input.wav -f wav - | \
    ffmpeg -i - -af "volume=2.0" -f wav - | \
    ffmpeg -i - -c:a alac output.m4a

# Tee for multiple outputs
ffmpeg -i input.wav -f wav - | tee \
    >(ffmpeg -i - -c:a flac out1.flac) \
    >(ffmpeg -i - -c:a alac out2.m4a) \
    >/dev/null

# Process specific range via pipe
ffmpeg -i input.wav -ss 00:01:00 -t 00:01:00 -f s16le - | \
    ffmpeg -f s16le -ar 44100 -ac 2 -i - -c:a flac range.flac
```

---

## 16. IMPLEMENTATION CHECKLIST

### Basic Pipe Setup
- [ ] Verify FFmpeg supports pipe protocol (`ffmpeg -h` works)
- [ ] Test basic pipe operation
- [ ] Choose appropriate format (WAV vs raw PCM)
- [ ] Set correct sample format for raw PCM

### Progress Parsing
- [ ] Parse progress output in real-time
- [ ] Calculate percentage if duration is known
- [ ] Handle `progress=end` state
- [ ] Handle incomplete progress data

### Error Handling
- [ ] Check exit codes
- [ ] Parse stderr for error messages
- [ ] Implement timeout handling
- [ ] Clean up partial output on failure

### Performance
- [ ] Size buffers appropriately
- [ ] Use threading for progress reading
- [ ] Consider async I/O for high throughput
- [ ] Monitor memory usage

### Testing
- [ ] Test with small files first
- [ ] Test error scenarios
- [ ] Test cancellation
- [ ] Test timeout scenarios

---

## 17. QUICK REFERENCE

### 17.1 Common Pipe Formats

```bash
# Signed 16-bit PCM
-f s16le

# Signed 24-bit PCM
-f s24le

# Signed 32-bit PCM
-f s32le

# 32-bit float PCM
-f f32le

# WAV format
-f wav
```

### 17.2 Progress Fields Quick Reference

```bash
frame=1234       # Frame count
fps=25.00        # Frames per second
bitrate=1280k    # Current bitrate
total_size=5242880  # Total output size in bytes
out_time=00:01:23.456789  # Output time
speed=2.5x      # Processing speed
progress=continue  # State (continue/end)
```

### 17.3 Exit Codes

```bash
0   # Success
1   # Error
255 # Interrupted
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
