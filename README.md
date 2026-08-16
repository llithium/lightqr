# LightQR

A fast, browser-based QR code generator and scanner built with SvelteKit and [`qr`](https://www.npmjs.com/package/qr).

**Live app:** <https://lightqr.vercel.app>

## Features

### Generate QR codes

- Encode text or URLs into QR codes
- Export as **PNG**, **JPEG**, **WebP**, or **SVG**
- Choose raster output sizes from **25 px to 1000 px**
- Preview the exact output file size before downloading
- Select error correction levels:
  - Low — 7%
  - Medium — 15%
  - Quartile — 25%
  - High — 30%
- Keep format, size, and error-correction settings in the URL
- Smooth preview updates while changing raster size

### Scan QR codes

- Decode QR codes from image files
- Click, drag and drop, or paste an image from the clipboard
- Supports SVG QR images as well as raster images
- Copy decoded content to the clipboard
- Open decoded HTTP/HTTPS links directly

QR generation and decoding are handled in the browser.

## Tech Stack

- [SvelteKit](https://svelte.dev/docs/kit)
- [Svelte 5](https://svelte.dev/)
- [`qr`](https://www.npmjs.com/package/qr)
- [Tailwind CSS](https://tailwindcss.com/)
- [Bits UI](https://bits-ui.com/)
- [Lucide](https://lucide.dev/)
- [Vercel](https://vercel.com/)

## Development

This project uses a Bun lockfile.

```bash
bun install
bun run dev
```

Then open the local URL printed by Vite.

### Useful commands

```bash
bun run build
bun run check
bun run lint
bun run test
```

## Deployment

LightQR is configured with the SvelteKit Vercel adapter and can be deployed directly to Vercel.
