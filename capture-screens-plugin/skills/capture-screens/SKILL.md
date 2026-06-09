# Capture Screens

Capture full-page desktop screenshots of websites (1568px wide, 4000px tall) and upload them as image fills into Figma frames.

## Environment

### Paths (all sessions — /tmp is wiped on restart)

- Playwright: `/services/foundry/foundry.runfiles/_main/node_modules/.aspect_rules_js/playwright@1.58.1/node_modules/playwright/index.mjs`
- Firefox binary: `/home/foundry/.cache/ms-playwright/firefox-1509/firefox/firefox`
- LD_LIBRARY_PATH: `/home/foundry/.cache/ms-playwright/firefox-1509/firefox:/tmp/chrome_libs/usr/lib/x86_64-linux-gnu:/tmp/chrome_libs/lib/x86_64-linux-gnu`

Both `/usr/lib/` AND `/lib/` paths are required — dbus lives in `/lib/`, not `/usr/lib/`.

### chrome_libs (rebuilt each session — /tmp is wiped)

Download Debian Bullseye .deb packages and extract with `dpkg -x`. Some packages (e.g. libgdk-pixbuf) must come from `security.debian.org`, not `deb.debian.org`. Check `packages.debian.org/bullseye/amd64/<pkg>/download` for correct URLs.

### Inter font

Download once per session to `/tmp/inter.woff2`. Route all font requests to it:

    await page.route(/\.(woff2?|ttf|otf)(\?.*)?$/, route => route.fulfill(INTER_RESPONSE));

Then inject post-load (see Font Injection section).

### Static ffmpeg for H.264 (rebuilt each session)

`/tmp/ffmpeg_h264` — from GitHub BtbN/FFmpeg-Builds. Download and extract via Python lzma (no `xz` command available):

    python3 -c "
    import lzma, tarfile, io, urllib.request
    url = 'https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-linux64-gpl.tar.xz'
    data = urllib.request.urlopen(url).read()
    with lzma.open(io.BytesIO(data)) as xz:
        with tarfile.open(fileobj=xz) as tar:
            for m in tar.getmembers():
                if m.name.endswith('/ffmpeg'):
                    m.name = 'ffmpeg_h264'
                    tar.extract(m, '/tmp')
                    break
    "
    chmod +x /tmp/ffmpeg_h264

---

## Screenshot Script Template

    import { firefox } from '/services/foundry/foundry.runfiles/_main/node_modules/.aspect_rules_js/playwright@1.58.1/node_modules/playwright/index.mjs';
    import { statSync, readFileSync } from 'fs';
    import { execSync } from 'child_process';

    process.env.LD_LIBRARY_PATH = '/home/foundry/.cache/ms-playwright/firefox-1509/firefox:/tmp/chrome_libs/usr/lib/x86_64-linux-gnu:/tmp/chrome_libs/lib/x86_64-linux-gnu';
    const FF = '/home/foundry/.cache/ms-playwright/firefox-1509/firefox/firefox';
    const INTER_WOFF2 = readFileSync('/tmp/inter.woff2');
    const INTER_RESPONSE = { status: 200, contentType: 'font/woff2', body: INTER_WOFF2, headers: { 'Access-Control-Allow-Origin': '*' } };
    const interB64 = INTER_WOFF2.toString('base64');

    async function injectFonts(page) {
      await page.evaluate((b64) => {
        const dataUri = `data:font/woff2;base64,${b64}`;
        const el = document.getElementById('_fo'); if (el) el.remove();
        const s = document.createElement('style'); s.id = '_fo';
        s.textContent = `
          @font-face { font-family: "_I"; src: url("${dataUri}") format("woff2"); font-weight: 100 900; font-style: normal; }
          @font-face { font-family: "_I"; src: url("${dataUri}") format("woff2"); font-weight: 100 900; font-style: italic; }
          html *, html *::before, html *::after { font-family: "_I", monospace !important; }
        `;
        document.head.appendChild(s);
      }, interB64);
    }

    const browser = await firefox.launch({
      headless: true, executablePath: FF,
      firefoxUserPrefs: {
        'gfx.downloadable_fonts.enabled': true,
        'gfx.downloadable_fonts.fallback_delay': -1,
        'intl.accept_languages': 'en-GB, en',
        'geo.enabled': false,
      },
    });
    const ctx = await browser.newContext({
      viewport: { width: 1568, height: 900 },
      userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:149.0) Gecko/20100101 Firefox/149.0',
      locale: 'en-GB',
    });
    const page = await ctx.newPage();
    page.on('dialog', d => d.dismiss().catch(() => {}));
    await page.route(/\.(woff2?|ttf|otf)(\?.*)?$/, route => route.fulfill(INTER_RESPONSE));

    try { await page.goto(URL, { waitUntil: 'networkidle', timeout: 50000 }); }
    catch { await page.goto(URL, { waitUntil: 'load', timeout: 35000 }).catch(() => {}); }
    await page.waitForTimeout(3000);

    await page.evaluate(() => {
      document.querySelectorAll('[class*="popup"], [class*="modal"], [class*="overlay"], [class*="cookie"]').forEach(el => {
        if (['fixed','absolute'].includes(getComputedStyle(el).position)) el.remove();
      });
      document.body.style.overflow = 'auto';
      document.documentElement.style.overflow = 'auto';
    }).catch(() => {});

    await injectFonts(page);
    await injectVideoFrames(page);

    await page.evaluate(() => {
      document.querySelectorAll('img[loading="lazy"]').forEach(img => {
        img.loading = 'eager';
        if (img.dataset.src) img.src = img.dataset.src;
        if (img.dataset.srcset) img.srcset = img.dataset.srcset;
      });
    }).catch(() => {});

    for (let y = 0; y <= 8000; y += 300) {
      await page.evaluate(p => window.scrollTo(0, p), y).catch(() => {});
      await page.waitForTimeout(200);
    }
    await page.waitForTimeout(2000);

    for (let i = 0; i < 5; i++) {
      await page.evaluate(() => { window.scrollTo(0, 0); document.documentElement.scrollTop = 0; document.body.scrollTop = 0; }).catch(() => {});
      await page.waitForTimeout(600);
    }

    await injectFonts(page);
    await injectVideoFrames(page);
    await page.waitForTimeout(1000);

    await page.screenshot({ path: '/tmp/captures/out_full.png', fullPage: true });
    execSync('python3 /tmp/crop_png.py /tmp/captures/out_full.png /tmp/captures/out.png 4000');
    await browser.close();

---

## Font Injection

**Critical rule: inject AFTER `page.goto()`, never in `addInitScript`.**

Sites load their own CSS with `!important` after `addInitScript` runs, overriding any style injected there. Using `page.evaluate()` post-load appends the style last, so it wins.

Embed Inter as a base64 data URI — avoids URL resolution issues in headless context:

    const dataUri = `data:font/woff2;base64,${interB64}`;

Target selector must include pseudo-elements:

    html *, html *::before, html *::after { font-family: "_I", monospace !important; }

Call `injectFonts()` twice: once before scrolling, once after — dynamic content added during scroll may not have the override applied.

---

## Video Heroes

**Problem:** Firefox headless cannot play H.264 MP4 (`networkState: 3` = NETWORK_NO_SOURCE). Video areas show as blank/white in screenshots.

**Detection:** After page load, check for `<video>` elements with non-zero `getBoundingClientRect()`. If found, treat as a video hero.

**Fix — extract a frame and inject as an img:**

1. Find the MP4 URL (intercept network requests or inspect `video.currentSrc` / `source[src]`)
2. Download the MP4 with curl
3. Extract a frame at 1s with static ffmpeg:

        /tmp/ffmpeg_h264 -ss 1 -i /tmp/video.mp4 -vframes 1 -q:v 2 /tmp/captures/hero_frame.jpg -y

4. Read the frame, base64-encode it, and inject:

        async function injectVideoFrames(page, heroB64) {
          await page.evaluate((heroB64) => {
            const heroDataUri = `data:image/jpeg;base64,${heroB64}`;
            document.querySelectorAll('video').forEach(v => {
              const img = document.createElement('img');
              img.src = heroDataUri;
              img.style.cssText = 'width:100%;height:100%;object-fit:cover;display:block;';
              if (v.parentElement && getComputedStyle(v.parentElement).position === 'static') {
                v.parentElement.style.position = 'relative';
              }
              v.parentNode.insertBefore(img, v);
              v.style.display = 'none';
            });
          }, heroB64);
        }

Call `injectVideoFrames()` before AND after scrolling, same as fonts.

---

## Cropping (fullPage + Python crop)

**Always use `viewport.height: 900`, NOT 4000.** With height 4000 the page doesn't scroll, lazy images never load, and content is missing.

Take a `fullPage: true` screenshot then crop to 4000px with this pure-Python PNG cropper — save as `/tmp/crop_png.py`:

    import sys, zlib, struct

    def u32(b, i): return struct.unpack('>I', b[i:i+4])[0]

    def crop(src, dst, h):
        with open(src, 'rb') as f: data = f.read()
        assert data[:8] == b'\x89PNG\r\n\x1a\n'
        w = u32(data, 16); orig_h = u32(data, 20)
        bd = data[24]; ct = data[25]
        h = min(h, orig_h)
        bpp = (3 if ct == 2 else 4 if ct == 6 else 1)
        stride = w * bpp + 1
        raw = b''
        i = 8
        while i < len(data):
            length = u32(data, i); ctype = data[i+4:i+8]
            chunk_data = data[i+8:i+8+length]
            if ctype == b'IDAT': raw += chunk_data
            i += 12 + length
        pixels = zlib.decompress(raw)
        cropped = pixels[:h * stride]
        def make_chunk(t, d):
            c = t + d
            return struct.pack('>I', len(d)) + c + struct.pack('>I', zlib.crc32(c) & 0xffffffff)
        ihdr = struct.pack('>II', w, h) + bytes([bd, ct, 0, 0, 0])
        idat = zlib.compress(cropped, 6)
        with open(dst, 'wb') as f:
            f.write(b'\x89PNG\r\n\x1a\n')
            f.write(make_chunk(b'IHDR', ihdr))
            f.write(make_chunk(b'IDAT', idat))
            f.write(make_chunk(b'IEND', b''))

    crop(sys.argv[1], sys.argv[2], int(sys.argv[3]))

Usage: `python3 /tmp/crop_png.py full.png cropped.png 4000`

---

## Uploading to Figma

The `upload_assets` `nodeId` parameter is unreliable — the upload succeeds but the frame fill sometimes isn't updated. Always use this two-step approach instead:

1. Upload without `nodeId` to get an imageHash:

        curl -s -X POST "$SUBMIT_URL" -H "Content-Type: image/png" --data-binary @/tmp/captures/out.png
        # returns: {"success":true,"imageHash":"abc123...","placedOnNodeId":"373:2"}

2. Set the fill via `evaluate_script` and clean up the stray placed node:

        const node = await figma.getNodeByIdAsync('296:19');
        node.fills = [{ type: 'IMAGE', imageHash: 'abc123...', scaleMode: 'FILL' }];
        const stray = await figma.getNodeByIdAsync('373:2');
        if (stray) stray.remove();

---

## Known Sites & Quirks

| Site | Frame | Notes |
|------|-------|-------|
| browniespain.com | 296:19 | Country selector: `button.brn-country-selector__button` ("Go!"). Full-viewport video hero — extract frame from MP4 with ffmpeg. Remove `.brn-gift-sheet`, `.brn-gift-drawer__overlay`, `[class*="country-selector"]` |
| uk.brandymelville.com | 296:16 | UK domain (not `brandymelvilleusa.com`). No video hero. Font fix required. |
| H&M (hm.com) | 296:2, 296:3 | Blocked by Cloudflare — cannot capture from this environment |

---

## Debugging Checklist

- **Blank content / tiny screenshot:** viewport height set to 4000 — change to 900 + fullPage:true
- **Broken/emoji fonts:** font injection in `addInitScript` — move to `page.evaluate()` after `page.goto()`
- **Blank video hero area:** H.264 video — detect via `video.networkState === 3`, extract frame with static ffmpeg
- **LD_LIBRARY_PATH error / browser crash:** missing `/tmp/chrome_libs/lib/` path (dbus) — add both `/usr/lib/` and `/lib/` variants
- **MP4 download fails:** try the SD version of the video URL if HD is blocked
- **Figma fill not updating:** use the two-step imageHash approach above, not `upload_assets` with `nodeId`
