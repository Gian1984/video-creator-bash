# 🎬 Video Shorts Generator (Bash + Python on Linux)

A simple, automated open-source pipeline to generate YouTube Shorts using Bash, FFmpeg, and Python (gTTS) on Ubuntu Linux.

---

## ⚙️ Requirements

Tested on **Ubuntu Linux**.

### 📦 Required Packages

Install all dependencies:

```bash
sudo apt update
sudo apt install -y ffmpeg imagemagick python3-pip python3-venv vlc
```

### 🐍 Python Virtual Environment + gTTS

```bash
python3 -m venv ~/env-tts
source ~/env-tts/bin/activate
pip install gTTS
```

---

## 📁 Project Structure

```
video-shorts/
├── immagini/              # Folder with your images
│   ├── slide1.png
│   ├── slide2.png
│   └── slide3.png
├── musica.mp3             # Royalty-free background music
├── short_generator.sh     # The main Bash script
└── README.md
```

---

## 🧠 How It Works

1. Text is converted to voice using `gTTS`
2. A slideshow video is created from images using `ffmpeg`
3. Background music is mixed in (with volume lowered)
4. Final video includes fade-in/out and voice
5. Output: `short_finale.mp4` ready for YouTube Shorts

---

## 🚀 Usage

Make the script executable and run it:

```bash
chmod +x short_generator.sh
./short_generator.sh
```

---

## ✏️ How to Edit

Edit the script with:

```bash
nano short_generator.sh
```

You can change:
- `TEXT="..."` — the message to be spoken
- The images inside `./immagini/`
- Background music: `musica.mp3`

---

## 🧩 Future Ideas

- Add zoom/pan effects
- Generate subtitles automatically
- Publish directly via YouTube API

---

## 📜 Script: `short_generator.sh`

```bash
#!/bin/bash

source ~/env-tts/bin/activate

TEXT="3 gadget tech da meno di 30 euro che sembrano fantascienza!"
VOCE="voce.mp3"
MUSICA="musica.mp3"
IMAGES_DIR="./immagini"
VIDEO="video.mp4"
OUTPUT="short_finale.mp4"

# Find images
shopt -s nullglob
IMMAGINI=("$IMAGES_DIR"/slide*.jpg "$IMAGES_DIR"/slide*.png)
shopt -u nullglob

NUM_IMMAGINI=${#IMMAGINI[@]}
if [ "$NUM_IMMAGINI" -lt 1 ]; then
  echo "[❌] No images found in $IMAGES_DIR"
  exit 1
fi
echo "[ℹ️] Found $NUM_IMMAGINI images."

# Generate voice
echo "[🎤] Generating voice..."
python3 -c "from gtts import gTTS; gTTS('$TEXT', lang='it').save('$VOCE')"

# Get audio duration
DURATA=$(ffprobe -i "$VOCE" -show_entries format=duration -v quiet -of csv="p=0")
DURATA=$(printf "%.0f" "$DURATA")
DURATA_SLIDE=$(( DURATA / NUM_IMMAGINI ))
RESTO=$(( DURATA % NUM_IMMAGINI ))

echo "[⏱️] Audio duration: $DURATA sec"

# Create video from images
echo "[🖼️] Creating video from images..."
FILTERS=""
INPUTS=()
for i in "${!IMMAGINI[@]}"; do
  DUR=$DURATA_SLIDE
  [ "$i" -eq 0 ] && DUR=$(( DUR + RESTO ))
  INPUTS+=(-loop 1 -t "$DUR" -i "${IMMAGINI[$i]}")
  FILTERS+="[$i:v]scale=1080:1920:force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2:color=black,format=yuv420p[frame$i];"
done

for i in $(seq 0 $((NUM_IMMAGINI - 1))); do
  FILTERS+="[frame$i]"
done
FILTERS+="concat=n=$NUM_IMMAGINI:v=1:a=0[outv]"

ffmpeg -y "${INPUTS[@]}" -filter_complex "$FILTERS" -map "[outv]" -c:v libx264 -pix_fmt yuv420p -r 30 "$VIDEO"

# Mix voice and background music
echo "[🎧] Mixing voice and music..."
ffmpeg -y -i "$VOCE" -i "$MUSICA" -filter_complex "[1:a]volume=0.2[music];[0:a][music]amix=inputs=2:duration=first[aout]" \
-map "[aout]" -c:a aac audio_mix.m4a

# Combine video and audio
echo "[🎬] Generating final short..."
ffmpeg -y -i "$VIDEO" -i audio_mix.m4a -c:v copy -c:a aac -shortest "$OUTPUT"

echo "[✅] Done! Your short is ready: $OUTPUT"
```
