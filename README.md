# Bookshelf Capture Agent

A Windows voice assistant that guides you through photographing every bookshelf in a room. It watches a live camera feed, talks to you, listens to your replies, and saves sharp, readable photos of each shelf into an output folder. A later stage (not in this program) reads the book spines from those photos.

You talk, Gemini looks at the camera and answers by voice, and the app saves only the frames that pass its own quality checks.

> **Platform:** Windows 10/11, 64-bit only.
> **Internet:** required (the camera frames are sent to Google Gemini).

---

## What it does

1. Connects to a live camera over RTSP.
2. Runs a YOLO model on every frame and draws green boxes on the books (monitor window only).
3. Every few seconds, or when you speak, sends the newest frame plus sensor readings to Gemini.
4. Gemini decides what to say: move closer, hold still, step left, and so on. The reply is spoken aloud.
5. When a frame looks good, Gemini proposes a save. You answer "yes" or "no" by voice.
6. The app saves the frame as `shelf_1_1.jpg`, `shelf_1_2.jpg`, `shelf_2_1.jpg`, and so on.

### Rules enforced by the code (not by Gemini)

Gemini only proposes. The code refuses a save when:

- the image is blurry (sharpness below 50),
- the image is too dark (brightness below 45),
- the spines are too small to read later (median spine length below 300 px),
- the view is nearly identical to the last saved image.

Only the person can end a shelf or end the session.

---

## How the pieces fit together

```
 Camera (RTSP)
      |
      v
 YOLO (obj detection) --> green boxes, book count, spine size
      |
      v
 Sensors: sharpness, brightness, glare, spine px
      |
      +----------------------+
      v                      |
 Gemini (cloud) <--- your speech (Whisper -> text)
      |
      v
 reply text --> KittenTTS --> speaker
      |
      v
 propose save --> code quality checks --> you say yes --> JPG saved
```

---

## Which model does what

| File | Used by this version? | Job |
|---|---|---|
| `model\yolo.onnx` | **Yes** | Book detector. Finds each book in every frame, draws green boxes, counts books, measures spine size. |
| `model\obj_yolo.onnx` | No (reserved) | Non-book object detector. Passed as the 6th argument but not used in this stage. |
| `model\voice\ggml-base.bin` | **Yes** | Whisper speech-to-text. Turns your voice into text. |
| `model\voice\ggml-silero-v6.2.0.bin` | **Yes (automatic)** | Silero voice-activity detector. Whisper uses it automatically when it sits next to `ggml-base.bin`. |
| `model\voice\kitten-nano-en-v0_8-int8\` | **Yes** | KittenTTS text-to-speech (through sherpa-onnx). Speaks Gemini's replies. Contains `model.int8.onnx`, `voices.bin`, `tokens.txt`, `espeak-ng-data\`. |
| `model\voice\en_US-lessac-medium.onnx` + `.json` | No (old) | Piper voice from an earlier version. Safe to delete. |
| `model\voice\espeak-ng-data\`, `lexicon-en-us.txt` | No (old) | Leftovers from the Piper setup. Kitten uses the `espeak-ng-data` inside its own folder. |
| `model\GGUF\VL.gguf`, `mmproj.gguf` | No (reserved) | A local vision-language model. Passed as the 1st argument but not used in this stage. |
| Google Gemini (cloud) | **Yes** | The agent brain. Looks at the frame, talks to you, decides what to ask and when to propose a save. |

---

## Folder layout

```
Book Shelf\exe\
|-- build\Release\
|   |-- Bookshelf.exe          <- the program
|   `-- *.dll                  <- every required DLL (must stay next to the exe)
`-- model\
    |-- yolo.onnx
    |-- obj_yolo.onnx
    |-- GGUF\
    |   |-- VL.gguf
    |   `-- mmproj.gguf
    `-- voice\
        |-- ggml-base.bin
        |-- ggml-silero-v6.2.0.bin
        `-- kitten-nano-en-v0_8-int8\
            |-- model.int8.onnx
            |-- voices.bin
            |-- tokens.txt
            |-- espeak-ng-data\
            |-- LICENSE
            `-- README.md
```

---

## Requirements

- Windows 10 or 11, 64-bit.
- A working **microphone** and **speakers or headphones**.
- An **RTSP camera stream** reachable from the PC (for example a phone running an RTSP camera app).
- A **Google Gemini API key** from Google AI Studio.
- A CPU that can run YOLO at a reasonable frame rate. No GPU is needed.
- The Microsoft Visual C++ Redistributable (x64), if Windows reports a missing `VCRUNTIME` or `MSVCP` DLL.

---

## Setup

1. **Download** the repository (or the release zip). It contains `Bookshelf.exe` and all the DLL files next to it.
2. **Download the models** and put them in the folders shown in the layout above.
3. **Get a Gemini API key** from Google AI Studio.
4. **Start your camera stream** and note its RTSP URL, for example `rtsp://192.168.0.100:8554/`.
5. Make sure the PC and the camera are on the same network.

---

## Running

Open PowerShell and run:

```powershell
& "D:\My Start Up\Book Shelf\exe\build\Release\Bookshelf.exe" `
  "D:\My Start Up\Book Shelf\exe\model\GGUF" `
  "D:\My Start Up\Book Shelf\exe\model\yolo.onnx" `
  "rtsp://192.168.0.100:8554/" `
  "C:\Users\sagar\Downloads\test\outputs" `
  "YOUR_GEMINI_API_KEY" `
  "D:\My Start Up\Book Shelf\exe\model\obj_yolo.onnx" `
  "D:\My Start Up\Book Shelf\exe\model\voice"
```

Keep the quotes, because the paths contain spaces. Never commit your real API key to GitHub.

### The 7 arguments, in order

| # | Argument | Meaning |
|---|---|---|
| 1 | `model_path` | The `GGUF` folder. Not used in this stage. |
| 2 | `yolo_model_path` | `yolo.onnx`, the book detector. |
| 3 | `rtsp_feed` | RTSP URL of the camera. |
| 4 | `output_dir` | Where photos and logs are saved. Created if missing. |
| 5 | `gemini_api` | Your Gemini API key. |
| 6 | `yolo_obj` | `obj_yolo.onnx`. Not used in this stage. |
| 7 | `voice_dir` | The `voice` folder with `ggml-base.bin` and the Kitten folder. |

### What you should see at startup

```
[voice] speech output ready (Kitten, 24000 Hz)      <- and you hear "Voice ready."
[Gemini] preflight OK: model 'gemini-3.5-flash-lite' is available.
[voice] microphone is listening
[RTSP] Connected.
```

Gemini then speaks first and asks which shelf this is.

---

## How to use it

1. **Shelf number.** Gemini asks "Is this shelf 1?" once. Say "yes", or say a number ("shelf three"). If you stay silent for 12 seconds, the suggested number is assumed.
2. **Frame the shelf.** Step back so the whole bookcase fits, or follow Gemini's multi-shot plan for big shelves (top band first, then slide right, using a thick, easy-to-spot book as a landmark).
3. **Follow the coaching.** Gemini says things like "hold still", "closer, so every spine is readable", "tilt the camera, there is glare".
4. **Save.** When the view is good, Gemini asks "Save this one?" Say **yes** or **no**. Say "always" to save automatically from then on.
5. **Next shelf.** After the last image of a shelf, Gemini moves on to the next one. Say "this shelf is finished" to end a shelf early, or "skip this shelf".
6. **Finish.** Say "no more shelves" or "end session". You can also type `/quit`, or press `q` or Esc in the video window.

Typing in the terminal also works as a fallback when the microphone is not available. Click the terminal window first, then type and press Enter.

### Voice commands

| Say | Effect |
|---|---|
| "yes" / "save" / "okay" | Confirm a save or the shelf number |
| "no" / "retake" / "skip" | Decline a save |
| "always" / "save everything" | Switch to automatic saving |
| "shelf three" | Set the shelf number |
| "this shelf is finished" | End the current shelf |
| "no more shelves" | End the session |
| "end session" / "quit the program" | Stop the program |

---

## Output files

Everything is written to the output folder (argument 4):

| File | Content |
|---|---|
| `shelf_1_1.jpg`, `shelf_1_2.jpg`, `shelf_2_1.jpg`, ... | Saved shelf photos. Clean frames with no boxes drawn. |
| `session_log.json` | Every saved image: shelf, file name, sharpness, time. |
| `agent_log.jsonl` | One line per Gemini turn: timing, sensor values, message. No images and no API key. |

---

## Optional environment variables

Set them before running, for example `$env:AUTO_SAVE = "1"` in PowerShell.

| Variable | Default | Effect |
|---|---|---|
| `GEMINI_MODEL` | `gemini-3.5-flash-lite` | Gemini model id to use |
| `GEMINI_IMAGE_SIDE` | `640` | Longest side of images sent to Gemini (512 to 640) |
| `TICK_SEC` | `3` | Seconds between automatic checks (minimum 2) |
| `AUTO_SAVE` | off | `1` = save every good frame without asking |
| `MIN_SPINE_PX` | `300` | Smallest median spine length the app accepts |
| `NO_WINDOW` | off | `1` = terminal only, no video window |
| `KITTEN_SPEAKER` | `0` | Voice number, 0 to 7 |
| `KITTEN_DIR` | `<voice_dir>\kitten-nano-en-v0_8-int8` | Override the Kitten folder |
| `MIC_DEVICE` | default mic | Microphone number |
| `LIST_MICS` | off | `1` = print the list of microphones |
| `VOICE_DEBUG` | off | `1` = print voice-layer details |
| `GEMINI_IPV4` | off | `1` = force IPv4 (helps if connections hang) |
| `CURL_VERBOSE` | off | `1` = print network debug output |
| `SHELF_PROMPT_FILE` | built-in | Path to a text file with your own Gemini instructions |

---

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `speech output NOT available` | Wrong `voice_dir`, or the Kitten folder is missing a file. The error names the missing file. Check that `kitten-nano-en-v0_8-int8` holds `model.int8.onnx`, `voices.bin`, `tokens.txt` and `espeak-ng-data`. |
| No sound, but no error | Set the right speakers as the Windows default playback device. Run with `VOICE_DEBUG=1` to see the device name. |
| `speech input NOT available` | `ggml-base.bin` is missing from `voice_dir`, or no microphone was found. Typing still works. |
| It does not hear you | Run with `LIST_MICS=1` and set `MIC_DEVICE`. Check the Windows microphone permission. |
| It does not hear you while it speaks | By design: the mic is muted while it talks, so you can't talk over it. |
| Missing DLL on start | The DLLs must sit next to `Bookshelf.exe`. Install the Visual C++ Redistributable (x64). |
| `API version ... not available` | Two different `onnxruntime.dll` versions are in conflict. Keep the newer one next to the exe. |
| `Cannot open stream` | Wrong RTSP URL, camera not running, or different network. Test the URL in VLC. |
| Gemini preflight fails (HTTP 400/401/403) | The API key is wrong or the API is not enabled for it. |
| HTTP 404 on the model | Set `GEMINI_MODEL` to a valid model id. |
| HTTP 429 | Quota or rate limit reached. Wait, or raise `TICK_SEC`. |
| Slow or stalled requests | Read the `[timing:...]` line. It shows where time was lost. Try `GEMINI_IPV4=1`. |
| Typing does nothing | Click the terminal window before typing. If the video window has focus, `q` or Esc closes the program. |
| Always refused: "move closer" | Lower `MIN_SPINE_PX`, or move closer to the shelf. |

---

## Privacy

- Camera frames (resized, up to 640 px) are sent to Google Gemini during the session. Do not point the camera at people or at documents with personal data.
- Your voice is processed **locally** by Whisper. Only the recognised text goes to Gemini.
- Saved photos stay on your PC in the output folder.
- Keep the API key out of the repository and out of screenshots.

---

## Current limits

- Windows only. The code uses Windows APIs and prebuilt Windows libraries.
- English only for speech in and out.
- Quality checks (sharpness, brightness, spine size) use fixed thresholds. Tune them in `main.cpp` for your camera.
- `obj_yolo.onnx` and the `GGUF` models are loaded or passed but not used yet. This stage only captures images. Reading spines and pricing happen in a later stage.
- The built-in license check expires on 2027-01-31.

---

## Third-party components

- [whisper.cpp](https://github.com/ggerganov/whisper.cpp): speech-to-text
- [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) with KittenTTS: text-to-speech
- [ONNX Runtime](https://onnxruntime.ai/): runs the YOLO model
- [OpenCV](https://opencv.org/): video, image processing
- [libcurl](https://curl.se/libcurl/): talks to the Gemini API
- [miniaudio](https://miniaud.io/): microphone and speaker access
- [nlohmann/json](https://github.com/nlohmann/json): JSON
- Google Gemini API: the agent
