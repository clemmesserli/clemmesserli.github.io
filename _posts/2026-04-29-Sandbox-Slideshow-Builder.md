---
layout: post
title: "Vibe Series: Sandbox Slideshow Builder — Part I"
date: 2026-04-29 10:00:00 +0000
categories: [PowerShell, Automation]
tags: [vibe-coding, windows-sandbox, imagemagick, ffmpeg, scripting, automation, video]
---

### Introduction

Ever wanted to turn a plain text file full of quotes—or a folder of images—into a polished MP4 slideshow, without installing a single tool on your host machine? That's exactly what the Sandbox Slideshow Builder does.

This project uses Windows Sandbox as a fully isolated execution environment. Every dependency (ImageMagick, FFmpeg, VC++ Runtime, 7-Zip) is downloaded and installed inside the sandbox at runtime. When the sandbox closes, it's gone. Your host stays clean.

In Part I we'll walk through the full pipeline: how it's structured, how to run it, and what's happening under the hood.

### How It Works

```
source assets → ImageMagick (compose/overlay frames) → PNG frames → FFmpeg → MP4
```

There are four files that make up the entire framework:

| File | Role |
|---|---|
| `Start-Sandbox.ps1` | Host-side launcher — preps the share folder and fires up the sandbox |
| `sandbox.wsb` | Windows Sandbox config template — maps the share folder and sets the logon command |
| `Invoke-SlideshowBuilder.ps1` | Runs inside the sandbox — installs tools, generates frames, encodes video |
| `input.txt` | Your content — one quote or caption per line |

### Prerequisites

Before running anything, confirm you have:

- **Windows 10/11 Pro, Enterprise, or Education** — Sandbox is not available on Home editions
- **Windows Sandbox feature enabled** — `Start-Sandbox.ps1` will detect if it's missing and offer to enable it (requires a reboot)
- **Virtualization enabled in BIOS/UEFI**
- **PowerShell 5.1+**
- **Internet access** — tools are downloaded on first run inside the sandbox

### Step 1: Prepare Your Input

Create a plain text file with one entry per line. Each line becomes a slide:

```
The only way to do great work is to love what you do.
In the middle of every difficulty lies opportunity.
It always seems impossible until it's done.
```

Save it somewhere accessible, e.g. `C:\Temp\quotes.txt`.

### Step 2: Run Start-Sandbox.ps1

From a PowerShell prompt in the project folder:

```powershell
.\Start-Sandbox.ps1 -SharePath "C:\SlideshowShare" -InputPath "C:\Temp\quotes.txt"
```

What this does:

1. Checks your Windows edition and confirms Sandbox is enabled
2. Creates `C:\SlideshowShare` if it doesn't exist
3. Copies `Invoke-SlideshowBuilder.ps1` and `sandbox.wsb` into the share folder
4. Copies your text file as `input.txt` in the share folder
5. Patches `sandbox.wsb` with the real share path (written to a temp file — your original is untouched)
6. Launches the sandbox

```powershell
param(
    [Parameter(Mandatory)]
    [string]$SharePath,

    [Parameter(Mandatory)]
    [string]$InputPath
)
```

The `-SharePath` and `-InputPath` parameters are both mandatory — the script won't run without them.

### Step 3: The Sandbox Takes Over

Once the sandbox boots, `Invoke-SlideshowBuilder.ps1` runs automatically via the `LogonCommand` in `sandbox.wsb`:

```xml
<LogonCommand>
  <Command>powershell.exe -ExecutionPolicy Bypass -File C:\Shared\Invoke-SlideshowBuilder.ps1</Command>
</LogonCommand>
```

> **Why `-ExecutionPolicy Bypass`?** Windows Sandbox starts with a default execution policy of `Restricted`, which blocks all scripts from running. The `-ExecutionPolicy Bypass` flag overrides this for the duration of that single process — it doesn't change any system-wide policy and has no effect on your host machine. Without it, `Invoke-SlideshowBuilder.ps1` would be silently blocked and nothing would happen after the sandbox boots.

The script mounts your share as `C:\Shared` (read/write) and installs tools into `C:\Tools`. Here's the install sequence:

1. **VC++ Runtime** — required by ImageMagick
2. **7-Zip** (`7zr.exe`) — used to extract the ImageMagick `.7z` archive
3. **ImageMagick** (portable `.7z`) — frame composition engine
4. **FFmpeg** (`.zip`) — video encoder

All downloads include retry logic and skip re-downloading if the file already exists — useful if you re-run inside the same sandbox session.

### Step 4: Frame Generation

With tools ready, the script reads `input.txt` and generates one PNG per line using ImageMagick:

```powershell
$width  = 1280
$height = 720

$magickArgs = @(
    '-size', "${width}x${height}",
    'gradient:#1a1a2e-#16213e',
    '-fill', '#c9a84c',
    '-draw', "line 80,$($height - 60) $($width - 80),$($height - 60)",
    '(',
        '-background', 'none',
        '-fill', 'white',
        '-font', $fontPath,
        '-pointsize', '42',
        '-size', "$($width - 160)x$($height - 160)",
        '-gravity', 'center',
        "caption:@$tmpTxt",
    ')',
    '-gravity', 'center',
    '-composite',
    $outFile
)
```

Breaking down what ImageMagick is doing here:

- `gradient:#1a1a2e-#16213e` — dark navy-to-midnight blue background gradient
- `'-draw', "line ..."` — draws a gold accent line near the bottom
- `caption:@$tmpTxt` — auto-wraps the quote text to fit the canvas
- `-composite` — overlays the text layer onto the background

Each frame is generated as a parallel background job, so a 50-quote file doesn't take 50x longer than a single frame.

### Step 5: Video Encoding

Once all PNGs exist, FFmpeg stitches them into an MP4:

```powershell
$ffmpegArgs = @(
    '-y',
    '-framerate', '0.33',
    '-i', "$imgDir\frame_%03d.png",
    '-c:v', 'libx264',
    '-pix_fmt', 'yuv420p',
    '-r', '30',
    $videoOut
)
```

Key settings:

- `-framerate 0.33` — each frame displays for ~3 seconds
- `-c:v libx264` — H.264 encoding, broadly compatible
- `-pix_fmt yuv420p` — ensures compatibility with most players and platforms
- `-r 30` — output at 30fps

### Step 6: Output

When complete, Explorer opens automatically to your share folder. You'll find:

| File | Description |
|---|---|
| `slideshow.mp4` | Your finished video |
| `images\frame_NNN.png` | Individual PNG frames |
| `sandbox_log_YYYYMMDD_HHmmss.txt` | Transcript of the sandbox run — one file per run |

The MP4 is written directly to your host share folder — no manual copying needed.

### Customization

The most common tweaks are all at the top of `Invoke-SlideshowBuilder.ps1`:

| Setting | Variable / Flag | Default |
|---|---|---|
| Resolution | `$width` / `$height` | `1280` / `720` |
| Slide duration | `-framerate` | `0.33` (~3s per slide) |
| Font size | `-pointsize` | `42` |
| Background colors | `gradient:#1a1a2e-#16213e` | Dark navy gradient |
| Accent line color | `-fill '#c9a84c'` | Gold |

For example, to produce a 1920x1080 video with 5 seconds per slide:

```powershell
$width  = 1920
$height = 1080
# ...
'-framerate', '0.2',
```

### Logging and Troubleshooting

Every sandbox run produces a timestamped `sandbox_log_YYYYMMDD_HHmmss.txt` in your share folder. Each run gets its own file, so if you need to re-run you won't lose the previous transcript. It captures the full PowerShell transcript including download progress, job output, and any errors. If something goes wrong, that's the first place to look.

The `Write-Step` helper wraps each major phase and reports elapsed time:

```
[START] Generate PNGs
[DONE]  Generate PNGs (14.3s)
[START] Generate Video
[DONE]  Generate Video (2.1s)
=== COMPLETE === Total elapsed: 187.4s
```

### Common Troubleshooting Tips

#### Windows Sandbox Not Found

If you see an error like `'WindowsSandbox' is not recognized` or the sandbox feature is missing entirely, `Start-Sandbox.ps1` will detect this and prompt you:

```
Windows Sandbox is not enabled. Enable it now? A reboot will be required. (y/n)
```

Enter `y` and reboot. If the option never appears, confirm your Windows edition — Sandbox is not available on Home editions. You can check with:

```powershell
(Get-WindowsEdition -Online).Edition
```

#### VC++ DLL Not Found

ImageMagick depends on the Visual C++ Runtime. If you see errors referencing missing `.dll` files (e.g. `VCRUNTIME140.dll`) when frames are being generated, the VC++ install likely failed or didn't complete in time. Check `sandbox_log.txt` for the VC++ install exit code. A non-zero exit code usually means the installer was blocked or timed out — re-running the sandbox from scratch typically resolves it.

#### Only One Instance of Sandbox Can Run at a Time

Windows Sandbox only supports a single running instance. If you try to launch a second sandbox while one is already open, nothing will happen — no error, no new window. Close the existing sandbox first, then re-run `Start-Sandbox.ps1`.

#### Script Cannot Be Loaded — Running Scripts Is Disabled

If you see this on your **host machine** when running `Start-Sandbox.ps1`:

```
Start-Sandbox.ps1 cannot be loaded because running scripts is disabled on this system.
```

Your host execution policy is blocking it. Run this once in an elevated PowerShell prompt to allow local scripts:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

This allows locally written scripts to run while still requiring downloaded scripts to be signed. Inside the sandbox this is handled automatically via `-ExecutionPolicy Bypass` in the `LogonCommand`.

#### Re-Running the Script Inside the Sandbox Without Rebuilding

If the script fails partway through and you want to retry without closing and relaunching the full sandbox (which would re-download all tools), you can open a PowerShell window directly inside the sandbox and re-run manually:

1. Inside the sandbox, click **Start**, type `powershell`, and open **Windows PowerShell**
2. Run the script directly:

```powershell
powershell.exe -ExecutionPolicy Bypass -File C:\Shared\Invoke-SlideshowBuilder.ps1
```

Because all downloads are skipped when files already exist in `C:\Tools`, only the failed step will re-execute. Check `C:\Shared\sandbox_log_YYYYMMDD_HHmmss.txt` afterward for the updated transcript.

### Conclusion

The Sandbox Slideshow Builder is a clean example of using Windows Sandbox as a disposable CI-style environment for media processing. No Docker, no VMs, no leftover installs — just a self-contained PowerShell pipeline that runs, produces output, and disappears.

In Part II, we'll look at extending the image pipeline to support a folder of photos as input, adding overlay text and branding to existing images rather than generating frames from scratch.

### Next Steps

- Clone the repo and try it with your own `quotes.txt`
- Adjust the gradient colors and font size to match your style
- Check `sandbox_log.txt` to understand the full timing breakdown
- Stay tuned for Part II — the image folder pipeline

---

*This post is part of my Vibe Series — AI-assisted projects built with the help of coding tools like Amazon Q, GitHub Copilot, and Claude, with a focus on clean automation, isolation, and repeatability.*
