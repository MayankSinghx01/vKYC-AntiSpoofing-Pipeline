# vKYC AntiSpoofing Pipeline

<img width="1536" height="1024" alt="vkyc_full_pipeline" src="https://github.com/user-attachments/assets/39f8dd80-3ab3-4b4d-b52a-ff3b3052a6c9" />

## **Machine Learning Internship — Computer Vision, Face Anti-Spoofing & Image Forensics**

During my ML internship at Fibe, I worked on improving a selfie-based identity verification system used in their video-KYC (Know Your Customer) pipeline. The goal was to make the system more robust against fake or manipulated selfies — including photos of photos, screen replays, and AI-generated or deepfake images.

> This repository is a technical write-up of the internship. Proprietary code, internal datasets, model weights, credentials, and company infrastructure are excluded due to an NDA.

---

## Problem

A selfie verification system can't just check "is there a face in the image?" It also needs to catch cases like:

- A photo or screen being held up to the camera instead of a live face
- A manipulated or deepfake image
- Poor facial visibility, multiple people, or unwanted objects in frame
- An image suspiciously similar to a previously submitted one
- An AI-generated or heavily edited image

The existing system already handled a lot of the basics. My work focused on adding new detection layers for the harder cases above.

---

## Existing System (before my work)

- **OpenCV** — image quality checks
- **MediaPipe Face Landmarker** — facial landmarks, eye state, head pose
- **YOLOv8n** — object detection
- **MobileCLIP2-S4** — zero-shot visual rule checks

## What I Added

1. A real-time deepfake attack demonstration to test the system's blind spots
2. Face anti-spoofing using MiniFASNet
3. Extended facial analysis using MediaPipe
4. Identity similarity checks using ArcFace
5. Image authenticity checks using C2PA, FFT analysis, and sensor noise
6. AI/deepfake image classification using a fine-tuned CLIP ViT-L/14

---

## 1. Red-Teaming the Pipeline: A Real-Time Deepfake Attack

Before adding new defenses, I first tested whether the existing checks could be bypassed. I built a real-time deepfake attack and ran it against the pipeline.

**Result:** face detection, image quality checks, object detection, and facial geometry checks all passed — even though the face was fake. This confirmed the system needed dedicated anti-spoofing and image-authenticity checks, not just "is there a plausible face here."

---

## 2. Face Anti-Spoofing — MiniFASNet

I integrated **MiniFASNet**, a lightweight model built specifically to detect presentation attacks (e.g., a photo or video replayed to the camera), separate from general "is this AI-generated" detection.

Run using **ONNX Runtime** for fast inference. The face crop includes a margin around the detected region before being resized and normalized to the model's input size.

---

## 3. Facial Analysis — MediaPipe

Used MediaPipe's Face Landmarker to extract detailed facial geometry:

- Face count, bounding box, size, and framing
- Eye state, Eye Aspect Ratio (EAR), and blink scores
- Head pose (yaw, pitch, roll)
- Landmark visibility and facial obstruction

EAR (Eye Aspect Ratio) is used for determining closed or opened eyes.
A lower EAR indicates a more closed eye. EAR was combined with MediaPipe's blink-related blendshape scores for more reliable eye-state detection.

**Head pose** (yaw/pitch/roll) was used to check whether the face was properly oriented toward the camera — a basic but important gate before running heavier checks downstream.

---

## 4. Identity Similarity — ArcFace

Used **ArcFace** to generate a 512-dimensional embedding for each detected face, then compared it against recently stored embeddings using cosine similarity:

$$ \text{sim}(A, B) = \frac{A \cdot B}{\|A\| \|B\|} $$

---

## 5. Image Authenticity & Forensic Signals

Three complementary signals, none of them conclusive on their own:

**C2PA (Content Provenance)**
Checks whether the image carries verifiable provenance metadata, run on the raw image before any other processing. This is a different kind of evidence from pixel-level analysis — it's about the image's history, not its appearance.

**FFT Analysis**
Used a 2D Fast Fourier Transform to examine an image's frequency spectrum and compared it against a baseline built from real camera images. Image generation and processing pipelines tend to leave detectable traces in frequency space.

**Sensor Noise**
Estimated noise levels and compared them against a reference distribution built from real camera images. Unusually low noise is a warning sign, but compression and processing can also cause this, so it's treated as supporting evidence rather than proof.

---

## 6. AI / Deepfake Image Detection — Fine-Tuned CLIP ViT-L/14

Fine-tuned OpenAI's `clip-vit-large-patch14` (~428M parameters) for binary classification of AI-generated and deepfake images.

**Dataset & Methodology:**
The model was trained and validated on a custom internal dataset of **40,000 images** (20,000 genuine selfies, 10,000 AI-generated images, and 10,000 deepfakes).

**Results:**

| Task | Validation Accuracy |
|---|---|
| AI-generated image detection | ~70% |
| Deepfake detection | ~80% |

These numbers reflect performance on the validation set used during the internship — a useful signal, but also a reminder that detectors trained this way can struggle to generalize to generation methods, datasets, or compression levels they weren't trained on.

---

## 7. Evaluated Baselines & Literature Review

Before finalizing the multi-model architecture, several baselines from established repositories (such as DeepFakeBench) and academic literature were evaluated. These were ultimately excluded from the final production pipeline due to real-world deployment constraints:

*   **[Insert Model/Paper 1]**: [Briefly explain why it was rejected—e.g., Inference latency was too high for a real-time vKYC pipeline].
*   **[Insert Model/Paper 2]**: [Briefly explain the bottleneck—e.g., Struggled heavily with specific mobile compression artifacts or lacked generalizability across diverse lighting].

---

## How the Pieces Fit Together

| Component | What it checks |
|---|---|
| OpenCV | Image quality |
| MediaPipe | Facial geometry |
| YOLOv8n | Objects and their locations |
| MobileCLIP2 | Visual/semantic conditions |
| MiniFASNet | Presentation attacks |
| ArcFace | Face-embedding similarity |
| C2PA | Content provenance |
| FFT | Frequency-domain characteristics |
| Sensor noise | Camera/image noise |
| CLIP ViT-L/14 | Learned visual patterns for AI/deepfake classification |

No single component is meant to solve the whole problem. The verification system combines these signals rather than relying on any one of them, because each has different strengths and different ways of failing.

---

## What I Learned

This internship was my first real exposure to building a security-oriented CV system rather than a purely accuracy-oriented one — the mindset shift from "how do I classify this correctly" to "how could someone try to fool this, and what happens when they do" was the most useful part.

Concretely, I worked with:

- Computer vision and face analysis pipelines
- Presentation-attack detection (anti-spoofing)
- Face embeddings and similarity search
- Object detection and vision-language models
- Image provenance and frequency-domain forensics
- Fine-tuning Vision Transformers
- Threshold selection and validation trade-offs (false rejections vs. false acceptances)

The main takeaway: real-world image verification isn't about picking one "best" model — it's about combining methods with different failure modes and tuning the system as a whole for accuracy, robustness, and cost.

---

## Technologies

· Python · PyTorch · OpenCV · MediaPipe · YOLOv8 · MobileCLIP2 · MiniFASNet · ArcFace · CLIP · Vision Transformers · ONNX Runtime · C2PA
