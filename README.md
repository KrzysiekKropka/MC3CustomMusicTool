# MC3 Custom Music Tool

`MC3CustomMusicTool` adds custom music to the decoded files of *Midnight Club 3: DUB Edition* and can rebuild the game's `ASSETS.DAT` and `STREAMS.DAT` archives.

The main entry point is the interactive [`tool.py`](tool.py) script. It is designed for a decoded MC3 asset tree, but it can also decode the two DAT archives first when they are placed beside the script.

## Features

- Decode `ASSETS.DAT` and `STREAMS.DAT` into `ASSETS/` and `STREAMS/`.
- Discover music in either `STREAMS/Music/` or `ASSETS/audio/STREAMS/Music/`.
- Convert supported audio files to MC3 `.rsm` streams.
- Normalize non-WAV input to stereo, 44.1 kHz, signed 16-bit PCM before RSTM conversion.
- Add song title and artist information to the four supported string tables:
  `mcstrings01`, `mcstrings02`, `mcstrings04`, and `mcstrings08`.
- Add songs to the correct genre race playlist and to `sd.play`.
- Keep songs in the same relative interval in `sd.play` as their genre race playlist when possible.
- Keep instrumental songs in the garage playlists instead of adding them to `sd.play`.
- Find stale `music\\...` entries in every `.play` file and remove references whose `.rsm`/`.rstm` stream is missing.
- Repair each changed playlist's `num_songs:` count.
- Create timestamped backups of existing decoded `ASSETS/` and `STREAMS/` folders before changes.
- Rebuild the decoded folders into `ASSETS.DAT` and `STREAMS.DAT`.

The tool does not extract an ISO, rebuild an ISO, or handle `VIDEOS/`. Those steps must be performed separately with an ISO/archive tool.

## Requirements

- Python 3.9 or newer for the main workflow.
- Python 3.11 or newer for the bundled `dave.py` archive builder and therefore for the complete decode/edit/rebuild workflow.
- FFmpeg:
  - Windows: the bundled `external_tools/ffmpeg.exe` is used.
  - Linux/macOS: an `ffmpeg` executable must be available on `PATH`.
- A WAV-to-PS-ADPCM converter for `external_tools/rstm_build.py`:
  `external_tools/ps2str.exe` is preferred when present; otherwise `MFAudio.exe` is used.

Run commands from the repository root. The main script uses absolute paths based on its own location, but `rstm_build.py` locates its converter paths relative to the current working directory.

On non-Windows systems, the script displays a permission warning before processing audio. Follow the prompt if you choose to continue without elevated permissions.

## Repository layout

Before running the tool, the repository should contain either DAT archives or decoded game data like this:

```text
MC3CustomMusicTool/
├── tool.py
├── ASSETS.DAT                         # optional if decoded ASSETS/ is present
├── STREAMS.DAT                        # optional if decoded STREAMS/ is present
├── ASSETS/
│   ├── audio/STREAMS/                 # alternate stream location, optional
│   ├── fonts/
│   │   ├── mcstrings01.strtbl
│   │   ├── mcstrings02.strtbl
│   │   ├── mcstrings04.strtbl
│   │   └── mcstrings08.strtbl
│   └── tune/audio/playlist/
├── STREAMS/
│   └── Music/                          # standard stream/music location, optional
└── external_tools/
```

At least one usable music input folder, `ASSETS/tune/audio/playlist/`, and `ASSETS/fonts/` must exist before the editing workflow can continue. The decoded asset folders must be complete enough for the archive rebuild if you choose to create DAT files afterward.

## Quick start

1. Extract `ASSETS.DAT` and `STREAMS.DAT` from the game, or prepare already-decoded `ASSETS/` and `STREAMS/` folders.

2. Put each new audio file in a genre folder under one of these locations:

   ```text
   STREAMS/Music/<Genre>/
   ASSETS/audio/STREAMS/Music/<Genre>/
   ```

   If both locations exist, both are scanned. Use the standard genre folder names when you want the built-in playlist mapping:

   | Folder | Race playlist |
   | --- | --- |
   | `HipHop` | `rap_race_music.play` |
   | `Rock` | `pop_race_music.play` |
   | `Dancehall` | `dance_hall_race_music.play` |
   | `Techno` | `techno_race_music.play` |
   | `Drum_N_Bass` | `drums_bass_race_music.play` |
   | `Instrumental` | `garage.play` in the city and regional garage playlists |

   Other genre names are accepted. They create or update `<lowercase-genre>_race_music.play`.

3. Name integrated songs using:

   ```text
   Artist - Song.ext
   ```

   Supported input extensions are `.aac`, `.aif`, `.aiff`, `.alac`, `.flac`, `.m4a`, `.mp3`, `.ogg`, `.opus`, `.wav`, and `.wma`. Existing `.rsm` and `.rstm` files are not treated as input audio.

4. Start the tool from this directory:

   ```bash
   python tool.py
   ```

5. Follow the prompts. The current script uses exact readiness phrases:

   - When it asks whether you are ready to add songs, enter `REAL BIG`.
   - If songs are found, confirm the list with `DICK REAL BIG`.
   - If no `Artist - Song.ext` files are found, continuing requires `DICK REAL SMALL`.

   These phrases are case-insensitive because the script lowercases the response.

6. When asked whether to encode the archives, answer `Y` only after checking that both decoded asset trees are complete. Otherwise answer `N` and rebuild the DAT files later with the helper commands below.

## What the main script does

### 1. Optional DAT extraction

If `ASSETS.DAT` or `STREAMS.DAT` is present, the script first asks whether to decode them. It uses:

- `dave.py` for `ASSETS.DAT` → `ASSETS/`.
- `hash_build.py` for `STREAMS.DAT` → `STREAMS/`, using the bundled `external_tools/STREAMS.LST` name list, the `MClub` hash algorithm, and a 45% fallback matching threshold.

After extraction it asks whether to continue with music processing.

### 2. String-table preparation

For each of `mcstrings01`, `mcstrings02`, `mcstrings04`, and `mcstrings08`, an existing `.strtbl` is decoded to a temporary `.json` file. The original `.strtbl` is kept while the JSON is edited. The JSON data is loaded, updated with new song entries, and encoded back to `.strtbl` at the end; the helper may ask for confirmation before overwriting the existing `.strtbl`. The temporary JSON files are removed after encoding.

New entries contain six language variants with the game's existing font template. The generated display text follows this pattern:

```text
"Song"
by Artist
```

The localized prefixes currently generated by the code are `by`, `de`, `par`, `von`, `di`, and `by` for languages 00–05.

### 3. Song names and metadata

For a file such as:

```text
Artist - Song (feat. Guest).mp3
```

the tool uses `Artist_Song` as the stream identifier and writes `Song` / `Artist feat. Guest` to the string tables. `(feat. ...)` and `(ft. ...)` are removed from the title and appended to the displayed artist.

New metadata is converted to ASCII, whitespace is normalized, punctuation is removed from the stream identifier, and the combined `Artist_Song` identifier is limited to 31 characters. A name that becomes empty after ASCII cleanup is skipped. Files without ` - ` can still be converted by the RSTM builder, but they are not listed as integrated songs and do not receive playlist or string-table entries.

### 4. RSM generation

For non-WAV files, the script invokes FFmpeg with these output settings:

```text
2 channels, 44100 Hz, signed 16-bit PCM WAV
```

It then invokes `external_tools/rstm_build.py`, which writes an `.rsm` beside the input. Temporary WAV files and successfully converted source files are removed afterward. Existing WAV input is also renamed to the generated identifier and removed after the RSM is built, so keep the timestamped backup if you need to recover the original input.

### 5. Playlist updates

- Every newly processed song is inserted at a random position in its genre race playlist.
- Non-instrumental songs are added to `ASSETS/tune/audio/playlist/city/sd/music/sd.play`.
- The `sd.play` insertion attempts to use the same neighboring songs as the genre race playlist. If no matching interval exists, the song is inserted randomly.
- Instrumental songs are added to the city `garage.play` and the Tokyo, Detroit, and Atlanta garage playlists, but not to `sd.play`.
- Playlist files are rewritten with a correct `num_songs: N` header.

Before editing playlists, the tool scans every `.play` below `ASSETS/tune/audio/playlist`. References beginning with `music\\` are matched against available `.rsm` and `.rstm` files case-insensitively, with either slash style and with or without the stream extension. Missing references are removed automatically. Cleanup is skipped when no stream folder or no stream files are found, preventing an empty/incomplete stream tree from deleting valid playlist entries.

### 6. Optional DAT rebuild

If confirmed, the script creates:

- `ASSETS.DAT` from `ASSETS/` using the DAVE builder with compact alignment, compressed names, and safe file compression.
- `STREAMS.DAT` from `STREAMS/` or, when appropriate, `ASSETS/audio/STREAMS/` using the MC3 hash algorithm.

The archive helper asks before overwriting an existing output file. Replace the corresponding DAT files in the extracted game directory and rebuild the ISO separately.

## Backups and safety

Before changing existing decoded folders, the tool copies each existing `ASSETS/` and `STREAMS/` directory to:

```text
backups/YYYYMMDD_HHMMSS/ASSETS/
backups/YYYYMMDD_HHMMSS/STREAMS/
```

If a timestamp already exists, a numeric suffix is added. Only folders that exist are copied; DAT files themselves are not backed up by this step. The `backups/` directory is ignored by Git.

RSM generation is completed before stale playlist references are removed or playlists are rewritten. If conversion fails, the command stops and the playlist edits are not applied.

## Bundled helper tools

All helper commands below should be run from the repository root.

### `dave.py`: DAVE/Dave archive extraction and building

Extract an archive:

```bash
python external_tools/dave.py X ASSETS.DAT -o ASSETS
```

Build an archive:

```bash
python external_tools/dave.py B ASSETS ASSETS.DAT \
  -ca -cn -cf -fc 1
```

Useful build options include `-cl 1..9` for compression level, `-d` for directory entries, and `-a N` for 16-byte alignment. `-fc 1` forces compression for files considered safe by the tool; `-fc 2` forces all files and may be unsafe for MC3 resources.

### `hash_build.py`: MC3 hash archive extraction and building

Extract and resolve names with the bundled list:

```bash
python external_tools/hash_build.py X STREAMS.DAT \
  -o STREAMS -nl external_tools/STREAMS.LST -a MClub -th 45
```

Build a stream archive:

```bash
python external_tools/hash_build.py B STREAMS STREAMS.DAT -a MClub
```

Unmatched extracted files are placed under `__hashed/`. `-a` accepts `MClub` or `Bully`; `-be` builds big-endian output for platforms that require it. `-th` controls the percentage required before a name list is accepted during extraction.

### `strtbl.py`: string-table conversion

Decode a string table to JSON:

```bash
python external_tools/strtbl.py dec ASSETS/fonts/mcstrings02.strtbl
```

Encode the resulting JSON back to a string table:

```bash
python external_tools/strtbl.py enc ASSETS/fonts/mcstrings02.json
```

The helper prompts before overwriting an existing output.

### `rstm_build.py`: standalone RSTM/RSM conversion

Convert a WAV or ADS/SS2 file:

```bash
python external_tools/rstm_build.py input.wav -o output.rsm
```

The default output is the input path with its extension replaced by `.rsm`. Loop options are available for standalone use:

```bash
python external_tools/rstm_build.py input.wav -o output.rsm -lf
python external_tools/rstm_build.py input.wav -o output.rsm -ls 100 -le 10000
```

Loop positions are specified in frames of 28 samples. For WAV input, `ps2str.exe` is preferred over `MFAudio.exe` when both configured files exist.

### Bundled executables and support files

- `ffmpeg.exe` is used by `tool.py` on Windows for audio normalization.
- `ps2str.exe` and `MFAudio.exe` provide the WAV-to-PS-ADPCM conversion used by `rstm_build.py`.
- `ps2strw.exe` and `encvag.dll` are included in the bundle but are not invoked directly by the current Python workflow.

## Troubleshooting

### “No usable music/assets” or the script exits immediately

Check that the following exist relative to the repository root:

- `STREAMS/Music/` or `ASSETS/audio/STREAMS/Music/`
- `ASSETS/tune/audio/playlist/city/sd/music/`
- `ASSETS/fonts/`

### A song is not detected

Use a supported extension and the exact `Artist - Song.ext` naming pattern. Hidden files, `.rsm`, and `.rstm` files are ignored. Non-ASCII characters are stripped from new metadata; names that become empty are skipped.

### RSM conversion cannot start

Confirm that FFmpeg is installed or that `external_tools/ffmpeg.exe` exists on Windows. For WAV-to-RSM conversion, confirm that `external_tools/ps2str.exe` or `external_tools/MFAudio.exe` exists and that the command is being run from the repository root.

### Playlist entries were not cleaned

Cleanup intentionally does nothing when the stream tree is absent or contains no `.rsm`/`.rstm` files. Restore or decode a usable stream inventory and run the tool again.

### DAT rebuilding fails

Make sure the decoded folders are complete, filenames are ASCII-compatible for the archive formats, and the required Python version is being used. Check the backup before retrying if a previous run stopped partway through.

## Credits and licenses

The Python archive and conversion helpers are based on tools by [Edness](https://ednessp.github.io/). The repository also includes MFAudio by [MuzzleFlash](http://muzzleflash.da.ru/) and FFmpeg binaries from the [FFmpeg project](https://ffmpeg.org/). Review the upstream licenses and redistribution terms before publishing a package containing the bundled binaries.
