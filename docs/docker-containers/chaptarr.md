# Chaptarr

Readarr was archived in June 2025 after its metadata backend went offline for good, which left books as the one gap in an otherwise complete *arr stack. Chaptarr fills it: same idea, same UI conventions, but built for audiobooks and ebooks from the start.

[Chaptarr](https://github.com/Chaptarr/chaptarr) is an audiobook and ebook collection manager.

1. **Audiobooks and ebooks in one instance** — no more running two Readarrs side by side
2. **Narrator-aware** — tells two readings of the same book apart, which general-purpose managers ignore
3. **Multi-provider metadata** — resolves books across several sources and aggregates by consensus, so one dead API cannot take it down the way it did Readarr
4. **MP3 → M4B conversion** with chapters preserved
5. **Series handling, renaming and quality upgrades** — the usual *arr feature set
6. **SQLite by default**, optional external PostgreSQL

!!! note "Prerequisites"

    - The [media stack](media-stack.md) layout — Chaptarr slots into it like Sonarr or Radarr
    - A download client: [qBittorrent](qbittorrent.md) or [SABnzbd](sabnzbd.md)
    - [Prowlarr](prowlarr.md) for indexers

## 1. The container

Upstream's example mounts `/audiobooks`, `/ebooks` and `/downloads` separately. Don't — separate mounts break hardlinks, exactly as they would for Radarr. Use the single `/data` mount like the rest of the stack:

```yaml
  chaptarr:
    image: chaptarr/chaptarr:latest
    container_name: chaptarr
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/chaptarr:/config
      - /mnt/user/data:/data
    ports:
      - 8789:8789
```

Create the config directory first, or Docker creates it as `root:root` and Chaptarr cannot write to it:

```bash
sudo mkdir -p /mnt/user/appdata/chaptarr
sudo chown 99:100 /mnt/user/appdata/chaptarr
mkdir -p /mnt/user/data/media/{audiobooks,books}
docker compose up -d chaptarr
```

## 2. Authentication and API key

`http://<host-ip>:8789`. Set up authentication on first run, then copy the API key from **Settings → General** — Prowlarr needs it.

## 3. Root folders

**Settings → Media Management → Root Folders → Add** one per media type:

- `/data/media/audiobooks`
- `/data/media/books`

Start with a small folder if you are importing an existing library. Matching books is harder than matching films, and it is much easier to fix a bad match on twenty titles than on two thousand.

## 4. Download clients

**Settings → Download Clients → +**:

| Field | qBittorrent | SABnzbd |
|---|---|---|
| Host | `qbittorrent` | `sabnzbd` |
| Port | `8080` | `8080` |
| Auth | WebUI user/pass | API key |
| Category | `books` | `books` |

**Host is the container name, never `localhost`.** Inside the Chaptarr container, `localhost` is Chaptarr itself. Both containers must be on the same Docker network.

Create the `books` category in the client too, or downloads land in the wrong folder and never import.

## 5. Indexers

Chaptarr speaks the standard *arr indexer protocol. In Prowlarr, **Settings → Apps → +** and check whether your Prowlarr version lists Chaptarr. If it does, give it `http://chaptarr:8789` and the API key and Prowlarr pushes indexers across like it does for Sonarr.

If it does not, add indexers in Chaptarr directly (**Settings → Indexers → +**, Torznab/Newznab) using each indexer's Prowlarr proxy URL.

Make sure your indexers actually carry book categories. Most general trackers have few audiobooks; this is where private trackers matter.

## 6. Coming from Readarr

There is no import path. Quality profiles, naming and settings have to be rebuilt by hand. The files themselves are fine — point a root folder at the existing library and let Chaptarr match them.

## Updating

```bash
cd /opt/docker/media
docker compose pull chaptarr
docker compose up -d chaptarr
```

Chaptarr is new. Pin a version tag once your library is in it, and back up before upgrading.

## Backup

```bash
docker compose stop chaptarr
sudo tar czf /mnt/user/backups/chaptarr-$(date +%F).tar.gz -C /mnt/user/appdata chaptarr
docker compose start chaptarr
```

Stop it first — SQLite.

## Privacy note

Metadata lookups go to Chaptarr's own service at `api2.chaptarr.com`. Requests can include search text, provider IDs, tags and filenames, but not full paths or credentials. Worth knowing if that matters to you.

## Troubleshooting

**Permission denied on `/config`.** The directory was created by Docker as root. `chown 99:100` it (or your `PUID:PGID`).

**Download client test fails.** Host is set to `localhost`, or the two containers are not on the same network. `docker network inspect <network>` and check both are listed.

**Imports copy instead of hardlink.** Downloads and library are on different mounts. See [the media stack page](media-stack.md#the-one-thing-everyone-gets-wrong-hardlinks).

**Downloads complete but do not import.** Category mismatch between Chaptarr and the client, or permissions on `/data`.

**Wrong edition or narrator matched.** Edit the book and pick the correct edition manually; then rename.

## Where this sits in my lab

<!-- TODO: fill in from your setup -->
