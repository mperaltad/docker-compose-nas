# Docker Compose NAS

## Quick Start

1. `cp .env.example .env`
1. edit .env to your needs
1. `docker compose up -d`
1. Time to config services

## Useful stuff to configure/do

1. In calibre, go to Preferences > Sharing over the net > Enable the http server (this comes pretty handy for e-ink devices)
1. The indexers are configured through Prowlarr. They synchronize automatically to Radarr and Sonarr.
    1. Radarr and Sonarr may be added via Settings > Apps
    1. Their API keys can be found in Settings > Security > API Key


## Services useful urls to have as bookmarks :)

1. Calibre Server VNC https://your-nas-ip:8181/
1. Calibre Server web https://your-nas-ip:8081/
1. Qbittorrent http://your-nas-ip:8080/
1. Prowlarr http://your-nas-ip:9696/
1. Radarr http://your-nas-ip:7878/
1. Sonarr http://your-nas-ip:8989/
1. Bazarr http://your-nas-ip:6767/
1. Grafana http://your-nas-ip:3000/

