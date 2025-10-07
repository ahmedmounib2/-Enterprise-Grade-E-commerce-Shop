# Certificate pinning assets

Place production certificate bundles here for `react-native-ssl-pinning`.

- Export the public certificate for your API origin (e.g. `api.example.com`) in DER format and save
  it as `<name>.cer` inside this folder.
- List the basename(s) (without the `.cer` suffix) in `EXPO_PUBLIC_SSL_PINNING_CERTS` so the mobile
  app can enable SSL pinning at runtime.
- Commit only non-sensitive placeholder files; never commit private keys.

- Run:

```bash
HOST=api.ahmedmonib-eshop-demo.com
openssl s_client -servername "$HOST" -connect "$HOST:443" -showcerts </dev/null \
  | openssl x509 -outform der -out mobile/certs/eshop_api.cer
```
