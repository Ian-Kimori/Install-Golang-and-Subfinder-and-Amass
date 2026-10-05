# Install-Golang-and-Subfinder-and-Amass

Here's the full setup in order, with every issue we hit built in.

## 1. Clear the dead proxy

The `172.234.27.37:8080` proxy broke `go install`. Unset it for this session, then find where it's set permanently:

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy
grep -rn -i proxy /etc/environment ~/.bashrc ~/.zshrc ~/.profile /etc/apt/apt.conf.d/ 2>/dev/null
```

Delete any line pointing at that address (use `sudo nano` for files under `/etc`). Then open a new terminal and confirm it's gone:

```bash
env | grep -i proxy    # should print nothing
```

## 2. Install Go

Use `golang-go`, not `gccgo-go`. gccgo is stuck around Go 1.18, which is too old for these tools, and it conflicts with `golang-go`. Refreshing the index fixes the `linux-libc-dev` 404:

```bash
sudo apt remove -y gccgo-go 2>/dev/null
sudo apt clean
sudo apt update
sudo apt install -y golang-go
go version    # should show go1.26.x, not gccgo
```

If you still get a 404, the mirror is mid-sync. Wait a few minutes and rerun `sudo apt update`.

## 3. Put Go's bin directory on PATH

This is what caused `subfinder: command not found`. Use `~/.zshrc` instead if `echo $SHELL` shows zsh:

```bash
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
go env -u HTTP_PROXY HTTPS_PROXY 2>/dev/null   # clear any proxy Go saved itself
```

## 4. Install subfinder

```bash
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
subfinder -version
```

## 5. Install Amass (v5)

```bash
CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
amass -h
```

Skip `apt install amass` and the snap package, since both carry older v3/v4 builds with different commands.

## If `go install` times out

The Go module proxy may be blocked on your network. Fetch straight from GitHub instead:

```bash
GOPROXY=direct go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
CGO_ENABLED=0 GOPROXY=direct go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
```

## Fallback: prebuilt binaries, no Go needed

```bash
cd /tmp

# subfinder
V=$(curl -s https://api.github.com/repos/projectdiscovery/subfinder/releases/latest | grep -oP '"tag_name": "v\K[^"]+')
wget https://github.com/projectdiscovery/subfinder/releases/download/v${V}/subfinder_${V}_linux_amd64.zip
unzip -o subfinder_${V}_linux_amd64.zip subfinder && sudo mv subfinder /usr/local/bin/

# amass
wget https://github.com/owasp-amass/amass/releases/latest/download/amass_Linux_amd64.zip
unzip -o amass_Linux_amd64.zip && sudo mv amass_Linux_amd64/amass /usr/local/bin/
```

## 6. Verify and configure

```bash
which subfinder amass
subfinder -version
amass -h
```

API keys make a big difference to how many subdomains each tool finds:

- **subfinder:** `~/.config/subfinder/provider-config.yaml`, created on first run.
- **amass:** `~/.config/amass/datasources.yaml`.

Quick test:

```bash
subfinder -d example.com -silent
amass enum -d example.com
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `proxyconnect tcp ... i/o timeout` | Proxy env var still set. Redo step 1 in a fresh terminal. |
| `404 Not Found` from apt | Stale index. Run `sudo apt update`, or wait for the mirror to sync. |
| `command not found` after install | PATH missing Go's bin. Redo step 3, or run `~/go/bin/subfinder`. |
| `requires go >= 1.xx` | Go is too old or gccgo is installed. Redo step 2. |
| `proxy.golang.org` timeout | Prefix the install with `GOPROXY=direct`. |
