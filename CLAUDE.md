# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MATLAB-based EEG signal processing pipeline for neurofeedback (NF) systems, using the **DASPS dataset** — EEG recordings of 23 participants under anxiety-inducing stimulation. The goal is anxiety detection from EEG to support a neurofeedback system for students.

## Data Architecture

### Raw struct format (per subject)
Each `Raw data mat/S##.mat` loads as a struct with:
- `.data` → `14 × 1920 × 12` (electrodes × samples × recordings)
- `.labels` → `12 × 2` (valence, arousal — SAM self-report per recording)

Dimensions:
- **14** = EMOTIV EPOC+ channels: `AF3 F7 F3 FC5 T7 P7 O1 O2 P8 T8 FC6 F4 F8 AF4`
- **1920** = 15 seconds × 128 Hz
- **12** = 6 situations × 2 phases each

### Recording index convention
- **Odd indices** (1,3,5,7,9,11): psychologist recites the anxiety scenario (happens first)
- **Even indices** (2,4,6,8,10,12): participant recites/imagines the scenario (happens second)
- ⚠️ **Neither phase is a true rest baseline** — both phases induce anxiety. Use SAM arousal scores or Hamilton labels to separate low vs high anxiety states.

### Labels
- `Documents and Code/participant_rating_dasps.xlsx`: one row per subject per situation (6 rows/subject), with valence, arousal, Hamilton1 (pre), Hamilton2 (post)
- `.labels` in each `.mat` duplicates the situation rating for both phases → 12 rows total

### Preprocessed data
- `Preprocessed data mat/S##preprocessed.mat`: cleaned/filtered versions of raw data, same struct format
- `preprocessed_DASPS_dataset/preprocessed_DASPS_dataset.mat`: all subjects consolidated

## Processing Scripts (`Documents and Code/Code/`)

| Script | Input | Output | Purpose |
|---|---|---|---|
| `TransformData.m` | `data` (14×1920×12) | `datas1` cell array (12×1, each 14×1920) | Splits 3D array into per-recording 2D matrices |
| `Segmentation1s.m` | `Regim_datasub.trial` cell array | `DASPS_datasub.trial` cell array | Segments each recording into 1-second (128-sample) windows |
| `TransformLabels.m` | `Regim_datasub.label` | `DASPS_datasub.Labels` cell array | Expands labels to match each 1-second segment (15 labels per recording) |

### Typical pipeline order
```
Raw S##.mat → TransformData → Segmentation1s → TransformLabels → ML classification
```

## EEGLAB Integration

EEGLAB cannot directly open DASPS `.mat` files (wrong struct format). Convert first:

```matlab
fs = 128;
paciente_1 = load('Raw data mat/S01.mat');
datos_2d = reshape(paciente_1.data, 14, []);  % 14 × 23040

ch_names = {'AF3','F7','F3','FC5','T7','P7','O1','O2','P8','T8','FC6','F4','F8','AF4'};
EEG = pop_importdata('dataformat','array','nbchan',14,'data',datos_2d,'srate',fs,'pnts',size(datos_2d,2),'xmin',0);
for i = 1:14, EEG.chanlocs(i).labels = ch_names{i}; end
EEG = eeg_checkset(EEG);

% Save as EEGLAB-native format for future GUI use:
pop_saveset(EEG, 'filename', 'S01_eeglab.set', 'filepath', './');
```

## Key Parameters

| Parameter | Value |
|---|---|
| Sampling rate | 128 Hz |
| Channels | 14 |
| Segment length (raw) | 1920 samples = 15 s |
| Segment length (processed) | 128 samples = 1 s |
| Subjects | 23 (S01–S23) |
| Situations | 6 per subject |
| Total duration per subject | 180 s = 3 min |

## Neuromarker: Frontal Alpha Asymmetry (FAA)

The chosen neuromarker for the NF system is **Frontal Alpha Asymmetry (FAA)**:

- **Formula:** `FAA = ln(alpha_F4) - ln(alpha_F3)`
- **Alpha band:** 8–13 Hz
- **Channels used:** F3 (left frontal) and F4 (right frontal) from EMOTIV EPOC+
- **Interpretation:** More negative FAA (greater left alpha, less left activation) correlates with higher anxiety/negative affect. More positive FAA correlates with approach motivation and lower anxiety.
- **Why:** FAA is well-validated in anxiety and emotion research. It reflects hemispheric asymmetry in prefrontal activity — right-dominant activation during anxiety reduces right-side alpha relative to left.
- **Estimation method:** Compute alpha band power per channel using `pwelch` in sliding windows (2 s window, 0.5 s step), then apply the log-difference formula across time.

## Working File

`matlab_reto.mlx` is the main MATLAB Live Script for the project. It loads S01 as the primary subject and is where active development happens.
