# Local TLS Certificates

Certificate files are **not committed** to this repository (see `.gitignore`).

## Regenerating local TLS certificates

1. Install [mkcert](https://github.com/FiloSottile/mkcert):

   ```
   # macOS
   brew install mkcert

   # Linux (via Homebrew or manual — see mkcert releases)
   brew install mkcert

   # Windows
   choco install mkcert
   ```

2. Install the local CA into your system trust store (run once per machine):

   ```
   mkcert -install
   ```

3. Generate certificates for local development:

   ```
   mkcert -key-file cert/localhost+2-key.pem \
          -cert-file cert/localhost+2.pem \
          localhost 127.0.0.1 ::1
   ```

4. The backend reads these files at startup via `backend/src/server.js` when `NODE_ENV` is not
   `production`. The paths resolve relative to the repo root.

> **Never commit these generated files.** `cert/*.pem` and `cert/*.key` are listed in `.gitignore`.
