# NOM

A single-file browser arcade game: steer a blob with the mouse, eat smaller blobs, grow through stages, pick an upgrade and fight a boss at each new stage.

**Play:** https://nom.pete-85b.workers.dev

## Run locally

Open `index.html` in a browser, or serve the folder with any static server:

```bash
python3 -m http.server
```

## Deploy

Pushes to `master` deploy automatically to Cloudflare Workers via Workers Builds. Only `index.html` is published (see `.assetsignore`).

## License

MIT
