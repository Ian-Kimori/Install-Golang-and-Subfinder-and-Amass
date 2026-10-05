# Install-Golang-and-Subfinder-and-Amass

Here's the full sequence in one place, using apt:

```bash
# 1. Clear the dead proxy for this session
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy

# 2. Find where it's set permanently
grep -rn -i proxy /etc/environment ~/.bashrc ~/.zshrc ~/.profile /etc/apt/apt.conf.d/ 2>/dev/null
```

Delete any line pointing at `172.234.27.37:8080` (use `sudo` for files under `/etc`), then open a new terminal and continue:

```bash
# 3. Remove gccgo if you installed it, and refresh the stale package index
sudo apt remove -y gccgo-go 2>/dev/null
sudo apt clean
sudo apt update

# 4. Install Go
sudo apt install -y golang-go
go version                      # should show go1.26.x

# 5. Put Go-installed tools on your PATH (use ~/.zshrc if on zsh)
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# 6. Clear any proxy Go saved, then install subfinder
go env -u HTTP_PROXY HTTPS_PROXY 2>/dev/null
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest

# 7. Verify
subfinder -version
```

If step 4 still returns a 404, the mirror hasn't finished syncing. Wait a few minutes and rerun `sudo apt update` before trying again.

If step 6 times out, bypass the Go module proxy and fetch straight from GitHub:

```bash
GOPROXY=direct go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```
Since Go is already set up from the subfinder install, use the same pattern. Amass is now on v5, and the current v5 command-line binary installs from source with `CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main`.

```bash
# make sure the dead proxy isn't back in this shell
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy

CGO_ENABLED=0 go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
amass -h
```

If the Go module proxy times out again, bypass it:

```bash
CGO_ENABLED=0 GOPROXY=direct go install -v github.com/owasp-amass/amass/v5/cmd/amass@main
```

**Prebuilt binary (no Go needed).** Releases ship as a zip that unpacks into its own folder:

```bash
cd /tmp
wget https://github.com/owasp-amass/amass/releases/latest/download/amass_Linux_amd64.zip
unzip amass_Linux_amd64.zip
sudo mv amass_Linux_amd64/amass /usr/local/bin/
amass -h
```

**Docker**, if you'd rather keep it isolated:

```bash
docker pull owaspamass/amass:latest
docker run --rm -it -v ~/.config/amass:/.config/amass owaspamass/amass:latest enum -d example.com
```

Avoid `sudo apt install amass` on Ubuntu, and the snap package too. Both tend to carry older v3/v4 builds whose commands differ from v5.

Basic usage once installed:

```bash
amass enum -d example.com
amass subs -d example.com -names    # list what's been collected
```

API keys for data sources go in `~/.config/amass/datasources.yaml`. As with subfinder, results are much thinner without them.
