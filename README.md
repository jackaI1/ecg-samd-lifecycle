# ECG rhythm screening

I built a heart-rhythm screening tool because I wanted a way to check whether an ECG algorithm still did what I intended after I changed its code. It finds heartbeats in public recordings, measures the time between them, and marks 30-second sections with uneven beat spacing for inspection. It also checks each result against the recording's reference annotations, which are labels supplied with the data.

This is a learning exercise, not a cleared device. Do not use it for diagnosis, triage, treatment or reassurance about a person's health. It collects no new patient data and uses no private or identifiable patient records. The inputs are public, de-identified historical ECG recordings from real people, not synthetic signals.

## Try it

Install [uv](https://docs.astral.sh/uv/getting-started/installation/), a Python environment manager, then run:

```sh
git clone https://github.com/jackaI1/ecg-samd-lifecycle.git
cd ecg-samd-lifecycle
uv sync --frozen --python 3.12
uv run ecg-lifecycle evaluate
```

The first run downloads about 6 MB from PhysioNet. It reads three complete recordings and writes `build/benchmark.json`. Each 30-second window includes its beat-spacing measurements and a `regular`, `irregular` or `insufficient_data` result. A regular result only means the timing thresholds were not crossed; it says nothing about whether a heart is healthy.

To inspect one recording, run `uv run ecg-lifecycle evaluate --records 201 --output build/201.json`. The command accepts only the three public demonstration records. It has no patient upload feature.

## What I check

A good-looking result is not enough to tell me whether a change is safe to keep. I wrote down the expected behaviour, the ways the output could mislead someone, and the tests for each control. A script builds the table connecting those requirements and controls to tests. The check fails if any link is missing, the named test does not exist, or the test did not pass.

I used an IEC 62304 style software lifecycle and ISO 14971 style risk file to organize that work. These standards describe software development and medical-device risk management. This small project does not claim compliance, certification or clinical validation. My [design-control record](docs/lifecycle-process.md) explains the scope and what a real release would still require.

The lifecycle files are:

- [Requirements](docs/requirements.md): what the tool must do and how I accept a change.
- [Architecture and SOUP list](docs/architecture-and-soup.md): how the parts fit together and which outside software I rely on. SOUP means software of unknown provenance, used here as a dependency-review concept.
- [Risk file](docs/risk-file.md): hazards, controls, remaining problems and the decision to prohibit clinical use.
- [Generated traceability matrix](docs/traceability.md): the requirements and controls connected to executed tests.
- [Verification report](docs/verification-report.md): the saved local baseline. Every CI run produces a fresh report and downloadable evidence.

## How the detector works

WFDB reads the [MIT-BIH Arrhythmia Database, version 1.0.0](https://physionet.org/content/mitdb/1.0.0/). Its XQRS detector finds the main electrical peaks of heartbeats on the first ECG channel. The detector never receives the reference annotations.

An RR interval is the time from one detected heartbeat to the next. For each complete 30-second window, I calculate the average interval, its coefficient of variation (spread divided by average), and RMSSD (how much neighbouring intervals differ). I flag the window when the coefficient of variation is at least 0.12 and RMSSD divided by the average interval is at least 0.08. Too few intervals or an interval outside 0.25 to 2.5 seconds produces `insufficient_data`.

These simple thresholds do not identify a particular disease. Extra beats, missed beats and noise can all look irregular. I separately compare the flags with AFIB annotations, which mark atrial fibrillation, to make those limitations visible. Mixed or unknown windows are excluded, and abstentions are counted separately. [The evaluation protocol](docs/evaluation.md) explains every denominator.

## Reproduce the evidence

```sh
# Unit tests, real-data integration tests, traceability gate and full benchmark
uv run --frozen python -m scripts.verify

# Fast offline tests, after installing the environment
uv run pytest -m 'not integration'

# Refresh the saved local report and generated matrix after reviewing a change
uv run --frozen python -m scripts.verify --update-docs
```

`build/` contains the report, test results, matrix, environment details, input hashes and every window's features. Raw recordings stay in the ignored `.cache/mitdb/` directory. Set `ECG_DATA_DIR` to reuse a verified cache elsewhere. Every input file is checked against a committed SHA-256 digest before it is read. A corrupt cache stops the command; remove the named file to download it again.

## Baseline results and CI

Heartbeat matching allows a 150 ms difference from an annotated beat, with each beat used at most once. Sensitivity means the share of reference beats found. Positive predictive value (PPV) means the share of detected beats that matched a reference beat.

| Full record | Reference beats | Missed beats | Extra detections | Sensitivity | PPV |
| --- | --- | --- | --- | --- | --- |
| 100 | 2,273 | 0 | 0 | 100.00% | 100.00% |
| 201 | 1,963 | 76 | 0 | 96.13% | 100.00% |
| 203 | 2,980 | 65 | 18 | 97.82% | 99.39% |

The irregular-window comparison is much less reliable as a disease proxy: record 201 has 35 false-positive AFIB windows and only 32.69% PPV. Record 203 has four AFIB-positive abstentions; counting those as misses gives 91.67% sensitivity. Record 100 has no AFIB-positive windows, so its AFIB sensitivity is undefined. I have not validated the tool across the full database or a new population.

[GitHub Actions](https://github.com/jackaI1/ecg-samd-lifecycle/actions/workflows/verify.yml) runs the full suite on pull requests and on `main`, generates the verification report, checks the matrix is current, and uploads an evidence artifact. The baseline has 25 passing tests, 11 requirements and 10 risk controls. Passing CI permits an educational code release only.

Data citation: Moody, G., and Mark, R. (2005), MIT-BIH Arrhythmia Database v1.0.0, [doi:10.13026/C2F305](https://doi.org/10.13026/C2F305). Original database description: Moody and Mark (2001), *The impact of the MIT-BIH Arrhythmia Database*. Dataset use is under the [Open Data Commons Attribution License v1.0](https://physionet.org/content/mitdb/view-license/1.0.0/); the code has a separate [MIT license](LICENSE).
