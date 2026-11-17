# About EA Jelly

 EA Jelly website is a Jellyfin server with movies, music, tv shows for personal use that's usually not on Netflix PH. Not for distribution or resell.

<a name="available-apps"></a>
 ## Apps for EA Jelly

 EA Jelly can be accessed through the address http://eajelly.xyz on any browsers that supports HTML5 and Javascript. Watching or listening through the website can generally work but Jellyfin has community driven third party app to make viewing experience better. For example, some browser don't support playback content by default, or even if your hardware does support it, the browser can sometimes fall back to using CPU to play content that can consume more power and heat. The apps below are proven to be simple set and forget. Note that official Jellyfin app exists on iOS and Android. They can work but the app recommended below supports more feature.
> [!TIP]
> These apps have server configuration baked in. To be able to change server use the following switch `--server-override` before running the app.
 - For Windows or MacOS or Debian based system the app contains features that works well with EA Jelly server. You just need an account to access.
   - Jellium Desktop (formerly Jellyfin Desktop): [Multiple OS download](https://github.com/eaforlife/jellium-desktop-eajelly#jellium-desktop---eajelly)
   - EAJelly Music Player by Feishin: [Multiple OS download](https://github.com/eaforlife/jellium-desktop-eajelly#jellium-desktop---eajelly)
 - For iOS you would need additional cofiguration before using the app <sup>[1](#Configure-apps-to-connect-to-EA-Jelly)</sup>
   - Streamify for iOS: [Link to App Store](https://apps.apple.com/app/streamyfin/id6593660679)
   - Manet music player for iOS: [Link to App Store](https://apps.apple.com/us/app/manet-music/id6470928235)
 - For Android you would need additional cofiguration before using the app <sup>[1](#Configure-apps-to-connect-to-EA-Jelly)</sup>
   - Streamify for Android: [Link to Play Store](https://play.google.com/store/apps/details?id=com.fredrikburmester.streamyfin)
   - Jellify music player for Android: [Link to Play Store](https://play.google.com/store/apps/details?id=com.cosmonautical.jellify)

 <a name="setup-apps"></a>
 ## Configure apps to connect to EA Jelly

   Below are setting that is required to use Jellyfin apps to EA Jelly server

> [!TIP]
> Some apps require HTTPS protocol by default. EA Jelly server supports both HTTP and HTTPS. If the URL below doesn't work switch URL to `https://eajelly.xyz`
   
```
   URL: http://eajelly.xyz
   Ignore SSL: off
   Ignore CORS: off
```

 ## Server datasheet
| Item  | Detail | Value |
| :---: | ------------- | :---: |
| 1  | TV Show, Movies and Music Video 4K resolution  | `2960:auto` |
| 2  | TV Show, Movies and Music Video 1080p resolution  | `1890:auto`  |
| 3  | TV Show, Movies and Music Video 720p resolution  | `1280:auto`  |
| 4  | TV Show, Movies and Music Video cellular resolution  | `540:auto`  |
| 5  | TV Show, Movies and Music Video 4K bitrate  | `VBR Max Rate: 8000K` |
| 6  | TV Show, Movies and Music Video 1080p bitrate  | `VBR Max Rate: 5000K`  |
| 7  | TV Show, Movies and Music Video 720p bitrate  | `VBR Max Rate: 2500K`  |
| 8  | TV Show, Movies and Music Video cellular bitrate  | `VBR Max Rate: 2500K`  |
| 9  | TV Show, Movies and Music Video audio bitrate  | `Stereo: 96K or 128k downmix from surround or AAC 540kbit for 5.1 or 620kbit for 7.1 surround.`  |
| 10  | TV Show, Movies and Music Video audio codec  | `OPUS or AAC for compatible surround`  |
| 11  | TV Show, Movies and Music Video video codec | `H.265 or H.265 Main10; AV1 for some 720p or Cellular`  |
| 12  | TV Show, Movies and Music Video video extension | `MP4`  |
| 13  | TV Show, Movies and Music Video subtitle format | `UTF-8`  |
| 14  | TV Show, Movies and Music Video subtitle codec | `MP4 compatible mov_text`  |
| 15  | Music Bitrate | `256K`  |
| 16  | Music Codec | `AAC`  |
| 17  | Music Audio | `Stereo or Stereo downmixed from DOLBY ATMOS`  |
| 18  | Music Lyrics | `embedded timed lyrics or lyric`  |
| 19  | TV Shows, Movies and Music Video 4K Data Rate | `11 mbps or 2 hour video 4 GB`  |
| 20  | TV Shows, Movies and Music Video 1080p Data Rate | `3 mbps or 2 hour video 1.8 GB`  |
| 21  | TV Shows, Movies and Music Video 720p Data Rate | `2 mbps or 1.5 hour video 1.6 GB`  |
| 22  | TV Shows, Movies and Music Video cellular Data Rate | `1 mbps or 1.5 hour video 1.4 GB`  |
| 23  | Music Data Rate | `300 kbps or 5 minute music 7 MB`  |



<a name="ea-media-tools"></a>
## EA Media Tools

[EA Media Tools](https://github.com/eaforlife/media-converter) is a desktop companion for preparing video, music-video, and audio-library files for playback. Download the current release from the [releases page](https://github.com/eaforlife/media-converter/releases). Its output profiles are configurable; the defaults below describe the current Streaming preset and can differ when the app's configuration is changed.

### First launch

An internet connection is needed for initial setup and later update checks. On startup, the app checks the required app version and prepares the Jellyfin FFmpeg runtime. It installs the stable runtime and offers the prerelease channel when its major/minor version is newer than stable. Choose the active channel with the **Stable** setting. The app also manages rsgain for library loudness normalization and, on supported platforms, CCExtractor for detected music-video captions.

The app tests available decoders and encoders on the machine and exposes only the options that pass those tests. A supported GPU is recommended for hardware acceleration, and CPU encoding/decoding is available when hardware acceleration is turned off. Runtime, preset, and config updates are validated before use; a failed download keeps the last valid installed copy.

Audio libraries are scanned recursively. Audio encoding keeps the source filename stem and artist/album folder layout. Optional loudness normalization runs after all audio jobs in the library complete successfully. When enabled for surround audio, the app creates a coefficient-normalized stereo downmix; optional compression and peak limiting are applied after the downmix. With **Keep surround audio and add a stereo downmix**, each retained surround source track and its generated stereo track can be included independently.

For music videos (videos under eight minutes with attached cover art), the app can copy artwork and extract detected CEA-608/708 captions. It scans beside video files for SRT, ASS, SSA, and WebVTT subtitle files. MP4 output converts supported text subtitles to `mov_text`.

### Presets and configuration

`config.ini` supplies app-wide and UI defaults, output dimensions and rate limits, codec rate multipliers, audio choices and filters, and job behavior. `presets.ini` supplies named preset choices such as quality, speed, format, audio settings, and preset-specific advanced video settings. The app validates and refreshes these managed files from the media-converter repository at startup. Edits to an installed copy can be replaced by a later source refresh.

For persistent personal choices, edit settings in the app or save a named custom preset. Custom presets are stored separately from the managed built-in presets. To change defaults for users of a repository build, update the source INI file and publish the change through the repository instead of relying on edits to a packaged copy.

Key configuration areas:

| File/section | Controls |
| --- | --- |
| `config.ini` `[UI Defaults]` | Default video preset, codec, and Main10 state |
| `config.ini` `[Video Output: 4k, 1080p, 720p, 360p]` | Output dimensions and H.264 bitrate limits for each tier |
| `config.ini` `[Video Codec Rate Multipliers]` and `[Video Behavior]` | Codec rate scaling and the buffer multiplier |
| `config.ini` `[Job Behavior Defaults]` | Default audio/subtitle inclusion, metadata stripping, surround retention, and stereo downmix behavior |
| `config.ini` `[Audio Bitrates]` and `[Audio Filters]` | App-wide audio rate choices and filter templates |
| `presets.ini` named sections | Preset-specific codec, quality, tune, speed, audio, scaling, and advanced settings |

Boolean values use `1`/`true` for enabled and `0`/`false` for disabled. In `presets.ini`, a blank boolean is unchecked; an omitted optional override inherits the app default. The `[Version: major.minor.patch]` line at the start of `presets.ini` is its source freshness tag. The `[App Version: major.minor.patch]` line at the start of `config.ini` identifies the app version for its managed defaults.

Current app-wide defaults include the Streaming preset, HEVC, Main10 enabled where supported, metadata stripping enabled, a 3× buffer multiplier, and configured output dimensions of `2900:-2` (4K), `1890:-2` (1080p), and `1300:-2` (720p). Streaming uses NVENC CQ 33 at 4K and its configured base quality at other tiers; the preset's codec, backend, and tier overrides determine the final settings. Check the current INI files for the exact values supplied by a repository build.

Custom preset files and personalized app settings are separate from `config.ini` and `presets.ini`; source refreshes preserve personal settings while updating managed defaults.

### Conversion workflow

The pre-encode workspace scans selected files or folders before queueing jobs. Folder scans recurse through subfolders; video scans preserve previews and crop suggestions. Sidecar SRT, ASS, SSA, and WebVTT files with matching basenames are offered as subtitle tracks. Two- or three-letter filename tags can set the language, and `default`, `forced`, and `sdh` tags set subtitle flags.

Video, audio, and subtitle processing can be enabled independently. Enabled tracks use the selected conversion settings; disabled sections are copied without re-encoding. When all three are disabled, metadata-only mode lets you edit stream languages and dispositions or import UTF-8 subtitle files. Output behavior controls include audio/subtitle streams and metadata stripping. The default is to strip metadata.

Auto Scale chooses an output tier from the source. Explicit scaling applies that tier's dimensions and rate limits. Auto Crop can detect black bars and derive an even output size from the cropped display aspect ratio. Tone mapping can convert HDR to SDR. On NVIDIA, the app uses a supported CUVID decoder for decoder-side crop when possible; otherwise it uses a protected decode/crop/upload path. A hardware decode failure can retry with software decoding while keeping the selected video encoder.

Streaming currently defaults to HEVC Main10 when the selected encoder supports it. It uses NVENC CQ 33 at 4K and CQ 32 at other tiers; the active output tier and encoder may apply their own settings. Its configured dimensions are 2900 pixels for 4K, 1890 for 1080p, and 1300 for 720p. The default H.264 maximum-rate baselines are 8000, 7500, and 4000 kbit/s for those tiers. The buffer is calculated from the final maximum rate and a 3× multiplier. HEVC, HEVC Main10, AV1, and AV1 10-bit use configured ratios against the H.264 baseline. These values are managed defaults and can change in `config.ini` or `presets.ini`.

A source under eight minutes with attached cover art can use the Music Video workflow. It preserves the filename stem and metadata, copies the artwork, and can extract detected CEA-608/708 captions. Music Video uses the P6 NVENC speed where supported, with H.264 maximum-rate overrides of 11,000 kbit/s at 4K and 9,000 kbit/s at 1080p; other values come from the selected preset and config.

Audio-only folders are scanned recursively. Streaming uses Opus at 96 kbit/s for stereo and 128 kbit/s for surround downmix; Archive uses AAC at 224 kbit/s for stereo and 256 kbit/s for downmix. Enabled loudness normalization runs against the selected output root after all audio jobs complete successfully. Surround downmixing uses the configured coefficients, followed by optional compression and peak limiting. **Keep surround audio and add a stereo downmix** retains the surround stream and creates a two-channel encode; each source surround track and each generated downmix can be included separately.

Queued files have separate progress pages. The app reports current bitrate from recent FFmpeg samples and shows live process output. Start New clears the queued sources after work settles. Batch concurrency can be disabled; otherwise NVENC batches adapt to measured throughput within the tested session limit, while other video encoder families run one at a time. Audio-only batches can process up to four files concurrently.

With smart naming enabled, files with season and episode numbers use a series/season layout; other files are treated as movies. Audio output preserves the source filename stem and relative artist/album folders. When available, matching `.lrc` lyric files are copied beside audio outputs.

Windows x64 checks the Squirrel update feed at startup and periodically while running. Unsigned macOS builds receive updates from the releases page, and Linux packages are updated through the package manager. If a release policy requires a newer version, the app blocks startup until the update is installed or the user exits.

### Hardware encoder support

EA Media Tools uses encoder and decoder options provided by Jellyfin FFmpeg. Available families include NVIDIA NVENC/NVDEC with CUDA, Intel Quick Sync Video, AMD AMF, Linux VA-API, and Apple VideoToolbox. Platform, GPU generation, driver, and runtime affect which codecs and features are available. The app runs hardware checks and only enables options that pass; consult Jellyfin's [hardware acceleration documentation](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/) and the GPU vendor's documentation for platform setup.

| Vendor/platform | Hardware interfaces |
| --- | --- |
| NVIDIA, Windows/Linux | NVENC encoding; NVDEC decoding and CUDA processing when supported |
| Intel, Windows/Linux | Quick Sync Video; VA-API may be available on Linux |
| AMD, Windows | AMF |
| AMD/Intel, Linux | VA-API, depending on the installed driver and device |
| Apple, macOS | VideoToolbox |

Encoder availability is determined by the app's startup test. Laptop variants, disabled integrated GPUs, missing firmware or permissions, and driver/runtime versions can affect results. Linux QSV and VA-API require an accessible render device and the matching vendor media driver.

### System requirements

- Windows 11 x64, macOS on Intel or Apple silicon, or Debian/Ubuntu Linux on amd64 or arm64.
- Internet access for first-launch runtime downloads and update checks.
- A supported GPU and current graphics/media driver for hardware acceleration. GPU acceleration is optional; CPU processing is available.
- On Linux, permission to access the GPU render device when using QSV or VA-API.

For app downloads and version-specific release details, see the [EA Media Tools releases](https://github.com/eaforlife/media-converter/releases).

## Server Link
 - [EA Jelly Home](http://eajelly.xyz)
 - [EA Jelly Registration](https://discord.gg/nwud34Sghr)
 - [EA Jelly repository](https://github.com/eaforlife)
 - [VPS/Nginx operations guide](for-vps-nginx/AGENTS.md)
