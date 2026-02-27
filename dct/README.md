# DCT Image Steganography
### Hide secret text inside images — invisible to the human eye

---

## Table of Contents
1. [What is this?](#what-is-this)
2. [Why Python?](#why-python)
3. [How to run on Debian](#how-to-run-on-debian)
4. [Image requirements](#image-requirements)
5. [Embedding steps — how data gets hidden](#embedding-steps)
6. [Extraction steps — how data gets recovered](#extraction-steps)
7. [Project files](#project-files)
8. [Parameters you can tune](#parameters-you-can-tune)
9. [Limitations](#limitations)

---

## What is this?

This project hides a secret text message inside a PNG image using the **Discrete Cosine Transform (DCT)**. The output image (called the *stego image*) looks completely identical to the original. No one can tell there is a hidden message just by looking at it.

---

## Why Python?

Python is the best choice for this because:

- **NumPy and SciPy** give you fast, reliable DCT math out of the box
- **OpenCV** handles image loading, color space conversion, and saving
- **JupyterLab** gives you a Kaggle-style block-by-block execution environment on your own machine
- Clean readable syntax makes the algorithm easy to understand and modify
- Massive community and documentation for image processing topics

---

## How to run on Debian

### Step 1 — Install Python
```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv -y
```

### Step 2 — Create a virtual environment
```bash
cd ~
python3 -m venv stego_env
source stego_env/bin/activate
```
You will see `(stego_env)` in your terminal. Run the `source` command every time you open a new terminal.

### Step 3 — Install dependencies
```bash
pip install jupyterlab numpy opencv-python scipy Pillow matplotlib
```

### Step 4 — Launch JupyterLab
```bash
cd ~/dct
jupyter lab
```
A browser window opens. Open `dct_steganography.ipynb` and run each block with `Shift + Enter`.

### Step 5 — Set your inputs in Block 7
```python
COVER_IMAGE_PATH = 'your_image.png'
SECRET_MESSAGE   = 'your secret text here'
STEGO_IMAGE_PATH = 'stego_output.png'
```

Run Block 7 → Block 8 → Block 9 in order.

---

## Image requirements

| Requirement | Detail |
|---|---|
| Format | PNG only — never JPEG (compression destroys hidden data) |
| Size | Recommended 1000x1000 or larger |
| Note | Width and height do not need to be multiples of 8 — the code crops automatically |

To verify your image is a true PNG:
```bash
file your_image.png
# must say: PNG image data
```

If it says JPEG, convert it:
```bash
python3 -c "import cv2; img=cv2.imread('your_image.jpg'); cv2.imwrite('your_image.png', img)"
```

---

## Embedding Steps

This is the exact sequence the code follows to hide your message inside the image.

```
+-----------------------------------------------------+
|                   COVER IMAGE                       |
|              (your original PNG file)               |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 1 — Crop to multiple of 8                     |
|                                                     |
|  Width and height are cropped to the nearest        |
|  multiple of 8 pixels.                              |
|  Example: 1313x736 becomes 1312x736                 |
|                                                     |
|  WHY: The DCT works on 8x8 blocks. Leftover pixels  |
|  cause block misalignment between embed and         |
|  extract, corrupting the recovered message.         |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 2 — Convert color space: BGR -> YCrCb         |
|                                                     |
|  The image is split into 3 channels:                |
|  Y  = luminance (brightness)      <- we use this    |
|  Cr = red chrominance (color)     <- untouched      |
|  Cb = blue chrominance (color)    <- untouched      |
|                                                     |
|  WHY: Human eyes are far more sensitive to color    |
|  changes than brightness changes. Modifying only    |
|  Y keeps the distortion completely invisible.       |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 3 — Convert message to bits                   |
|                                                     |
|  Each character -> ASCII code -> 8 bits             |
|  A null character '\x00' is added at the end        |
|  as an end-of-message marker.                       |
|                                                     |
|  Example:                                           |
|  'Hi' -> H=01001000, i=01101001 -> 16 bits total   |
|  + '\x00' -> 00000000 appended at end               |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 4 — Split Y channel into 8x8 blocks           |
|                                                     |
|  The Y channel is divided into a grid of 8x8        |
|  pixel blocks, processed left-to-right,             |
|  top-to-bottom. One bit is embedded per block.      |
|                                                     |
|  A 1312x736 image gives:                            |
|  (1312/8) x (736/8) = 164 x 92 = 15,088 blocks     |
|  = max ~1,886 characters                            |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 5 — Apply 2D DCT to each block                |
|                                                     |
|  Each 8x8 pixel block is transformed from           |
|  spatial domain -> frequency domain.                |
|                                                     |
|  Result: 64 DCT coefficients per block              |
|  Top-left     = low frequencies  (image structure)  |
|  Bottom-right = high frequencies (fine detail)      |
|  Middle area  = mid frequencies  <- we embed here   |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 6 — Embed one bit using coefficient pair      |
|                                                     |
|  We use TWO mid-frequency coefficients:             |
|  A = coefficient at position (row=4, col=5)         |
|  B = coefficient at position (row=5, col=4)         |
|                                                     |
|  Calculate midpoint: mid = (A + B) / 2             |
|                                                     |
|  To embed bit = 1:                                  |
|    A = mid + DELTA/2   ->   force A > B             |
|                                                     |
|  To embed bit = 0:                                  |
|    A = mid - DELTA/2   ->   force A < B             |
|                                                     |
|  DELTA = 25 (the enforced gap between A and B)      |
|                                                     |
|  WHY two coefficients instead of one?               |
|  Comparing A vs B is a relative check that          |
|  survives uint8 rounding. Checking an exact value   |
|  does not — rounding corrupts it.                   |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 7 — Apply Inverse DCT (IDCT)                  |
|                                                     |
|  The modified DCT block is converted back           |
|  from frequency domain -> pixel values.             |
|  The block is placed back into the Y channel.       |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 8 — Reconstruct and save stego image          |
|                                                     |
|  Clip Y values to [0, 255] and cast to uint8        |
|  Merge modified Y with original Cr and Cb           |
|  Convert YCrCb -> BGR                               |
|  Save as PNG (lossless — no data destroyed)         |
+-----------------------------------------------------+
                         |
                         v
                  STEGO IMAGE (done)
```

---

## Extraction Steps

This is the exact sequence the code follows to recover the hidden message.

```
+-----------------------------------------------------+
|                   STEGO IMAGE                       |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 1 — Load image and convert BGR -> YCrCb       |
|                                                     |
|  Same color space conversion as embedding.          |
|  Extract only the Y (luminance) channel.            |
|  The stego image is already cropped to a multiple   |
|  of 8 from the embedding step — no cropping needed. |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 2 — Split into 8x8 blocks                     |
|                                                     |
|  Same grid traversal as embedding:                  |
|  left-to-right, top-to-bottom.                      |
|  CRITICAL: must match embedding order exactly.      |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 3 — Apply 2D DCT to each block                |
|                                                     |
|  Same DCT transform as embedding.                   |
|  Recover the 64 frequency coefficients.             |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 4 — Read the bit from coefficient pair        |
|                                                     |
|  Read A at (4,5) and B at (5,4)                     |
|                                                     |
|  If A > B  ->  bit = 1                              |
|  If A < B  ->  bit = 0                              |
|                                                     |
|  Simple relative comparison —                       |
|  no exact values needed, very robust.               |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 5 — Collect bits and check for end marker     |
|                                                     |
|  After every 8 bits collected, check if the         |
|  last byte equals 0 (null character '\x00').        |
|  If yes -> end of message found, stop reading.      |
|  If no  -> continue to next block.                  |
+------------------------+----------------------------+
                         |
                         v
+-----------------------------------------------------+
|  STEP 6 — Convert bits back to text                 |
|                                                     |
|  Group bits into bytes (8 bits each)                |
|  Convert each byte to its ASCII character           |
|  Join all characters -> recovered message           |
+-----------------------------------------------------+
                         |
                         v
                SECRET MESSAGE (recovered)
```

---

## Project files

```
dct/
├── dct_steganography.ipynb   <- main notebook (9 blocks)
├── README.md                 <- this file
├── your_cover_image.png      <- your input image (you provide this)
└── stego_image.png           <- output: generated by Block 7
```

---

## Parameters you can tune

Defined in **Block 3** of the notebook:

| Parameter | Default | Effect |
|---|---|---|
| `COEFF_A` | (4, 5) | Position of first DCT coefficient used for embedding |
| `COEFF_B` | (5, 4) | Position of second DCT coefficient used for embedding |
| `DELTA` | 25 | Gap enforced between the two coefficients. Higher = more robust but slightly more distortion |

**Note for future developers:** `COEFF_A`, `COEFF_B`, and `DELTA` must be **identical** in both the embedder and extractor. If they differ, extraction will produce garbage output.

---

## Limitations

| Limitation | Detail |
|---|---|
| PNG only | JPEG re-compression destroys the embedded coefficients. Always save stego as `.png` |
| No encryption | The message is hidden but not encrypted. For real security, encrypt with AES before embedding |
| Capacity | 1 bit per 8x8 block. A 1312x736 image holds ~1,886 characters max |
| Fragile to editing | Any resize, rotate, or color adjustment will destroy the hidden data |
| Width cropped | Images with width not divisible by 8 lose at most 7 pixels on the right edge |

---

*Built with Python, NumPy, SciPy, and OpenCV.*
