# WhatsApp Chat Archive Viewer

A zero-dependency, client-side viewer for the Android WhatsApp export in this repository. The page reads `chat.txt` and attachment files from the same directory; it needs no Node.js, Python, build step, or server-side processing.

## Open the archive

Serve the repository root over HTTP/HTTPS, or publish it with GitHub Pages. Opening `index.html` directly as a `file://` URL will not load `chat.txt` because browsers restrict local-file fetches. The viewer supports participant perspective switching, keyword search, system events, images with a full-screen preview, audio, video, and downloadable files.

Each message displays only its clock time; a centered pill divider marks each new Cairo-local date and groups that day’s conversation beneath it. Times and date dividers use the IANA `Africa/Cairo` timezone, including Cairo’s date-specific daylight-saving offset. WhatsApp’s exported timestamp text does not contain a source timezone, so the viewer treats the exported date/time as Cairo-local wall time rather than guessing a conversion from another timezone.

For future Android exports, extract the ZIP and put the transcript at the repository root as `chat.txt`. Keep the original WhatsApp media filenames in the same directory. Check repository and individual file sizes before committing; GitHub rejects files larger than 100 MB.

## Privacy and access

This repository contains a personal WhatsApp conversation and media attachments. The repository is currently public, so its files are already publicly accessible. Publishing the viewer with GitHub Pages would also make the chat available through a convenient public website; GitHub Pages does not add authentication. Do not publish this content unless public access is intended. For restricted viewing, keep the repository private and use a host with access controls (for example, Cloudflare Pages with Cloudflare Access).

## Enable GitHub Pages

On GitHub, open **Settings → Pages**. Under **Build and deployment**, select **Deploy from a branch**, choose `main` and `/(root)`, then save. Pages will provide the site URL after deployment finishes. This is a static site and requires no build command.
