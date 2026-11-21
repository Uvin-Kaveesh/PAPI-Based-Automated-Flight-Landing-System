# Technical notes and verification plan

These notes interpret the [July 2024 group report](technical-report-group-02.pdf) as an engineering prototype. “Documented” means the report states or shows it; it does not mean the behavior has been independently measured.

## 1. Feedback-loop model

The simulator maps aircraft position relative to five approach bands into one of five PAPI states. The camera and classifier estimate that state from the displayed light pattern. The controller converts the state into keyboard input, causing the simulated aircraft to change altitude. The new game state then generates the next image.

```text
approach geometry → rendered PAPI pattern → camera frame → classified state
                  ↑                                        ↓
          game aircraft position ← keyboard command ← Nano / Pro Micro
```

Let `s` be the discrete observed state in `{too_high, slightly_high, on_path, slightly_low, too_low}`. A conceptual control law is `u = f(s)`, with `u` a keyboard action such as climb, descend, or neutral. The report describes the direction of correction and explicitly identifies `f(too_low) = w`; it does **not** specify every mapping or the rate at which actions are issued. This system is therefore best described as a **discrete visual feedback controller**, not a demonstrated continuous autopilot.

PAPI gives angular path information. Even perfect light recognition would not reveal airspeed, flare height, runway position, sink rate, wind, or aircraft system state. A real autoland function needs these and independent safety assurance.

## 2. Signal and interface inventory

| Boundary | Reported signal | Missing detail needed to reproduce it |
| --- | --- | --- |
| Game → webcam | Four red/white virtual lights | Screen scale, camera exposure, frame rate, occlusion tolerance |
| Webcam → Python | Video frames | Resolution, crop, color handling, preprocessing |
| Model → Nano | One of five state values | Serial protocol, baud rate, packet format, timeout |
| Nano → Pro Micro | Five separate state lines | Complete pin map, voltage levels, mutual exclusion, transition timing |
| Nano → LEDs | Eight outputs for red/white pairs | Pin map, resistor values, drive arrangement |
| Pro Micro → game | Keyboard events | Full state-to-key map, hold duration, neutral behavior |

Page 9 gives one concrete wiring/action example: the Nano raises **analog pin 4** for *too low*, the Pro Micro detects it, and the game receives `w`. Do not infer the other four pin assignments from that example.

## 3. Recognition evidence

The report gives a nominal dataset size of 750 images, evenly divided among the five states. It describes model training using “linear regression,” Google Colab, and TensorFlow Lite. Those descriptions are insufficient to reconstruct a classifier. In particular, a five-state decision needs an explicit class encoding and inference rule, which the report does not provide.

For a meaningful evaluation, reserve a test set captured in separate sessions or with changed camera placement and lighting. Record a five-by-five confusion matrix, per-class precision/recall, overall accuracy, and the rate of unsafe opposite-direction decisions. Measure end-to-end delay from a game light change to the resulting key event. Keep training and test frames from the same continuous recording out of different splits, because near-duplicate frames inflate apparent accuracy.

## 4. Closed-loop hazards and mitigations to test

| Failure mode | Effect in the simulator | Verification target |
| --- | --- | --- |
| White/red misclassification | Wrong altitude correction | Confusion matrix and opposite-direction error count |
| Flicker near a band boundary | Rapid alternating key commands | Transition trace; consider hysteresis or persistence |
| Camera blocked or stale frame | Old correction continues | Timeout should release all keys and enter neutral state |
| Two state lines high together | Ambiguous Pro Micro input | Define priority or reject invalid states |
| Serial data loss | LEDs and game control disagree | Sequence number or explicit state acknowledgement |
| Continuous key hold | Overshoot or oscillation | Measure command duration and aircraft response |
| Changed monitor brightness/exposure | Reduced recognition reliability | Test across documented display and camera settings |

These are **proposed tests and design considerations**, not features claimed to be in the original implementation.

## 5. Reproduction checklist

1. Obtain the original `.sb3` Scratch project, Python script, trained model, training notebook, Arduino Nano sketch, and Pro Micro sketch from the team.
2. Record exact board variants, voltage levels, pin mapping, LED current-limit resistors, and serial protocol.
3. Run a bench test with no game motion: switch through all five states and verify classifier output, one-hot state lines, keyboard events, and four LED colors.
4. Record timestamped end-to-end traces for each state transition, including invalid or missing camera input.
5. Run repeated simulated approaches from defined initial conditions; report success criteria, number of trials, failures, and variability.
6. Keep claims about real aircraft applications separate from this simulator until a formal architecture and verification program exists.

## 6. Source and limitations

Primary source: *Landing Rectifying System for Flights (LRSF)*, IE1004 Computational Thinking Project, Group 02, July 2024. The README images are extracted from its figures. Page 11 itself describes this as a simulated version and identifies improved detection as future work. Source code, raw images, model files, and test logs were not attached to the report provided for this repository.

