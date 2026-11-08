# hello-tauri

Tauri 2 + React + TypeScript (Vite) のデスクトップアプリ。Windows (x64) 向けバイナリを GitHub Actions でビルドする。

## 開発環境 (WSL2)

ツールのバージョンは [mise](https://mise.jdx.dev/) (`mise.toml`: Node 24 / pnpm 12) と rustup (stable) で管理する。

```sh
# Tauri の Linux 依存ライブラリ (初回のみ)
sudo apt update
sudo apt install -y libwebkit2gtk-4.1-dev build-essential curl wget file \
  libxdo-dev libssl-dev libayatana-appindicator3-dev librsvg2-dev

mise install
rustup update stable
pnpm install
pnpm tauri dev   # WSLg で Linux 版のウィンドウが開く
```

Windows 版はローカルでは作らず、CI でビルドする。

## CI

| ワークフロー | トリガー | 内容 |
|---|---|---|
| `ci.yml` | Pull Request | Linux で `tsc` + `vite build`、`cargo fmt --check`、`cargo clippy -D warnings` |
| `build-windows.yml` | `main` への push (PR マージ) / 手動 | Windows で NSIS (`.exe`) と MSI をビルドして署名し、Actions Artifact (保持 7 日) にアップロード |

## コード署名 (自己署名)

学習用の自己署名証明書で署名している。信頼されたルートに入っていないため、SmartScreen の「発行元不明」警告はそのまま出る。

GitHub Secrets:

- `WINDOWS_CERTIFICATE`: `.pfx` を base64 化したもの
- `WINDOWS_CERTIFICATE_PASSWORD`: `.pfx` のパスワード

Secret が無い場合は署名なしでビルドする。

### 証明書の再作成

`umask` と `cd` を手元のシェルに残さないよう、サブシェルで実行する。

```sh
(
D=~/.secrets/hello-tauri; mkdir -p "$D"; chmod 700 ~/.secrets "$D"; cd "$D"; umask 077
openssl rand -base64 24 | tr -d '\n' > pfx-password.txt
openssl req -x509 -newkey rsa:3072 -sha256 -days 1095 -nodes -keyout key.pem -out cert.pem \
  -subj "/CN=hello-tauri (virtual-hippo self-signed)" \
  -addext "keyUsage=critical,digitalSignature" -addext "extendedKeyUsage=codeSigning" \
  -addext "basicConstraints=critical,CA:FALSE"
openssl pkcs12 -export -inkey key.pem -in cert.pem -out cert.pfx -passout file:pfx-password.txt \
  -keypbe AES-256-CBC -certpbe AES-256-CBC -macalg sha256
openssl x509 -in cert.pem -outform der -out cert.cer   # 公開鍵のみ (信頼登録する場合に使う)

base64 -w0 cert.pfx | gh secret set WINDOWS_CERTIFICATE -R virtual-hippo/hello-tauri2
gh secret set WINDOWS_CERTIFICATE_PASSWORD -R virtual-hippo/hello-tauri2 < pfx-password.txt
)
```

`key.pem` は暗号化されていない秘密鍵で、`cert.pfx` と `pfx-password.txt` が揃っても秘密鍵を取り出せる。WSL のバックアップや export にはこれらも含まれる。証明書は作り直せるので、Secret の登録後にローカルで保管する必要がなければ削除する。

```sh
rm ~/.secrets/hello-tauri/{key.pem,cert.pfx,cert.pfx.b64,pfx-password.txt}
```
