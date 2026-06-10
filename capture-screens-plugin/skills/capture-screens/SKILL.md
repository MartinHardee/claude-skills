# Capture Screens

Capture full-page desktop (1568px wide) and mobile (430px wide, 4000px tall) screenshots of websites and upload them as image fills into Figma frames.

## Environment

### Paths (all sessions — /tmp is wiped on restart)

- Playwright: `/services/foundry/foundry.runfiles/_main/node_modules/.aspect_rules_js/playwright@1.58.1/node_modules/playwright/index.mjs`
- Firefox binary: `/home/foundry/.cache/ms-playwright/firefox-1509/firefox/firefox`
- LD_LIBRARY_PATH: `/home/foundry/.cache/ms-playwright/firefox-1509/firefox:/tmp/chrome_libs/usr/lib/x86_64-linux-gnu:/tmp/chrome_libs/lib/x86_64-linux-gnu`

Both `/usr/lib/` AND `/lib/` paths are required — dbus lives in `/lib/`, not `/usr/lib/`.

### chrome_libs (rebuilt each session — /tmp is wiped)

Download Debian Bullseye .deb packages and extract with `dpkg -x`. Several packages must come from `security.debian.org`, not `deb.debian.org`. Check `packages.debian.org/bullseye/amd64/<pkg>/download` for correct URLs.

**Required packages and correct source repos:**

| Package | Source |
|---------|--------|
| `libglib2.0-0` (2.66.8-1+deb11u8) | `security.debian.org` |
| `libgdk-pixbuf-2.0-0` (2.42.2+dfsg-1+deb11u5) | `security.debian.org` |
| `libx11-6` (1.7.2-1+deb11u2) | `security.debian.org` |
| `libx11-xcb1` (1.7.2-1+deb11u2) | `security.debian.org` |
| `libgtk-3-0` (3.24.24-4+deb11u4) | `security.debian.org` or `ftp.us.debian.org` |
| `libgraphite2-3` | `deb.debian.org` |
| `libxau6` | `deb.debian.org` |
| `libxdmcp6` | `deb.debian.org` |
| `libbsd0` | `deb.debian.org` |
| `libmd0` | `deb.debian.org` |
| `libasound2` | `deb.debian.org` or `packages.debian.org` |

**Test Firefox works after restoring libs:**

    LD_LIBRARY_PATH=/home/foundry/.cache/ms-playwright/firefox-1509/firefox:/tmp/chrome_libs/usr/lib/x86_64-linux-gnu:/tmp/chrome_libs/lib/x86_64-linux-gnu \
      /home/foundry/.cache/ms-playwright/firefox-1509/firefox/firefox --version
    # Should print: Mozilla Firefox 146.0.1

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

## Screenshot Script Template (Desktop, 1568px)

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
      }, interB64);  // ← pass interB64 (the outer variable), NOT a variable named b64
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

    // Dismiss popups (see Popup Dismissal section)
    await dismissAll(page);

    await page.evaluate(() => {
      document.querySelectorAll('img[loading="lazy"]').forEach(img => {
        img.loading = 'eager';
        if (img.dataset.src) img.src = img.dataset.src;
        if (img.dataset.srcset) img.srcset = img.dataset.srcset;
      });
      document.body.style.overflow = 'auto';
      document.documentElement.style.overflow = 'auto';
    }).catch(() => {});

    await injectFonts(page);
    await injectVideoFrames(page);

    // Standard scroll — 300px steps at 200ms
    for (let y = 0; y <= 8000; y += 300) {
      await page.evaluate(p => window.scrollTo(0, p), y).catch(() => {});
      await page.waitForTimeout(200);
    }
    await page.waitForTimeout(2000);

    // Scroll back to top
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

## Mobile Screenshot Script (430px)

Use a mobile UA and 430px viewport. **Critical: do NOT pass `isMobile: true` or `hasTouch: true` — these are not supported in Firefox and will throw an error.**

    const ctx = await browser.newContext({
      viewport: { width: 430, height: 900 },
      userAgent: 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1',
      locale: 'en-GB',
      // NO isMobile or hasTouch — not supported in Firefox, will throw
    });

**Check page height before screenshotting** — pages over ~30,000px hit Firefox's rendering limit:

    const pageHeight = await page.evaluate(() =>
      Math.max(document.body.scrollHeight, document.documentElement.scrollHeight)
    ).catch(() => 5000);

    if (pageHeight > 30000) {
      await page.screenshot({ path: full, clip: { x: 0, y: 0, width: 430, height: Math.min(pageHeight, 20000) } });
    } else {
      await page.screenshot({ path: full, fullPage: true });
    }

---

## Popup Dismissal

Many sites show cookie consent, geolocation/region, and newsletter popups. Dismiss them in multiple rounds.

**Include geolocation/region texts** — sites like EME Studios show "WE NOTICED YOU'RE BROWSING IN A DIFFERENT LOCATION" with buttons like "CONTINUE IN UNITED KINGDOM":

    const dismissAll = async (page) => {
      const clicked = await page.evaluate(() => {
        const texts = [
          'continue in united kingdom', 'continue in uk', 'stay in united kingdom',
          'accept all', 'accept cookies', 'allow all', 'i accept', 'agree', 'allow cookies', 'accept',
          'no thanks', 'no, thanks', 'dismiss', 'skip', 'close', 'not now',
        ];
        for (const btn of document.querySelectorAll('button, [role="button"], a')) {
          const txt = (btn.innerText || btn.textContent || '').trim().toLowerCase();
          if (texts.some(t => txt.includes(t))) { btn.click(); return txt; }
        }
        // Also try × / ✕ close icons
        for (const btn of document.querySelectorAll('button, [role="button"]')) {
          const html = btn.innerHTML || '';
          if (html.includes('×') || html.includes('✕') || html.includes('&times;')) { btn.click(); return 'close icon'; }
        }
      }).catch(() => null);
      if (clicked) console.log('Clicked:', clicked);
      return clicked;
    };

    // Multiple dismissal rounds
    for (let round = 0; round < 4; round++) {
      const clicked = await dismissAll(page);
      if (clicked) await page.waitForTimeout(2000);
      await page.keyboard.press('Escape');
      await page.waitForTimeout(500);
    }

    // Aggressive overlay removal — remove any large fixed/absolute element with z-index > 100
    await page.evaluate(() => {
      document.querySelectorAll('*').forEach(el => {
        const st = getComputedStyle(el);
        if (['fixed','absolute'].includes(st.position) && parseInt(st.zIndex) > 100) {
          const rect = el.getBoundingClientRect();
          if (rect.width > 300 && rect.height > 100) { el.remove(); }
        }
      });
    }).catch(() => {});

---

## Slow Scroll for Heavy Lazy Loading

Sites like ASOS use IntersectionObserver and require slower scroll to trigger image loading. Use 150px steps at 150ms instead of the standard 300px/200ms:

    // Slow scroll — 150px at a time, 150ms each
    const pageHeight = await page.evaluate(() => document.body.scrollHeight).catch(() => 8000);
    for (let y = 0; y <= Math.min(pageHeight, 15000); y += 150) {
      await page.evaluate(p => window.scrollTo(0, p), y).catch(() => {});
      await page.waitForTimeout(150);
    }

---

## Font Injection

**Critical rule: inject AFTER `page.goto()`, never in `addInitScript`.**

Sites load their own CSS with `!important` after `addInitScript` runs, overriding any style injected there. Using `page.evaluate()` post-load appends the style last, so it wins.

**Critical rule: pass `interB64` (the outer variable) as the argument to `page.evaluate`.**

The callback parameter name (`b64`) is the local name inside the callback. The second argument to `page.evaluate` must be the variable from the outer scope:

    // CORRECT
    await page.evaluate((b64) => { /* uses b64 */ }, interB64);

    // WRONG — b64 is not defined in outer scope, causes silent failure
    await page.evaluate((b64) => { /* uses b64 */ }, b64);

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

**Critical: concatenate ALL IDAT chunks before decompressing.** PNG files can have multiple IDAT chunks. Only decompressing the first chunk causes `zlib.error: Error -5 while decompressing data: incomplete or truncated stream`.

    import sys, struct, zlib, shutil

    def crop_png(inp, out, max_h):
        with open(inp, 'rb') as f: data = f.read()
        assert data[:8] == b'\x89PNG\r\n\x1a\n', "Not a PNG"
        i = 8; chunks = []
        while i < len(data):
            if i + 12 > len(data): break
            length = struct.unpack('>I', data[i:i+4])[0]
            ctype = data[i+4:i+8]; cdata = data[i+8:i+8+length]
            chunks.append((ctype, cdata)); i += 12 + length
        ihdr_data = next(d for t,d in chunks if t == b'IHDR')
        w = struct.unpack('>I', ihdr_data[:4])[0]
        h = struct.unpack('>I', ihdr_data[4:8])[0]
        new_h = min(h, max_h)
        if new_h >= h: shutil.copy(inp, out); return
        # CRITICAL: concatenate ALL IDAT chunks before decompressing
        raw_compressed = b''.join(d for t,d in chunks if t == b'IDAT')
        raw = zlib.decompress(raw_compressed)
        bit = ihdr_data[8]; color = ihdr_data[9]
        bpp = {0:1, 2:3, 3:1, 4:2, 6:4}.get(color, 3)
        if bit == 16: bpp *= 2
        stride = 1 + w * bpp
        rows = [raw[i*stride:(i+1)*stride] for i in range(new_h)]
        new_raw = zlib.compress(b''.join(rows), 6)
        def make_chunk(t, d):
            return struct.pack('>I', len(d)) + t + d + struct.pack('>I', zlib.crc32(t+d) & 0xffffffff)
        new_ihdr = ihdr_data[:4] + struct.pack('>I', new_h) + ihdr_data[8:]
        with open(out, 'wb') as f:
            f.write(b'\x89PNG\r\n\x1a\n')
            f.write(make_chunk(b'IHDR', new_ihdr))
            for t,d in chunks:
                if t not in (b'IHDR', b'IDAT', b'IEND'): f.write(make_chunk(t, d))
            f.write(make_chunk(b'IDAT', new_raw))
            f.write(make_chunk(b'IEND', b''))

    crop_png(sys.argv[1], sys.argv[2], int(sys.argv[3]))

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

| Site | Notes |
|------|-------|
| browniespain.com | Country selector: `button.brn-country-selector__button` ("Go!"). Full-viewport video hero — extract frame from MP4 with ffmpeg. Remove `.brn-gift-sheet`, `.brn-gift-drawer__overlay`, `[class*="country-selector"]` |
| uk.brandymelville.com | UK domain (not `brandymelvilleusa.com`). No video hero. Font fix required. |
| emestudios.com | Shows geolocation/region popup "CONTINUE IN UNITED KINGDOM" — requires multi-round dismissal including geolocation texts, ESC press, and z-index overlay removal |
| asos.com | Heavy lazy loading — use slow scroll (150px/150ms) instead of standard 300px/200ms |
| zara.com/uk/en | Mobile gendered URLs (`man-mkt1093`, `woman-mkt1040`) both redirect to new-in pages regardless of gender — results in same hash for both |
| pullandbear.com | Headless bot detection — returns minimal content, page height = viewport only |
| urbanoutfitters.com | Headless bot detection — returns minimal content |
| coldculture.com | Headless bot detection — returns minimal content |
| hm.com | Blocked by Cloudflare — cannot capture from this environment |

---

## Debugging Checklist

- **Blank content / tiny screenshot:** viewport height set to 4000 — change to 900 + fullPage:true
- **Broken/emoji fonts:** font injection in `addInitScript` — move to `page.evaluate()` after `page.goto()`
- **Font injection silently failing:** `injectFonts` passes wrong variable to `page.evaluate` — ensure second arg is `interB64` (outer scope), not `b64` (local name)
- **Blank video hero area:** H.264 video — detect via `video.networkState === 3`, extract frame with static ffmpeg
- **LD_LIBRARY_PATH error / browser crash:** missing `/tmp/chrome_libs/lib/` path (dbus) — add both `/usr/lib/` and `/lib/` variants; check `/tmp/chrome_libs` was rebuilt (wiped each session)
- **`libgmodule-2.0.so.0` not found:** libglib2.0-0 missing — download from `security.debian.org`, not `deb.debian.org`
- **`libgdk_pixbuf-2.0.so.0` not found:** download libgdk-pixbuf-2.0-0 from `security.debian.org`
- **`libX11.so.6` not found:** download libx11-6 from `security.debian.org`
- **`libasound.so.2` not found:** download libasound2 from packages.debian.org
- **`zlib.error: Error -5` in crop_png:** only decompressing first IDAT chunk — concatenate ALL IDAT chunks before decompressing
- **`isMobile is not supported in Firefox`:** removed `isMobile: true` and `hasTouch: true` from `browser.newContext()` — Firefox doesn't support these
- **MP4 download fails:** try the SD version of the video URL if HD is blocked
- **Figma fill not updating:** use the two-step imageHash approach above, not `upload_assets` with `nodeId`
- **Popup still showing after dismissal:** site has multiple popup layers (cookie + region + newsletter) — use multi-round loop with ESC + z-index overlay removal; add region-specific texts to click list
- **Images not loading below fold (ASOS-style sites):** use slow scroll (150px/150ms) to trigger IntersectionObserver reliably
- **Page height = 900 (viewport only) for mobile:** site detects headless browser and returns skeleton only — no fix available
