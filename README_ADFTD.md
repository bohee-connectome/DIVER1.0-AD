# ADFTD Dataset Preprocessing Guide

## Overview

This guide explains how to preprocess the ADFTD (Alzheimer's Disease and Frontotemporal Dementia) dataset from OpenNeuro for use with the DIVER CBraMod preprocessing pipeline.

**Dataset Information:**
- **Source**: OpenNeuro dataset ds004504 (https://openneuro.org/datasets/ds004504/versions/1.0.8)
- **Description**: EEG recordings from 88 subjects (29 healthy controls, 36 AD patients, 23 FTD patients)
- **Recording Details**:
  - Sampling rate: 500 Hz
  - Electrode system: GSN-HydroCel-129 (129 channels)
  - Recording duration: ~12-14 minutes per subject
  - Task: Resting state, eyes closed
- **Data Format**: EEGLAB .set files in BIDS format

## Files Created for ADFTD Support

### 1. preprocessing_generalized_datasetsetting_ADFTD.py
This is a **copy** of the original `preprocessing_generalized_datasetsetting.py` with the following additions:

**Location of changes:**
- **Lines 447-604**: New `ADFTDDatasetSetting` class added before `HBNDatasetSetting`
- **Lines 1748-1765**: New `'ADFTD'` entry added to `_REGISTRY` dictionary

**Key features:**
- `file_list()`: Filters out derivatives folder, uses only raw data
- `filter()`: Applies bandpass filtering (default 0.5-45 Hz as per dataset specs)
- `shaping()`: Reshapes data into segments (similar to HBN processing)
- `set_meta_data()`: Extracts subject ID and task from BIDS-formatted paths
- `qc()`: Quality control with bad channel removal and clipping

### 2. preprocessing_generalized_ADFTD.py
This is a **copy** of the original `preprocessing_generalized.py` with the following modification:

**Location of change:**
- **Lines 22-26**: Changed import statement to use `preprocessing_generalized_datasetsetting_ADFTD` instead of the original

## Dataset Download

### Step 1: Install openneuro-py
```bash
pip install openneuro-py
```

### Step 2: Download the dataset
Option A: Using the provided download script:
```bash
cd D:\GitHub\DIVER\data
python download_adftd.py
```

Option B: Manual download using openneuro-py:
```bash
cd D:\GitHub\DIVER\data\ADFTD
python -c "from openneuro import download; download(dataset='ds004504', target_dir='.')"
```

Option C: Direct web download:
Visit https://openneuro.org/datasets/ds004504/versions/1.0.8/download

### Expected Directory Structure After Download
```
D:\GitHub\DIVER\data\ADFTD\ds004504\
├── dataset_description.json
├── participants.tsv
├── participants.json
├── README
├── CHANGES
├── sub-001\
│   └── eeg\
│       ├── sub-001_task-eyesclosed_eeg.set
│       ├── sub-001_task-eyesclosed_eeg.fdt
│       ├── sub-001_task-eyesclosed_eeg.json
│       └── sub-001_task-eyesclosed_channels.tsv
├── sub-002\
│   └── eeg\
│       └── ...
...
└── derivatives\  # (will be skipped during preprocessing)
```

## Running Preprocessing

### Basic Command
```bash
cd D:\GitHub\DIVER\CBraMod\preprocessing

python preprocessing_generalized_ADFTD.py \
    --dataset_name ADFTD \
    --data_path D:\GitHub\DIVER\data\ADFTD\ds004504 \
    --save_path_parent D:\GitHub\DIVER\data\processed \
    --coordinate_file_path D:\GitHub\DIVER\CBraMod\preprocessing\GSN-HydroCel-129.sfp \
    --resample_rate 500 \
    --highpass 0.5 \
    --lowpass 45 \
    --notch_filter 60 \
    --segment_len 30 \
    --percent 1.0 \
    --num_chunks 10
```

### Command-Line Arguments

| Argument | Description | Default | Recommendation for ADFTD |
|----------|-------------|---------|--------------------------|
| `--dataset_name` | Name of dataset | 'TUEG' | **'ADFTD'** |
| `--data_path` | Path to raw data | None | **D:\GitHub\DIVER\data\ADFTD\ds004504** |
| `--save_path_parent` | Output directory | None | **D:\GitHub\DIVER\data\processed** |
| `--coordinate_file_path` | Electrode coordinates (.sfp or .elc) | None | **path/to/GSN-HydroCel-129.sfp** |
| `--resample_rate` | Target sampling rate (Hz) | 500 | **500** (matches original) |
| `--highpass` | High-pass filter (Hz) | 0.3 | **0.5** (dataset standard) |
| `--lowpass` | Low-pass filter (Hz) | 200 | **45** (dataset standard) |
| `--notch_filter` | Notch filter frequency (Hz) | 60 | **60** (for 60Hz line noise) |
| `--segment_len` | Segment length (seconds) | 30 | **30** |
| `--percent` | Fraction of data to process | 0.1 | **1.0** (full dataset) |
| `--num_chunks` | Number of file chunks | 10 | **10** |
| `--parallel` | Enable parallel processing | False | **(add flag for faster processing)** |
| `--n_jobs` | Number of parallel jobs | None | **4-8** (depends on CPU) |

### Example: Parallel Processing
```bash
python preprocessing_generalized_ADFTD.py \
    --dataset_name ADFTD \
    --data_path D:\GitHub\DIVER\data\ADFTD\ds004504 \
    --save_path_parent D:\GitHub\DIVER\data\processed \
    --coordinate_file_path D:\GitHub\DIVER\CBraMod\preprocessing\GSN-HydroCel-129.sfp \
    --resample_rate 500 \
    --highpass 0.5 \
    --lowpass 45 \
    --notch_filter 60 \
    --segment_len 30 \
    --percent 1.0 \
    --num_chunks 10 \
    --parallel \
    --n_jobs 8
```

## Output

The preprocessing will create an LMDB database at:
```
D:\GitHub\DIVER\data\processed\1.0_ADFTD\all_resample-500_highpass-0.5_lowpass-45.lmdb
```

### Output Structure
Each sample in the LMDB contains:
- **sample**: EEG data array (channels × time_segments × sampling_rate)
- **label**: None (pretrain dataset)
- **data_info**:
  - Dataset: 'ADFTD'
  - modality: 'EEG'
  - subject_id: e.g., '001', '002', etc.
  - task: 'eyesclosed'
  - resampling_rate: 500
  - original_sampling_rate: 500
  - segment_index: 0, 1, 2, ...
  - start_time: segment start time in seconds
  - channel_names: list of channel names (E1-E128, Cz)
  - xyz_id: 3D coordinates of electrodes

## Quality Control

The preprocessing includes automatic quality control:
1. **Bad channel detection**: Channels with >3.33% of samples exceeding 100µV threshold are marked as bad
2. **Segment rejection**: Segments with >50% bad channels are discarded
3. **Amplitude clipping**: Values exceeding threshold are clipped to ±100µV

## Troubleshooting

### Issue: "No module named 'preprocessing_generalized_datasetsetting_ADFTD'"
**Solution**: Make sure you're in the preprocessing directory and the file exists:
```bash
cd D:\GitHub\DIVER\CBraMod\preprocessing
ls -la preprocessing_generalized_datasetsetting_ADFTD.py
```

### Issue: "GSN-HydroCel-129.sfp not found"
**Solution**: The coordinate file should be in the preprocessing directory or provide the correct path

### Issue: Download fails or is very slow
**Solution**:
- Try downloading smaller chunks
- Use the web interface for manual download
- Check your internet connection
- The full dataset is ~2.3 GB

### Issue: "ValueError: Unknown dataset ADFTD"
**Solution**: Make sure you're using `preprocessing_generalized_ADFTD.py` (not the original file)

## References

**Dataset Publication:**
- Miltiadous, A., Tzimourta, K.D., Afrantou, T. et al. "A Dataset of Scalp EEG Recordings of Alzheimer's Disease, Frontotemporal Dementia and Healthy Subjects from Routine EEG"
- DOI: 10.18112/openneuro.ds004504.v1.0.8

**DIVER Project:**
- Main repository: D:\GitHub\DIVER\CBraMod

---

**Last Updated**: 2025-10-17
**Author**: Claude Code
