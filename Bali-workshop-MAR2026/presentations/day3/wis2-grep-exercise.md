# wis2-grep install, configure, deploy, initialize

- stop Prometheus, Grafana, scgb services
- ensure traefik still running on your VM (`docker ps | grep traefik`)

```bash
cd

# download wis2-grep via GitHub repository clone
git clone https://github.com/wmo-im/wis2-grep.git
cd wis2-grep

# download required configuration updates
curl –O https://raw.githubusercontent.com/wmo-im/wis2-champions/refs/heads/main/Bali-workshop-MAR2026/presentations/day3/wis2-grep.patch
# apply configuration updates as a git patch
git apply wis2-grep.patch

# update configuration
vi wis2-grep.env
# replace CHANGE_ME with your VM username
vi docker-compose.yml
# replace CHANGE_ME with your VM username

# build wis2-grep
make build

# startup wis2-grep
make up

# Go to https://USER.champion.wis2dev.io/wis2-grep
```
