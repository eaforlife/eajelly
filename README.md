# About EA Jelly

 EA Jelly website is a Jellyfin server with movies, music, tv shows for personal use that's usually not on Netflix PH. Not for distribution or resell.

<a name="available-apps"></a>
 ## Apps for EA Jelly

 EA Jelly can be accessed on http://eajelly.xyz on any browsers that supports HTML5 and Javascript. Watching or listening through the website can generally work but Jellyfin has community driven third party app to make viewing experience better. For example, some browser don't support playback content by default, or if your hardware does support it, the browser can sometimes fall back to using CPU to play content. The apps below are proven to be simple set and forget. Note that official Jellyfin app exists on iOS and Android. They can work but the app recommended below supports more feature.
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
| 1  | TV Show, Movies and Music Video 4K resolution  | `2720:auto` |
| 2  | TV Show, Movies and Music Video 1080p resolution  | `1760:auto`  |
| 3  | TV Show, Movies and Music Video 720p resolution  | `1320:auto`  |
| 4  | TV Show, Movies and Music Video cellular resolution  | `540:auto`  |
| 5  | TV Show, Movies and Music Video 4K bitrate  | `VBR Max Rate: 8000K` |
| 6  | TV Show, Movies and Music Video 1080p bitrate  | `VBR Max Rate: 5000K`  |
| 7  | TV Show, Movies and Music Video 720p bitrate  | `VBR Max Rate: 2500K`  |
| 8  | TV Show, Movies and Music Video cellular bitrate  | `VBR Max Rate: 2500K`  |
| 9  | TV Show, Movies and Music Video audio bitrate  | `Stereo: 96K or 128k downmix from surround`  |
| 10  | TV Show, Movies and Music Video audio codec  | `OPUS`  |
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


 ## Server Link
 - [EA Jelly Home](http://eajelly.xyz)
 - [EA Jelly Registration](https://discord.gg/nwud34Sghr)
 - [EA Jelly repository](https://github.com/eaforlife)
