# SonarQube Setup Guide

Quick setup to enable SonarQube static analysis in repo-intel.

---

## Option A — Docker (recommended for local use)

```bash
# Start SonarQube (Community Edition — free)
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:lts-community

# Wait ~60 seconds, then open http://localhost:9000
# Default credentials: admin / admin (you'll be prompted to change on first login)
```

### Generate an analysis token

1. Log in at http://localhost:9000
2. Go to **My Account → Security → Generate Token**
3. Name it `repo-intel`, type: **Global Analysis Token**
4. Copy the token and export it:

```bash
export SONAR_TOKEN=<your-token-here>
# Add to ~/.bashrc or ~/.zshrc to persist
```

---

## Option B — SonarQube Cloud (sonarcloud.io)

```bash
export SONAR_TOKEN=<sonarcloud-token>
# Then edit sonar-project.properties.template:
# Change: sonar.host.url=http://localhost:9000
# To:     sonar.host.url=https://sonarcloud.io
# Add:    sonar.organization=<your-org-key>
```

---

## Install sonar-scanner CLI

### macOS
```bash
brew install sonar-scanner
```

### Linux
```bash
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-6.2.1.4610-linux-x64.zip
unzip sonar-scanner-cli-6.2.1.4610-linux-x64.zip
sudo mv sonar-scanner-6.2.1.4610-linux-x64 /opt/sonar-scanner
echo 'export PATH="/opt/sonar-scanner/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Windows
```powershell
winget install SonarSource.SonarScanner
# Or download from https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/scanners/sonarscanner/
```

### Verify
```bash
sonar-scanner --version
```

---

## Verify everything works

```bash
# Check server is up
curl -s http://localhost:9000/api/system/status | python3 -c "import json,sys; print(json.load(sys.stdin))"

# Expected output: {"status":"UP","version":"..."}
```

---

## Supported languages (Community Edition)

| Language | Detection |
|----------|-----------|
| JavaScript / TypeScript | `package.json` present |
| Python | `requirements.txt` or `pyproject.toml` |
| Java | `pom.xml` or `build.gradle` |
| C# | `*.csproj` |
| Go | `go.mod` |
| PHP | `*.php` files |
| Ruby | `Gemfile` |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `sonar-scanner: command not found` | Install via brew/winget or add to PATH |
| Server returns 503 | Wait longer — SonarQube takes ~60s to start |
| `401 Unauthorized` | Check `SONAR_TOKEN` is exported correctly |
| `Project not found` | First run creates it automatically — retry |
| Java heap error in scanner | `export SONAR_SCANNER_OPTS="-Xmx512m"` |
| Docker container exits immediately | Check `docker logs sonarqube` — may need `vm.max_map_count=262144` |

```bash
# Fix vm.max_map_count on Linux
sudo sysctl -w vm.max_map_count=262144
```
