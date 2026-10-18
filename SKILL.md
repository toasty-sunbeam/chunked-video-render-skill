---
name: "chunked-video-render"
description: "Set up or run a resumable, multi-core chunked renderer for a code-drawn video (Python frame(t) + ffmpeg) that only re-renders the pieces that changed."
---

# Chunked, resumable video rendering

Use this for any video whose frames are drawn by code: a Python function `frame(t) -> PIL.Image` piped into ffmpeg (the Complaint Department music video is the reference project). It turns "render the whole thing" into: cut the timeline into fixed ~10 s pieces, render them in a worker pool with a progress bar per worker, keep finished pieces, and on later runs redo only the pieces listed as changed.

## When to use
- Setting up rendering for a new code-drawn video project, or porting one to the user's own PC (Windows included).
- After changing a project like this: tell the user which seconds changed (via `changed.txt`) so their re-render only redoes those pieces.

## Requirements the project must meet
1. **A pure frame function.** `frame(t)` must depend only on `t`, never on which frames were drawn before (seeded randomness and caches are fine). The optional spot-check re-renders frames out of order in a fresh process and compares hashes, so test this once: render a 2 s stretch with fingerprints, then run the checker on it and expect `OK`.
2. **A piece entry point** that renders `[t0, t1)` to a silent mp4: `python <timeline>.py T0 T1 out.mp4 video`.
3. **The encoder hook** (below) inside that entry point.
4. **Audio muxed once at the end.** Never per piece; AAC priming would leave tiny gaps at the joins.

## Encoder hook (inside the piece renderer)
**Draw whole frame numbers, `k / FPS`, never `t0 + i / FPS`.** The launcher passes `t0` as text rounded to 6 decimals, so `t0 + i/FPS` lands a hair off the exact frame time. Anything that floors `t` (`random.Random(int(t * 12))` seeds, `t >= threshold` switches, blink/flicker phases) can then flip on some frames, and the spot-check (which renders `k / FPS`) flags those pieces on every run even though nothing changed. The reference project lost an afternoon to this.

```python
def render_video(frame_fn, t0, t1, out):
    import os, subprocess, hashlib, json
    n = int(round((t1 - t0) * FPS))
    cmd = ['ffmpeg', '-y', '-loglevel', 'error', '-f', 'rawvideo', '-pix_fmt', 'rgb24', '-s', f'{W}x{H}', '-r', str(FPS), '-i', '-',
           '-c:v', 'libx264', '-preset', 'medium', '-crf', '18', '-pix_fmt', 'yuv420p', out]
    if os.environ.get('RENDER_FFMPEG_THREADS'):                  # the launcher caps encoder threads per worker
        cmd[-1:-1] = ['-threads', os.environ['RENDER_FFMPEG_THREADS']]
    every = int(os.environ.get('RENDER_SAMPLE_EVERY', '0'))      # fingerprints of every Nth frame
    prog = os.environ.get('RENDER_PROGRESS')                     # launcher's progress file for this piece
    samples = {}
    p = subprocess.Popen(cmd, stdin=subprocess.PIPE)
    try:
        k0 = int(round(t0 * FPS))
        for i in range(n):
            k = k0 + i                                            # global frame number
            b = frame_fn(k / FPS).convert('RGB').tobytes()        # exact frame time (see above)
            if every and k % every == 0: samples[k] = hashlib.md5(b).hexdigest()
            p.stdin.write(b)
            if prog and (i % 5 == 4 or i == n - 1):
                try: open(prog, 'w').write(f'{i + 1} {n}')
                except OSError: pass
        p.stdin.close()
    except OSError:                                               # ffmpeg died: report its exit code below
        pass
    rc = p.wait()
    if rc != 0:
        raise SystemExit(f'ffmpeg stopped with exit code {rc} (0x{rc & 0xffffffff:08X})')
    if every: json.dump(samples, open(out + '.json', 'w'))
```

## The launcher (`render_full.py`, next to the timeline script)
Set `END`, `FPS`, the timeline script name and the song path, then:
- Cut into `n = round(END / PIECE_S)` pieces on **whole-frame** boundaries: `cuts = [round(total * i / n) for i in range(n + 1)]`. Keep `PIECE_S` fixed (10 s); changing it (or `END`) moves every boundary and invalidates every kept piece. To add a tail after the song, append an extra piece rather than changing `END`.
- **Workers:** default `cpu_count // 2`, with `RENDER_FFMPEG_THREADS = cpu_count // workers`. Uncapped x264 in many processes at once can make encoders die silently and bogs down the machine. The reference project's first Windows run lost 3 of 11 pieces that way.
- **Environment for children:** `PYTHONUTF8=1` (Windows otherwise reads UTF-8 lyric/SRT files as cp1252), `RENDER_SAMPLE_EVERY=10`, and per render job `RENDER_PROGRESS=<piece log>.progress`.
- **State in `_parts/`:** `partNNN.mp4`, `.mp4.json` (fingerprints), `.done` (written by the launcher only when the piece exits 0), `.log` (the child's stdout and stderr).
- **Flags:** `--all` (redo everything), `--check` (also spot-check kept pieces), `--autostart` (open the finished video in the default player: `os.startfile` on Windows, `open` on macOS, `xdg-open` on Linux, wrapped in try so it never fails the render), an optional worker count as the first plain argument.
- **Deciding what to redo, in order:**
  1. `--all` deletes every `.done`.
  2. `changed.txt` (next to the script; `start-end` seconds per line, `#` comments allowed) deletes `.done` for every piece overlapping a range. The file is removed after a successful run. Print the piece names it marks (`to redo (changed.txt or unfinished): part011, part012`).
  3. **Only with `--check`:** spot-check every remaining `.done` piece that has a `.mp4.json`. Split those pieces across the workers, and in each worker run `render_full.py --check-worker i,j,k`. That imports the timeline, re-renders the fingerprinted frames (`k / FPS`), and prints `OK i` or `BAD i`. Anything not `OK` gets redone; print which pieces the check flagged. The check is off by default: the user found it slow and confusing when it flagged pieces nobody had touched, and `changed.txt` already covers deliberate changes.
- **Render** the remaining pieces in a pool of `workers`, drawing a progress block: a header line (`rendering: 2/6 done, 41 s`) and one bar per running piece (`part024 [######......] 180/304 frames`), read from each piece's `.progress` file and redrawn every second with ANSI cursor-up (`\x1b[{n}F`) and clear-line (`\x1b[2K`). Call `os.system('')` once on Windows to enable those codes in the console. Delete each `.progress` file when its piece finishes. Retry failures up to 2 times. If any still fail, exit naming their `.log` files and say that running again retries only those.
- **Join:** `ffmpeg -f concat -safe 0 -i _parts/list.txt -i song.m4a -map 0:v -map 1:a -c:v copy -c:a aac -b:a 192k -shortest -movflags +faststart out.mp4`. Tell the user to keep `_parts` for faster re-renders.

## Workflow when changing the project for the user
1. Make the change. Find the exact changed frames by diffing frame hashes of the old and new code over the affected window; render the old version from a backup copy of the module.
2. **Crash-check before shipping.** Previews only exercise the stretch you looked at; a change to shared code (e.g. a new argument to a card/insert helper) can break a scene you never previewed. Render `frame(t)` every 0.1 s over the whole timeline from the copy you are about to ship (a few minutes) and fix any exception first. In the reference project a renamed keyword argument shadowed a local animation variable and only two pieces, far from the edit, failed on the user's PC.
3. **Is the user rendering right now?** Before overwriting their files, look at their `_parts/` modification times and whether `changed.txt` still exists. A missing `changed.txt` with fresh `.done` files means they rendered the last change; a present one means it is still pending (or running). If they have not rendered the previous change yet, write `changed.txt` as the **union** of the old and new ranges, never just the new ones. Replacing `.py` files mid-run is safe for workers already started, but say so.
4. Write `changed.txt` with the ranges, padded slightly (e.g. `20.2-23.2`).
5. Deliver the changed source/asset files plus `changed.txt` to the user's project folder. If the session is linked to their computer, use the device tools; otherwise send the files and say where each one goes. After writing, hash the files on the device and compare with yours. If a re-sent file does not change on the device, stage it from a fresh folder name; a reused staged path can deliver the earlier copy. (PNG files may be re-encoded in transit; compare their decoded pixels, not file hashes.)
6. Tell them to run `python render_full.py`, and about how many pieces will re-render.

## Porting to the user's PC (Windows notes)
- They need `pip install pillow numpy` and ffmpeg (`winget install Gyan.FFmpeg`, then a new terminal).
- Make all paths relative to the project folder (derive the root from `__file__`); no absolute workspace paths. Keep a packaged mirror of the project and patch it by replacing whole function blocks, so its path differences from the working copy survive. Grep the package for absolute workspace paths before each delivery.
- Convert `.woff2` fonts to `.ttf` with fontTools (`TTFont(src); ft.flavor = None; ft.save(dst)`), because Pillow on Windows may lack WOFF2 support. Verify the package by rendering frames from the packaged copy and diffing them pixel-for-pixel against the originals.
- A Python audit hook (`sys.addaudithook`, logging 'open' events) around a full render lists every data file the render reads, which makes it easy to build a minimal package. Fonts load in C, so the hook misses them; add them by hand.
- The device shell on a linked PC can be a small sandbox VM rather than the PC itself. Check `nproc`/`free` before assuming rendering there uses the PC's full power.

## Testing the launcher before shipping
Monkeypatch `END = 4.0; PIECE_S = 1` in a scratch copy and run it:
1. A full run: all pieces render, the progress bars fill, and the output has `END * FPS` frames.
2. A second run with `--check`: everything is checked and unchanged; nothing renders.
3. Corrupt one hash in a `.mp4.json` and add a `changed.txt` range, then run with `--check`: exactly those two pieces re-render, both are named in the output, and `changed.txt` is removed afterwards.
4. Render one piece through the launcher and spot-check it in a fresh process: every sampled frame must match. A mismatch usually means the renderer is not drawing `k / FPS`.