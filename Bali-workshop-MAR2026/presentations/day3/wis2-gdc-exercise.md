# wis2-gdc install, configure, deploy, initialize


- stop Prometheus, Grafana, scgb services
- ensure traefik still running on your VM (`docker ps | grep traefik`)

```bash
cd

# download wis2-gdc via GitHub repository clone
git clone https://github.com/wmo-im/wis2-gdc.git
cd wis2-gdc

# download required configuration updates
curl –O https://raw.githubusercontent.com/wmo-im/wis2-champions/refs/heads/main/Bali-workshop-MAR2026/presentations/day3/wis2-gdc.patch
# apply configuration updates as a git patch
git apply wis2-gdc.patch

# update configuration
vi wis2-gdc.env
# replace CHANGE_ME with your VM username
vi docker-compose.yml
# replace CHANGE_ME with your VM username

# build wis2-gdc
make build

# startup wis2-gdc
make up

# Go to https://USER.champion.wis2dev.io/wis2-gdc
# How many records are in your GDC?

# login to the management container and load a metadata archive::
make login
curl -o /tmp/ca-eccc-msc-gdc-archive.zip https://wis2-gdc.weather.gc.ca/wis2-discovery-metadata-archive.zip
wis2-gdc restore /tmp/ca-eccc-msc-gdc-archive.zip

# Go to https://USER.champion.wis2dev.io/wis2-gdc
# How many records are in your GDC?
```
