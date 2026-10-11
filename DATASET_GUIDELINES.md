# Dataset Recording Guidelines — Scripted vs. Spontaneous Answers

Goal: 30 clips (5 subjects × 6 questions) that differ **only** in whether the words came from a screen or from the person's head. Anything else that differs between the two groups (topic, instructions, lighting, time of day, filming order) is a shortcut the model will learn instead of the real signal.

---

## 1. Study design

| | |
|---|---|
| Subjects | 5 (6 is better: it balances every question perfectly) |
| Questions | 6 (Q1–Q6) |
| Per subject | 6 clips: 3 scripted + 3 spontaneous, **each question answered once** |
| Total | 30 clips, 15 scripted / 15 spontaneous |
| Clip length | 30–45 s of speech (~75–110 words) |

### Question rotation (removes the "technical topic = scripted" shortcut)

Each question is scripted for some people and spontaneous for others. No one answers the same question both ways (they'd just repeat the script).

| Subject | Scripted | Spontaneous | Start with |
|---|---|---|---|
| S01 | Q1 Q2 Q3 | Q4 Q5 Q6 | scripted |
| S02 | Q4 Q5 Q6 | Q1 Q2 Q3 | spontaneous |
| S03 | Q1 Q2 Q3 | Q4 Q5 Q6 | spontaneous |
| S04 | Q4 Q5 Q6 | Q1 Q2 Q3 | scripted |
| S05 | Q1 Q2 Q3 | Q4 Q5 Q6 | scripted |
| (S06) | Q4 Q5 Q6 | Q1 Q2 Q3 | spontaneous |

**Alternate within the session** (e.g. S01: scripted, spontaneous, scripted, spontaneous, …) so tiredness and warm-up don't line up with one condition.

### The 6 questions (mix of technical and behavioral; all answerable both ways)

1. Tell me about a time you had to debug a difficult error in a project.
2. Explain the difference between a REST API and a GraphQL API.
3. Describe a challenging team project and your role in it.
4. How do you approach database normalization in a new project?
5. What is your favorite programming language or framework and why?
6. How do you use version control when working with a team?

---

## 2. Scripts (for scripted clips)

Generate each script with the **same** LLM prompt, so the scripts read like what a cheater would actually get:

> Answer this interview question in first person, as a spoken answer, in about 100 words: "<question>"

- Don't edit the output to make it sound more robotic or more natural. Use it as-is.
- **Cold read:** the subject must **not** see the script before recording. A real cheater reads LLM output for the first time, live. If they've seen it, use a different question.
- Keep all 6 scripts in `dataset/scripts/Q1.txt` … `Q6.txt` so the exact text is on record.

---

## 3. Instructions to subjects (identical for both conditions)

Read this out word for word before **every** clip:

> "This is a real job interview. Answer convincingly. The interviewer must not be able to tell if you're reading."

**Do not** say "sound robotic", "keep your eyes on the text", "use filler words", "look away to think", etc. Coaching people to show the behavior makes the model learn your instructions rather than real reading behavior. A real cheater tries to look natural; that's the case we need to catch.

---

## 4. Setup (same for every clip, every subject)

**Camera**
- Same laptop and webcam for all 30 clips if at all possible. If not, write the device down in the metadata.
- Native webcam resolution (720p or 1080p is fine), **30 fps**. Everything must have the same resolution and frame rate. Check with:
  ```bash
  venv/bin/python -c "import cv2,sys; c=cv2.VideoCapture(sys.argv[1]); print(c.get(5), c.get(3), c.get(4))" path/to/clip.mp4
  ```
  This prints `fps width height`. fps must be ~30 for every clip.

**Framing and light**
- Chest-up, 2–3 ft from the camera, face centred, **both eyes clearly visible**.
- Even light from the front (face a window or put a lamp behind the laptop). No shadows over the eyes, no strong backlight.
- The gaze tracker uses the **left eye**. Watch for glasses glare on that lens: tilt the glasses or the light until it's gone. Note glasses in the metadata.

**Screen**
- The text window is top-centre, directly under the webcam, about 40% of the screen width, 16 px font. Same size and position for everyone.
- **Leave the window open and blank for spontaneous clips too.** The screen should look identical in both conditions; only the presence of text differs.

**Audio**
- Built-in laptop mic, quiet room, door closed, no fans/music/other voices.
- **The interviewer's voice must not be on the recording.** Ask the question *before* pressing record (or show it on paper). Only the subject's answer is recorded.

---

## 5. Recording procedure (per clip)

1. Read the standard instruction (section 3) and the question.
2. Scripted: paste the script into the window **now**, just before recording.
3. Press record. The subject starts when ready. Leading and trailing silence is trimmed automatically, so a few seconds of quiet at either end is fine.
4. At ~45 s, give a "wrap up" hand signal. Stop recording when they finish.
5. Play back the first 5 s: is the face in frame and the voice audible?
6. Clear the text window.

**Retakes:** only for technical failures (face left frame, noise, interruption, the subject saw the script early). **Never retake because it "didn't look scripted/natural enough"**. That would bias the dataset toward the easy cases. Move the failed file to `dataset/rejected/` (it must not stay in `raw_videos/`) and save the new one as take `t2`.

---

## 6. File naming (strict — enforced by the code)

```
S<subject>_Q<question>_<scripted|spontaneous>_t<take>.mp4
```

Examples: `S01_Q1_scripted_t1.mp4`, `S04_Q5_spontaneous_t2.mov`

Any file that doesn't match this pattern is **skipped** with a warning (`.mp4`, `.mov` and `.avi` are accepted). The subject ID is used for grouping during testing, so **one person = one S-number, always**.

Folder layout (run from `src/`):

```
src/dataset/
├── raw_videos/     ← the 30 training clips, nothing else
├── rejected/       ← failed takes
├── hard_test/      ← optional hard cases (section 9), never in raw_videos
├── scripts/        ← Q1.txt … Q6.txt
└── metadata.csv
```

`dataset/` is git-ignored. **Never commit videos.**

### metadata.csv

```
filename,subject,question,condition,take,device,resolution,fps,glasses,lighting_notes,date
S01_Q1_scripted_t1.mp4,S01,Q1,scripted,1,MacBook M2,1920x1080,30,no,window front,2026-10-12
```

---

## 7. Consent

Faces and voices are biometric data. Before recording, get a short written consent from each subject saying: what's recorded, that it's used only for this project, stored locally, never published, and deleted on request (and by a stated date). Keep the signed forms outside the repo.

---

## 8. Train and test

```bash
cd src
../venv/bin/python extract_features.py     # reads dataset/raw_videos → data_features.csv
../venv/bin/python train_baseline.py       # leave-one-subject-out cross-validation
```

- `extract_features.py` overwrites `data_features.csv` (the mock data). Regenerate mock data any time with `generate_mock_data.py`.
- Because the CSV has a `subject` column, training automatically uses **leave-one-subject-out** testing: 5 folds, each trained on 4 people and tested on a 5th person it has never seen. That's the honest number, because it tells you whether the model works on a *new* candidate.

**Reading the results**
- With 30 clips, expect a lot of fold-to-fold variation (the `+/-`). Mean ROC-AUC ≥ 0.7 with no fold near 0.5 is a real signal. AUC ≈ 0.5 is a coin flip.
- Check `feature_importance.png`. If one feature dominates, open the CSV and check it isn't explained by something trivial (e.g. one subject's mic was louder).
- Before trusting the numbers, check the CSV for clips where every gaze value is `0.0`. That means the face was never detected; fix the lighting or framing and re-record.

**Using the results for the app**
The live app still uses the hand-set rules in `src/agents.py` (`gaze_std > 0.05`, `pause_ratio < 0.15 and pitch_std < 25`, `filler_ratio < 0.02`). Compare those cut-offs against the scripted and spontaneous columns in your `data_features.csv`, and move each threshold to where the two groups actually separate.

---

## 9. Optional: hard test set (later, ~6 clips)

Real cheaters don't just read cold. Film a few of these into `dataset/hard_test/` (same naming), and **never train on them**:

- reciting a memorised script
- glancing at bullet-point notes and expanding on them
- reading, then paraphrasing in their own words

Run them through the app one by one to see what slips past.

---

## Quick checklist per subject

- [ ] Consent signed
- [ ] S-number assigned, rotation row chosen (section 1)
- [ ] Light, framing, both eyes visible, no glare on the left lens
- [ ] Window blank for spontaneous, script pasted just before scripted
- [ ] Same instruction read before every clip
- [ ] Question asked *before* recording
- [ ] 6 clips named correctly in `raw_videos/`, metadata rows filled
