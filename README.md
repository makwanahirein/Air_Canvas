# Air Canvas

Draw in the air with your webcam. A finger becomes the pen; MediaPipe tracks the hand; OpenCV composites strokes onto a live camera feed.

This is a single-file Python app (`air_canvas.py`) using **MediaPipe Tasks Hand Landmarker** (video mode) and **OpenCV**.

## What you get

- Live mirrored webcam window (`Air Canvas`)
- Draw with the **index finger**
- Pick colors from a top palette with **index + middle**
- Erase with a thick black stroke
- Clear the canvas with `C`, quit with `Q`

No mouse, no tablet — only camera, lighting, and a reasonably stable hand.

## How it works

Each frame follows the same path:

```
webcam  →  flip (mirror)  →  MediaPipe hand landmarks
                ↓
         gesture → DRAW or SELECT
                ↓
         strokes written on an off-screen canvas
                ↓
         canvas blended onto the camera frame  →  imshow
```

**Tracking.** MediaPipe’s `HandLandmarker` runs in `RunningMode.VIDEO` with one hand. Confidence thresholds are `0.6` for detection, presence, and tracking. Landmarks are 21 3D points in normalized image space; the app uses the 2D `(x, y)` of the **index tip** (landmark 8) as the cursor.

**Gestures.** A finger is “up” if its tip is above its PIP joint (smaller `y` in image coordinates):


| Finger | Tip | PIP | Meaning                            |
| ------ | --- | --- | ---------------------------------- |
| Index  | 8   | 6   | Cursor / draw when only this is up |
| Middle | 12  | 10  | Combined with index → select mode  |


- **DRAW:** index up, middle down → line from previous tip to current tip (thickness 8, or 40 for eraser).
- **SELECT:** index and middle up → no drawing; if the tip is in the top 60px, that palette slot becomes the current color.
- **NONE:** any other pose, or no hand → previous point resets so the next stroke does not jump.

**Canvas.** Strokes live on a black `numpy` buffer the same size as the frame. Overlay uses a mask: non-black canvas pixels replace the camera pixels (`threshold` + `bitwise_and` / `bitwise_or`). Black eraser strokes punch holes back to the live video.

**Palette (BGR).** Purple, Blue, Green, Yellow, Eraser (black). The window is mirrored (`cv2.flip(..., 1)`) so moving your hand left moves the cursor left.

## Requirements

- Python 3.10+ (3.10 is what the project venv uses)
- A webcam (default device `0`)
- macOS / Linux / Windows with a desktop so OpenCV can show a window
- Camera permission for the terminal or IDE that launches Python

Python packages:

```text
mediapipe
opencv-python
numpy
```

You also need Google’s **Hand Landmarker** model file next to the script:

```text
hand_landmarker.task
```



## Setup

From the repo root:

```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install mediapipe opencv-python numpy
```

Download the official float16 model (required; it is not in git):

```bash
curl -L -o hand_landmarker.task \
  "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task"
```

On macOS, grant **Camera** access to Terminal (or iTerm / VS Code / Cursor) under **System Settings → Privacy & Security → Camera**.

## Run

```bash
source venv/bin/activate
python air_canvas.py
```

The process stays in the foreground until you press `Q` or close the window.

## Controls


| Input             | Action                                            |
| ----------------- | ------------------------------------------------- |
| Index finger only | Draw (or erase if Eraser is selected)             |
| Index + middle    | Selection mode; hover the top bar to change color |
| `C`               | Clear canvas                                      |
| `Q`               | Quit                                              |


Tips that actually help:

- Sit in even light; backlighting and heavy motion blur kill landmarks.
- Keep the hand fully in frame; the model is configured for **one** hand.
- Raise the middle finger clearly when selecting — a half-bent finger still counts as down.
- Start a stroke after a pause; lifting the index resets `prev_x/prev_y` so lines do not connect across gaps.
- Eraser is a wide black stroke, not an undo stack.



## Project layout

```text
air-canvas/
├── air_canvas.py           # entire app
├── hand_landmarker.task    # download locally (not committed)
├── README.md
└── .gitignore
```

The script loads the model from the **current working directory**. Run it from the project root so `hand_landmarker.task` resolves.

## Tuning (in `air_canvas.py`)


| Setting                         | Where                         | Effect                                                                        |
| ------------------------------- | ----------------------------- | ----------------------------------------------------------------------------- |
| `min_*_confidence = 0.6`        | `HandLandmarkerOptions`       | Lower if the hand drops out; higher if the cursor jitters on false detections |
| `num_hands=1`                   | same                          | Second hand is ignored                                                        |
| Palette height `60`             | `draw_palette` / select check | Taller bar is easier to hit                                                   |
| Draw thickness `8` / erase `40` | draw branch                   | Stroke weight                                                                 |
| `frame_timestamp_ms += 33`      | video loop                    | ~30 FPS timestamps for the video API; keep timestamps strictly increasing     |




## Troubleshooting

**Window never appears / script exits immediately.** Camera failed (`VideoCapture(0)`). Close Zoom, Meet, Photo Booth, etc. On some laptops try index `1`. Grant camera permission, then restart the terminal.

`Unable to open file at hand_landmarker.task`**.** Run from the project root and download the model (see Setup).

**Hand not detected.** More light, slower motion, full palm facing the camera. If another app has an exclusive camera lock, OpenCV may still open a black feed.

**Drawing jumps or scribbles.** Lift the index between strokes. Jitter is normal at low resolution; stand closer or improve lighting rather than lowering all confidence values at once.

**macOS: “camera in use” with a black frame.** The process has a device handle but not frames. Kill leftover `python` / `air_canvas` processes and retry.

## Limitations

- One hand, no pinch/pressure, no undo beyond eraser and `C`.
- Finger “up” is a 2D Y comparison — a rotated hand or sideways camera view will misclassify gestures.
- Canvas is in-memory only; nothing is saved to disk.
- Overlay treats near-black as empty, so very dark ink would not show well (current palette avoids that except the eraser).



## Ideas to extend

- Save PNG of `canvas` on `S`
- Brush size from thumb–index distance
- Two-hand mode (`num_hands=2`)
- Smoothing (EMA / One Euro) on landmark 8
- Shape tools: two-finger drag for rectangle/circle while in SELECT



