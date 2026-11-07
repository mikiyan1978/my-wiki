# YTDownloader TS (TrollStore Edition) Install Guide

日本語版: [YTDownloader TS(TrollStore版)インストールガイド](#/view/ytdownloader-trollstore-guide)

The TrollStore edition of **YTDownloader**, a self-made app for downloading YouTube videos. It runs without a jailbreak and works as a **completely separate app** from the official YTDownloader (jailbreak edition): the bundle ID, URL scheme, save folder and cookie file are all independent, so you can install both on the same device without interference.

Release: [GitHub Releases](https://github.com/mikiyan1978/sileo-repo/releases/tag/ytdownloader-trollstore-v1.0)

---

## What you need

- An iOS device with **TrollStore** installed
- iOS 14 or later
- A Wi-Fi connection (for the first-launch runtime download, about 120 MB)

---

## STEP 1: Download the .tipa

Download `YTDownloaderTrollStore.tipa` from [GitHub Releases](https://github.com/mikiyan1978/sileo-repo/releases/tag/ytdownloader-trollstore-v1.0).

- Open it directly in Safari on your iPhone/iPad and download it, or
- Download it on a computer and AirDrop it to the iPhone (save it to "On My iPhone" in the Files app, for example)

---

## STEP 2: Install with TrollStore

1. Open the **TrollStore** app
2. Tap **"+"** at the top right
3. Choose the downloaded `YTDownloaderTrollStore.tipa`
4. When the installation finishes, **"YTDownloader TS"** (purple icon) appears on your Home Screen

> If you opened the file directly from Safari, choose "Open in TrollStore" to go straight to the install screen.

---

## STEP 3: Runtime setup on first launch

YTDownloader TS uses a runtime built from Python, yt-dlp and Node internally. You need to download and set it up (about 120 MB) **once, on first launch**.

1. When you open the app, a **"Runtime Required"** screen appears
2. Tap **"Start Install (~120MB)"**
3. The download runs, then the files are unpacked and configured automatically
4. **If a screen says one more setup step is needed to make downloads work on this device**:
   - Read it and tap **"Continue (switch to TrollStore)"**
   - The app **switches to TrollStore automatically**
   - When TrollStore shows its confirmation screen, tap **"Install"**
   - After a few seconds YTDownloader TS **restarts automatically** and you are back in the app

> Once set up, the runtime is kept in the app's data area, which survives app updates. You do not need to repeat these steps on later updates.
> Switching to TrollStore and tapping Install is the standard confirmation step TrollStore uses when it officially installs a new app. An unfamiliar screen may surprise you, but it is fine. Follow the instructions and you will return to the app automatically.

When setup is done, the normal screen (paste a video URL to download) becomes available.

---

## Tips

- **Allow notifications**: When you start your first download you are asked whether to allow notifications. If you allow them, you are notified when downloads finish or fail.
- **Keeps going with the screen locked or in other apps**: While downloading, the app keeps running in the background and processes queued videos in order. (It plays silent audio only while downloading to stay awake; your other music is not interrupted.)
- **Download a copied URL right away**: Copy a video URL and open this app, and the start button changes to "Download copied URL". (The clipboard contents are not read until you tap the button.)
- **When YouTube changes and downloads stop working**: In the app, go to "Settings" → the bottom section "Download Engine (yt-dlp)" → **Check for updates**. No app reinstall and no TrollStore re-signing is needed. The app also checks once a day at launch.

---

## Features

### Quality and formats

- You can set the default quality in "Settings": Ask Every Time / Always Best Quality / Always 1080p / Always 720p / Audio Only.
  - With a default set, a link handed over **from another app** (such as the Share menu) first asks "Download this link?" for safety; tapping "Download" starts it at the default quality. If "Sites that need a login" is anything other than "Don't use", links to sites other than YouTube are confirmed regardless of the default quality, and the prompt says the saved login will be used. Pasting inside the app starts without asking.
- **Audio only**: MP3 (with title, artist and cover art) or FLAC (with title and artist).
- **With subtitles**: Choose "Download with Subtitles" in the quality picker to get an MKV with the subtitles embedded. The subtitle language is set in "Settings" → "Subtitle language": Japanese + English (default) / Japanese only / English only. An all-languages option is not offered, because YouTube limits it and the download would fail.

### Chapters

If a video has chapters, they are written into the file automatically (MP4, MKV and WebM). For videos without chapters nothing is added and no extra time is spent.

### Sponsor segments (SponsorBlock)

Choose in "Settings" → "Sponsor segments (SponsorBlock)". The default is Off.

| Setting | What it does |
|---|---|
| Off | Does nothing |
| Mark as chapters | Sponsor, self-promotion and "like and subscribe" segments become chapters (the video itself is unchanged) |
| Cut them out | Removes those segments from the video |

- Segments come from a public database registered by volunteers (sponsor.ajay.app). Only the first 4 characters of a hash are sent, not the video ID itself.
- "Cut" trims and joins the video without re-encoding, so it is fast and loses no quality. On H.264 MP4 (for example in Compatibility mode) the normal way of joining can fail; the app then switches to another method automatically. The cut points can be off by tens of milliseconds up to a few seconds because of keyframes. Only if that also fails is the video saved uncut, with "Done (cutting the sponsor segments failed)" shown.

### Recording live streams

Give the app the URL of a stream that is on air and it records it.

- The screen shows "● Recording", the recording time and the size written so far (a live stream has no fixed end, so no percentage or total is shown).
- The button at the top right is "■ Stop & Save". Tap it and the recording stops within a few seconds and is saved as an MP4.
- When free space runs low, the app stops and saves automatically.
- If the app is closed while recording, the recording cannot be continued.

### Resuming interrupted downloads

Even if the app is closed, the waiting queue and the download that was in progress are saved. The next time you launch the app:

- The queue is **restored paused** (so a large queue never starts by itself).
- For the interrupted download you are asked "Resume it from where it left off?" Tap "Resume" and it continues from where it stopped (it does not start over).

### Sites other than YouTube

You can give URLs of any site yt-dlp supports in the same way. (Verified: Niconico, SoundCloud, Dailymotion and TikTok.)

- **Playlist, album and channel URLs** are queued one item at a time. This includes SoundCloud sets and YouTube channel pages.
- Audio-only sites (such as SoundCloud) are shown as "Audio Only" and saved as M4A and similar formats.
- If you give the same video again and it is already saved, the app says it was already downloaded.
- Videos that need a login (members-only Niconico, some Vimeo videos, X, Instagram and so on) can sometimes be downloaded with the "Sites that need a login" setting below.

#### Sites that need a login

Choose in "Settings" → "Sites that need a login (other than YouTube)". Of the login information (cookies) stored in Safari and similar apps on this device, **only that of the site you are downloading** is used. (Log in to the site in Safari first.) It is never used for YouTube.

| Setting | What it does |
|---|---|
| Don't use (default) | Never uses login information |
| Automatic when needed | Tries without logging in first, and retries once only when the site says a login is required. Your account is not used for public videos |
| Always use | Attaches the login to every request to sites other than YouTube |

- If unsure, "Automatic when needed" is recommended. "Always use" can, depending on the site, get your account restricted, or make downloads fail because of an expired login.
- When a download fails because a login is needed, the app points you to this setting on screen and in a notification.

### Library

- **Switch by platform**: Once you have downloaded from two or more sites, tabs appear at the top: "All | YouTube | Niconico | SoundCloud …". Only the videos from the selected site are shown, grouped by channel. (The selected tab is remembered.) The save folder is still a single folder.
- The total size of your downloads and the free space on the device are shown at the top.
- From each video's share button you can **Save to Photos** (only for supported formats such as MP4) or **Save to Files / Share**.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| The "Runtime Required" screen keeps coming back | You may have closed the app before the download finished. Try again from "Start Install" |
| "Automatic signing timed out" is shown | Check whether you missed the TrollStore confirmation screen. Retrying usually fixes it |
| Choosing YTDownloader TS from YouTube's share menu opens the jailbreak edition | The two use different URL schemes, so this should not happen. If it does, try reinstalling YTDownloader TS once |
| After updating the app, "Runtime Required" came back | This happens **only once, when updating from a version older than this one** (older versions kept the runtime inside the app, so an update wiped it). Do STEP 3 again. Your settings and saved videos are not lost. **From this version on, the runtime survives updates**, so STEP 3 does not need to be repeated |
| The size during a download shows only "129.4 MB", with no total | For formats whose total is not known in advance, only the bytes actually written are shown, and the total is fixed when the download finishes. (YouTube's split formats show the total from the start.) |
| Downloads do not work (the video cannot be fetched) | Check that the runtime setup finished correctly (see STEP 3). A YouTube-side change can also be the cause |

---

## Differences from the jailbreak edition

| | Jailbreak edition (via Sileo) | TrollStore edition (this guide) |
|---|---|---|
| Requirement | A jailbreak such as palera1n | TrollStore only (no jailbreak) |
| Faster downloads (aria2c) | Yes (via apt) | Yes (bundled in the app) |
| First-time setup | Runtime download only | Runtime download + (depending on the device) the TrollStore re-sign confirmation |
| Save folder, cookies, icon | Jailbreak edition only | TrollStore edition only (fully separate) |

The features are nearly identical. Choose the TrollStore edition if you want to use it on a device that is not jailbroken.
