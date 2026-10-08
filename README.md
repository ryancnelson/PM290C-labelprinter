# PM290C Label Printer

A local Chrome app that prints text and images directly to a PM290C Bluetooth thermal sticker printer using **Web Bluetooth (BLE)**. Built and physically verified on October 8, 2026.

The printer normally uses Labelnize. Our app needs neither Labelnize nor the capture relay for everyday printing.

## Where the code lives

This directory currently contains this README; the implementation has not been moved or copied here.

Working artifacts:

- `~/Documents/Codex/2026-10-08/i-h/outputs/pm290c/`: SPA, local server, tests, protocol notes, and successful Chrome print log.
- `~/Documents/Codex/2026-10-08/i-h/outputs/pm290c-relay/`: native macOS BLE relay, Swift source, app bundle, capture analyzer, and captured traffic.

## Run it

Requires Node.js 18 or newer and desktop Chrome with Bluetooth enabled. Tested on macOS.

1. Start the server:

   ```sh
   cd ~/Documents/Codex/2026-10-08/i-h/outputs/pm290c
   npm start
   ```

2. Open <http://127.0.0.1:8769/> in Chrome.
3. Turn on the PM290C and disconnect it from Labelnize. Close the relay if it is running.
4. Click **Connect printer**, select PM290C, and pair.
5. Compose a sticker and click **Print sticker**. **Connection details** also has a small test print.

Keep the local server running. It binds only to loopback. No browser security restrictions need to be disabled.

Features include text wrapping, fonts, size, bold, alignment, margins, sticker length, local image import, dithering or threshold conversion, black-and-white preview, PNG export, and diagnostic export. Text drafts survive reloads; images must be reselected.

Text and images stay on the Mac and travel directly to the printer over Bluetooth. No remote service or account is involved.

## The man-in-the-middle story

The interesting part was discovering what the printer actually wanted.

We initially inspected the Android Labelnize APK and built a sender from its apparent PM290C print path. Chrome could connect and write bytes, but the printer produced nothing. Successful Bluetooth writes were not evidence of a valid print job.

After several unsuccessful trials, Ryan pushed for rigorous, evidence-driven debugging and suggested a better experiment: present a fake printer to the iOS app and intercept a real print. He first proposed capture with the real printer off, then authorized doing both sides if possible.

We built a native macOS app using CoreBluetooth that could play both roles at once:

```text
Labelnize on iPhone
        |
        | BLE: iPhone thinks it is talking to PM290C
        v
Mac running PM290CRelay
  peripheral role: advertises a printer to the iPhone
  central role: connects to the real PM290C
  logs and forwards traffic in both directions
        |
        | BLE: real printer sees the Mac as its client
        v
Real PM290C
```

The Mac discovered the real printer's GATT services and cloned the relevant service/characteristic layout. It advertised the PM290C local name and advertised service UUIDs. macOS does not allow arbitrary manufacturer advertisement payloads through this API, but the cloned name and services were sufficient for Labelnize to discover and connect.

This was an application-level BLE relay: two separate BLE connections, with the Mac forwarding characteristic writes, reads, subscriptions, and notifications. It was not a passive radio sniffer or a break of Bluetooth encryption.

### The packet-size snag

The iPhone sent 512-byte writes to the emulated printer, while the Mac's connection to the physical printer allowed at most 500 bytes per write without response.

The initial relay refused oversized writes and stalled. We corrected it to split 512-byte writes into 500+12, preserving every byte in order and logging the changed packet boundaries. We also fixed capture handling so all requests in an incoming callback batch were logged before processing.

That distinction mattered: raster bytes had to remain identical, but their BLE packet boundaries did not.

### The Sudoku breakthrough

Ryan printed a Sudoku puzzle from Labelnize through the relay. The real printer printed it, and Ryan confirmed it worked.

The capture gave us a known-good exchange instead of another inferred command sequence:

- One combined ASCII header, including **PRINT 1,1**.
- Printer-issued byte allowances before raster blocks.
- Exactly the raster bytes, with **no ALLEND trailer**.
- A final **63 00** completion notification.

The successful capture is:

```text
pm290c-relay/captures/capture-1791489431.jsonl
```

It records timestamped iPhone writes, outgoing printer writes, printer notifications, and notifications forwarded back to the iPhone. The analyzer verified that forwarded bytes remained identical and ordered.

We then closed the relay, updated the SPA from this evidence, and printed directly from Chrome. The printer returned the same completion response, and Ryan confirmed the test sticker “looks great.”

## The working protocol

The PM290C advertises local name PM290C and service AF30. Its print GATT service is FF00:

| Characteristic | Purpose |
| --- | --- |
| FF02 | Write without response: commands and raster bytes |
| FF01 | Notifications: status, byte allowances, completion |

The status query is hex `1b 21 3f 0d 0a`. The observed ready response is `00`.

For the 54 mm long Sudoku, this entire header was a single 90-byte write. Each displayed line ends in CRLF except the final comma, which has no newline:

```text
SIZE 54 mm,54 mm
GAP 0,0
DIRECTION 0,0
DENSITY 10
CLS
PRINT 1,1
BITMAP 0,0,48,432,1,
```

The paper is 54 mm wide; the printable bitmap is 384 dots / 48 mm wide. Raster rows are 48 bytes, MSB first, with 0 meaning black and 1 meaning white. Length uses 8 dots/mm.

The captured handshake:

1. Send the combined header.
2. Receive `f0 00 46 50`: allowance of 18,000 bytes, interpreting the final two bytes as big-endian.
3. Wait about 400 ms, then send 18,000 raster bytes.
4. Receive `f0 00 0a b0`: allowance of 2,736 bytes.
5. Wait about 400 ms, then send the remaining 2,736 bytes.
6. Receive `63 00`: print completion.

Total raster: 20,736 bytes = 48 × 432. No trailer was sent.

The SPA sends raster in 240-byte writes with 15 ms pacing, waits 400 ms after each allowance, and never exceeds the allowance or the remaining image length.

One final lesson came from the direct Chrome test: the printer allowed 5,762 bytes for a 5,760-byte image. An overly strict equality check initially stopped the job before sending the raster. After a power cycle, treating the response as an allowance and sending only the 5,760 image bytes worked. No invented padding was needed.

## Implementation map

Inside `pm290c/`:

- `dist/app.js`: editor, canvas rendering, preview, local draft storage, and controls.
- `dist/printer.js`: Web Bluetooth connection and print handshake.
- `dist/protocol.js`: combined header construction.
- `dist/raster.js`: monochrome conversion and bitmap packing.
- `server.mjs`: dependency-free local HTTP server.
- `PROTOCOL.md`: evidence, interpretation, and validation notes.
- `chrome-success-log.txt`: successful direct Chrome exchange.
- `test/`: automated tests, including a fixture extracted from the real capture.

Inside `pm290c-relay/`:

- `Relay.swift`: native AppKit/CoreBluetooth relay.
- `PM290CRelay.app`: built macOS app.
- `analyze.py`: capture analysis and byte-stream export.
- `captures/`: raw JSONL captures and exported streams.
- `README.md`: operation and rebuild instructions.

## Verification

- Labelnize iOS → Mac relay → real printer: Sudoku physically printed.
- Chrome → real printer, relay closed: small test physically printed and looked correct.
- Chrome received completion at 20:04:56 UTC on October 8, 2026.
- Eleven automated tests passed: captured header bytes, raster encoding, HTTP serving, multi-block allowances, oversized final allowances, missing replies, and radio failure.

Run tests from the SPA directory:

```sh
npm test
```

## Limits and recovery

This version supports monochrome, single-copy stickers up to 150 mm long. Other printer models and firmware versions have not been validated.

The app waits for the printer's completion response before reporting success. It never automatically retries a print. After cancellation or failure during a partial job, power-cycle the printer before retrying to clear buffered data.

Paper-out and temperature status codes remain uncharacterized; nonzero status is reported as raw hex.

The relay is retained for further investigation, but must be closed before Chrome connects directly.
