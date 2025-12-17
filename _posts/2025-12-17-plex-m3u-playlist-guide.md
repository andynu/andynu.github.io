---
layout: post
title:  "Importing M3U Playlists to Plex & PlexAmp"
date:   2025-12-17
categories: plex
---

**A working solution as of December 2025**

This guide documents a reliable method for importing M3U playlists into Plex so they appear in PlexAmp and other Plex clients. This approach uses Plex's hidden `/playlists/upload` API endpoint.

## Background

For years, Plex users have been asking for native M3U playlist support. The [forum thread "Can Plexamp read and use m3u playlists?"](https://forums.plex.tv/t/can-plexamp-read-and-use-m3u-playlists/234179/97) has been ongoing since 2018 with various workarounds proposed. This guide documents an approach that works reliably in late 2024/2025.

## What You Need

1. **Plex Media Server** running with a Music library
2. **Your Plex Token** (authentication)
3. **Your Music Library Section ID**
4. **M3U playlist files** with relative paths to your audio files
5. **bash**, **curl**, and **python3** (for URL encoding)

## Finding Your Plex Token

Your Plex token is stored in the Plex preferences file:

```bash
# Linux (typical location)
grep -oP 'PlexOnlineToken="\K[^"]+' \
  "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml"

# macOS
grep -oP 'PlexOnlineToken="\K[^"]+' \
  "$HOME/Library/Application Support/Plex Media Server/Preferences.xml"
```

The token looks something like: `xYz123ABCdef456GHI`

## Finding Your Music Library Section ID

Query your Plex server for library sections:

```bash
curl -s "http://127.0.0.1:32400/library/sections?X-Plex-Token=YOUR_TOKEN" | grep -oP 'key="\K[0-9]+'
```

Or visit `http://127.0.0.1:32400/library/sections?X-Plex-Token=YOUR_TOKEN` in a browser and look for your Music library's `key` attribute. It's typically a single digit like `1`, `2`, `3`, or `4`.

## The Hidden Upload Endpoint

Plex has an undocumented endpoint for uploading M3U playlists:

```
POST /playlists/upload?sectionID={SECTION_ID}&path={URL_ENCODED_PATH}&X-Plex-Token={TOKEN}
```

**Important notes:**
- The `path` must be the **absolute filesystem path** to the M3U file
- The path must be **URL encoded**
- The M3U file must be readable by the Plex server
- Paths inside the M3U should be **relative to the M3U file location**

## M3U Playlist Format

M3U files should use relative paths and start with `#EXTM3U`:

```m3u
#EXTM3U
Artist/Album/01 - Track One.mp3
Artist/Album/02 - Track Two.mp3
Another Artist/Song.flac
```

## Creating M3U Playlists

Here's a bash function to create an M3U playlist for all audio files in a folder:

```bash
#!/bin/bash
AUDIO_EXTENSIONS="mp3|MP3|flac|FLAC|m4a|M4A|wav|WAV|ogg|OGG|opus|OPUS|wma|WMA|aac|AAC|ape|APE|aiff|AIFF"

create_playlist() {
    local folder="$1"
    local playlist_name="$2"
    local playlist_path="$folder/${playlist_name}.m3u"

    # Find all audio files recursively, sorted
    audio_files=$(find "$folder" -type f 2>/dev/null | grep -iE "\.($AUDIO_EXTENSIONS)$" | sort)

    if [ -n "$audio_files" ]; then
        echo "#EXTM3U" > "$playlist_path"

        while IFS= read -r file; do
            # Convert to relative path
            rel_path=$(realpath --relative-to="$folder" "$file")
            echo "$rel_path" >> "$playlist_path"
        done <<< "$audio_files"

        file_count=$(echo "$audio_files" | wc -l)
        echo "Created: $playlist_name ($file_count tracks)"
    fi
}

# Example usage:
create_playlist "/path/to/music/Jazz" "Jazz"
```

## Uploading to Plex

```bash
#!/bin/bash
PLEX_URL="http://127.0.0.1:32400"
PLEX_TOKEN="your_token_here"
SECTION_ID="4"  # Your music library section ID

upload_to_plex() {
    local playlist_path="$1"
    local playlist_name="$2"

    if [ -f "$playlist_path" ]; then
        # URL encode the path
        encoded_path=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$playlist_path', safe=''))")

        curl -s -X POST \
            "${PLEX_URL}/playlists/upload?sectionID=${SECTION_ID}&path=${encoded_path}&X-Plex-Token=${PLEX_TOKEN}"

        echo "Uploaded: $playlist_name"
    fi
}

# Example usage:
upload_to_plex "/path/to/music/Jazz/Jazz.m3u" "Jazz"
```

## Deleting Old Playlists

When regenerating playlists, you may want to delete old uploaded versions first. Plex assigns each playlist a `ratingKey`. Uploaded playlists typically have high IDs (>200000), while original smart playlists have lower IDs.

```bash
#!/bin/bash
PLEX_URL="http://127.0.0.1:32400"
PLEX_TOKEN="your_token_here"

# Get all playlist IDs and delete those with high ratingKeys
curl -s "${PLEX_URL}/playlists?X-Plex-Token=${PLEX_TOKEN}" | \
    grep -oP 'ratingKey="\K[0-9]+' | while read id; do
    if [ "$id" -gt 200000 ]; then
        curl -s -X DELETE "${PLEX_URL}/playlists/$id?X-Plex-Token=${PLEX_TOKEN}"
        echo "Deleted playlist $id"
    fi
done
```

## Complete Regeneration Script

Here's a complete script that regenerates and uploads playlists:

```bash
#!/bin/bash
#
# Regenerate all music playlists and upload to Plex
# Run this whenever you add or move music files
#

PLEX_URL="http://127.0.0.1:32400"
PLEX_TOKEN="your_token_here"
SECTION_ID="4"  # Music library section ID
AUDIO_EXTENSIONS="mp3|MP3|flac|FLAC|m4a|M4A|wav|WAV|ogg|OGG|opus|OPUS|wma|WMA|aac|AAC|ape|APE|aiff|AIFF"

echo "=== Regenerating Music Playlists ==="

# Function to create a playlist
create_playlist() {
    local folder="$1"
    local playlist_name="$2"
    local playlist_path="$folder/${playlist_name}.m3u"

    audio_files=$(find "$folder" -type f 2>/dev/null | grep -iE "\.($AUDIO_EXTENSIONS)$" | sort)

    if [ -n "$audio_files" ]; then
        echo "#EXTM3U" > "$playlist_path"
        while IFS= read -r file; do
            rel_path=$(realpath --relative-to="$folder" "$file" 2>/dev/null || echo "$file")
            echo "$rel_path" >> "$playlist_path"
        done <<< "$audio_files"
        file_count=$(echo "$audio_files" | wc -l)
        echo "  Created: $playlist_name ($file_count tracks)"
        return 0
    fi
    return 1
}

# Function to upload a playlist
upload_to_plex() {
    local playlist_path="$1"
    local playlist_name="$2"

    if [ -f "$playlist_path" ]; then
        encoded_path=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$playlist_path', safe=''))")
        curl -s -X POST "${PLEX_URL}/playlists/upload?sectionID=${SECTION_ID}&path=${encoded_path}&X-Plex-Token=${PLEX_TOKEN}" > /dev/null
        echo "  Uploaded: $playlist_name"
    fi
}

# -----------------------------
# STEP 1: Create playlists for each folder you want
# -----------------------------
echo "--- Creating playlists ---"

# Add your music folders here
MUSIC_FOLDERS=(
    "/path/to/music/Jazz"
    "/path/to/music/Classical"
    "/path/to/music/Rock"
)

for folder in "${MUSIC_FOLDERS[@]}"; do
    if [ -d "$folder" ]; then
        folder_name=$(basename "$folder")
        create_playlist "$folder" "$folder_name"
    fi
done

# -----------------------------
# STEP 2: Delete old uploaded playlists
# -----------------------------
echo "--- Cleaning up old Plex playlists ---"

curl -s "${PLEX_URL}/playlists?X-Plex-Token=${PLEX_TOKEN}" | grep -oP 'ratingKey="\K[0-9]+' | while read id; do
    if [ "$id" -gt 200000 ]; then
        curl -s -X DELETE "${PLEX_URL}/playlists/$id?X-Plex-Token=${PLEX_TOKEN}" > /dev/null
    fi
done
echo "  Removed old uploaded playlists"

# -----------------------------
# STEP 3: Upload fresh playlists
# -----------------------------
echo "--- Uploading to Plex ---"

for folder in "${MUSIC_FOLDERS[@]}"; do
    if [ -d "$folder" ]; then
        folder_name=$(basename "$folder")
        playlist="${folder}/${folder_name}.m3u"
        upload_to_plex "$playlist" "$folder_name"
    fi
done

echo "=== Done! ==="
```

## Troubleshooting

### "localhost" doesn't work
Use `127.0.0.1` instead of `localhost` in the Plex URL.

### Playlists upload but are empty
- Ensure the M3U file uses **relative paths** from its own location
- Verify the audio files exist and are already indexed in your Plex library
- Check that Plex can read the M3U file (permissions)

### Permission denied errors
Some folders may be read-only. The scripts handle this gracefully by continuing to the next folder.

### Finding which playlists to keep
Original Plex smart playlists (like "Recently Added", "Recently Played") have low ratingKey IDs. Check your playlists before bulk deleting:

```bash
curl -s "http://127.0.0.1:32400/playlists?X-Plex-Token=YOUR_TOKEN" | \
    grep -oP '(ratingKey|title)="[^"]+"' | paste - -
```

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/playlists?X-Plex-Token={TOKEN}` | GET | List all playlists |
| `/playlists/upload?sectionID={ID}&path={PATH}&X-Plex-Token={TOKEN}` | POST | Upload M3U playlist |
| `/playlists/{ratingKey}?X-Plex-Token={TOKEN}` | DELETE | Delete a playlist |
| `/library/sections?X-Plex-Token={TOKEN}` | GET | List library sections |

## Automation

Add the regeneration script to a cron job to keep playlists updated:

```bash
# Run every Sunday at 3am
0 3 * * 0 /path/to/music/regenerate-playlists.sh >> /var/log/plex-playlists.log 2>&1
```

Or run manually whenever you add or reorganize music files.

---

*This guide was created in December 2024 and confirmed working with Plex Media Server. The `/playlists/upload` endpoint is undocumented and may change in future Plex versions.*
