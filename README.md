# LEMON EEG Portfolio Pipeline — DMN/CEN/SN Connectivity and Activity

## 1. Overview

This pipeline is a methodological portfolio piece on the Default Mode
Network (DMN). It runs an end-to-end EEG analysis, including preprocessing,
source localization, network parcellation, connectivity and activity extraction,
and correlation with self-report measures, on a small subset of the publicly
available LEMON dataset (Babayan et al., 2019).

This is a pipeline demonstration, not a powered study. N=8, results are
exploratory and hypothesis-generating, not confirmatory.

## 2. Theoretical Background

This pipeline operationalizes two components of Schlossberg's 4S Transition
Model (1981, 1984) — Self and Strategies — as applied to the transition to
higher education during emerging adulthood (Arnett, 2014). The Default Mode
Network (DMN) is consistently linked to self-referential processing,
autobiographical memory, and identity construction (Andrews-Hanna, 2012;
Buckner et al., 2008), motivating its use here as a candidate physiological
correlate of the Self component. The Triple Network Model (Menon, 2011)
frames the Central Executive Network (CEN) and Salience Network (SN) as the
circuitry governing switching between internally- and externally-directed
cognition, motivating the Strategies proxy built here around CEN activity
and SN-mediated connectivity with DMN and CEN.

## 3. Sample

-   N = 8, drawn from LEMON and restricted to the 20–25 age band (LEMON
    reports age in 5-year bins, not exact age).

-   Of 79 subjects in the 20–25 band, 70 had complete data across all
    planned Self and Strategies questionnaires plus raw resting EEG.

-   8 subjects were randomly sampled from the eligible pool (seed=42),
    landing at a balanced 4F/4M split by chance.

Known sample-level issues:

-   sub-032324: truncated recording. Reduced from the expected 8
    eyes-closed blocks to 4 after correcting for a marker-repetition
    artifact (see §5 (Eyes-closed epoch extraction)), then to 3 after epoch-length validation
    dropped one block that didn't reach the full 59.9s. Kept in the
    sample, since a demo pipeline should be able to handle real-world
    truncated data.

-   sub-032469: excluded from activity-based analyses only (retained for
    connectivity). The noise covariance estimation for this subject
    triggered an eLORETA whitener-instability warning (largest/smallest
    eigenvalue ratio \>10 orders of magnitude). Connectivity (i.e.,
    correlation-based) appears robust to this instability, but activity
    (i.e., absolute measure) does not.

## 4. Preprocessing

-   VEOG explicitly typed as eog (not eeg) before montage assignment.

-   Montage: easycap-M1 (standard template, see §7 (Atlas and network definitions)).

-   Resampled 2500 Hz 250 Hz (as per LEMON's published preprocessing
    pipeline).

-   Band-pass filtered 1-100 Hz (instead of 1-45 Hz). ICLabel's
    automated component classifier was originally misclassifying a
    disproportionate number of components as muscle artifact (\~38–40%)
    because muscle artifact is partly identified from broadband
    high-frequency content that a 45 Hz low-pass had already removed.
    Widening to 100 Hz resolved this. Theta/alpha band-limiting for the
    actual analysis happens downstream (i.e., 1-45 Hz), at the epoching
    stage.

-   50 Hz notch filter (German recording site).

-   Bad channel detection via pyprep's NoisyChannels (implements the
    PREP pipeline's noisy-channel criteria), followed by spline
    interpolation. Run before referencing, so a bad channel's noise
    doesn't contaminate the average reference.

-   Average reference applied as an SSP projection. This is required
    downstream by apply_inverse_epochs, which explicitly rejects a
    custom-applied reference.

-   ICA: extended Infomax, n_components=0.99 (variance-based, avoids
    manual rank-deficiency corrections). Fit on a separate
    average-referenced copy of the data, since ICLabel's classifier was
    trained on average-referenced input.

-   Component exclusion: ICLabel labels of eye blink, muscle artifact,
    or heartbeat with predicted probability \> 0.8.

## 5. Eyes-closed epoch extraction

LEMON's resting protocol alternates 8× 60s eyes-open and 8× 60s
eyes-closed blocks, marked with S200/S210 respectively. The S210 marker
does not fire once per block onset, it repeats every 2 seconds
throughout the entire eyes-closed period. Block onsets were recovered by
taking only the first marker in each cluster of repeated markers, using
a 10-second gap threshold (compared in seconds, not raw sample counts)
to separate distinct blocks. Epochs: 0-59.9s per block, no baseline
correction.

## 6. Source localization

-   A template forward model (fsaverage, standard BEM, oct6 source
    spacing) was used throughout.

-   Noise covariance estimated from the data itself (method='shrunk').

-   Inverse operator: loose=0.2, depth=0.8. Method: eLORETA
    (Pascual-Marqui, 2007), lambda2=1/9 (SNR=3 assumption).

## 7. Atlas and network definitions

-   Parcellation: Desikan-Killiany (aparc, 68 whole-brain regions). DK
    ships natively with fsaverage and needs no additional
    volumetric-to-surface resampling.

-   DK regions were assigned to DMN / CEN (Control) / SN
    (Salience/Ventral Attention) using the correspondence table in
    Fukushima et al. (2018).

The source table operates at a finer sub-parcel resolution than whole DK
regions; some regions were split across networks in the original table.
These were resolved as follows:

-   Regions with a clear sub-parcel majority for one network were
    assigned to that network only.

-   Two regions genuinely split by hemisphere (not just sub-parcel
    majority) were kept hemisphere-specific: caudalmiddlefrontal (left
    to DMN, right to CEN) and parstriangularis (left to DMN, right to
    SN).

-   Two regions were exact ties between two networks at the sub-parcel
    level (superiorfrontal-rh: DMN/SN; inferiorparietal-rh: DMN/CEN).
    Both were retained in both networks. This means the three networks
    are not fully mutually exclusive: two of 40 total regions are double
    counted.

-   Final region counts: 22 DMN, 4 CEN, 14 SN (40 total, with 2 regions
    counted in two networks each, per above).

## 8. Connectivity and activity extraction

-   Connectivity: as a measure of inter and intra-regional coupling,
    orthogonalized amplitude envelope correlation (AEC) was computed
    separately for theta (4–8 Hz) and alpha (8–12/13 Hz) via
    mne_connectivity.envelope_correlation(\...,
    orthogonalize='pairwise'). Diagonal is forced to 0 by the
    orthogonalization procedure.

-   Activity: as a measure of regional signal magnitude, Hilbert
    envelope amplitude (\|hilbert(data, axis=-1)\|) was averaged across
    epochs and time, per region, per band. Theta and alpha bands were
    kept separately.

-   Edge-filter effects from band-limiting already-epoched data were
    checked (FIR filter length =1.65s against 59.9s epochs, =1.3% of
    each epoch edge) and judged negligible.

## 9. Neural proxy construction

-   Self proxy: intra-DMN connectivity (mean of the DMN×DMN
    upper-triangle, excluding the diagonal) + mean DMN activity, theta
    and alpha computed separately (4 variables).

-   Strategies proxy: mean CEN activity + mean SN–CEN connectivity +
    mean SN–DMN connectivity, theta and alpha separately (6 variables).

Note: these proxies capture static, average coupling/activity over a
60-second window, not dynamic network switching. The proposal's
language of "switching flexibility" is a temporal claim; nothing
computed here tests temporal dynamics or mediation directly.

## 10. Self-report measures

Source: LEMON's Emotion and Personality Test Battery
(Behavioural_Data_MPILMBB_LEMON/Emotion_and_Personality_Test_Battery_LEMON/).
Subject IDs in these files use LEMON's internal anonymization scheme
(sub-010xxx), distinct from the public BIDS ID (sub-032xxx) used
everywhere else in this pipeline, hence a manual ID crosswalk (recovered
from BrainVision header mismatches encountered during raw data loading)
was used to join them.

**Self** (composite scores used, not subscale averages, where both exist):
NEO-FFI (5 trait scores: Neuroticism, Extraversion, Openness,
Agreeableness, Conscientiousness), LOT-R (LOT_sumscore), TEIQue-SF
(TeiQueSF_total), STAI trait anxiety, STAXI Trait_Anger (composite of
Temperament + Reaction subscales; State Anger excluded as a state, not
trait, measure).

**Strategies:** CERQ adaptive-subscale mean (5 of 9 subscales, adaptive
only, maladaptive subscales excluded to avoid valence cancellation),
Brief COPE approach-oriented subscale mean (8 of 14 subscales,
approach/adaptive categorization is not an official COPE grouping), ERQ
reappraisal (suppression excluded from the final variable set), and all
four BIS/BAS subscales (Drive, Fun Seeking, Reward Responsiveness, BIS).

## 11. Statistical approach

-   Pearson correlation between each neural proxy variable and each
    questionnaire variable, computed separately within the Self family
    (4 neural × 9 questionnaire = 36 tests) and Strategies family (6
    neural × 7 questionnaire = 42 tests). sub-032469 excluded row-wise
    from any test involving an activity variable (NaN), included for
    connectivity-only tests.

-   Multiple-comparisons correction: Benjamini-Hochberg FDR, applied
    separately within the Self family and the Strategies family,
    matching the pipeline's consistent Self/Strategies distinction
    throughout.

-   All results are reported as descriptive/exploratory. No claim of
    confirmed effects is made anywhere in this pipeline's output.

## 12. Limitations

-   Template (fsaverage) head model, no individual anatomy.

-   N=8, not powered for inferential claims. All statistics are
    exploratory.

-   Noise covariance estimated from resting data itself.

-   Two DK regions (of 40) counted in two networks simultaneously (see
    Section 6), thus networks are not fully mutually exclusive.

-   One subject (sub-032324) has roughly a third of the eyes-closed data
    of the rest of the sample.

-   One subject (sub-032469) excluded from activity measures due to an
    unstable inverse solution.

-   COPE's approach/avoidant subscale split is a *post-hoc*
    categorization applied by this pipeline, not an official instrument
    grouping.

-   Static connectivity/activity proxies do not test the "switching
    flexibility" or mediation claims in the proposal's theoretical
    framing directly.

## 13. Repository Structure

```
.
├── README.md
├── requirements.txt
├── .gitignore
└── lemon_dmn_proxy.py
```

Raw EEG, preprocessed data, and questionnaire files are **not** included in
this repository (see Reproduction Notes) — LEMON's usage terms restrict
redistribution of participant-level data, even de-identified.

## 14. Requirements

```
mne>=1.13
mne-connectivity
mne-icalabel
pyprep
pandas
numpy
scipy
statsmodels
```

Developed and run in Google Colab (Python 3.13). A `requirements.txt` with
these packages is included in the repository root.

## 15. Reproduction notes

-   Raw EEG and behavioral data: publicly available via GWDG
    (ftp.gwdg.de/pub/misc/MPI-Leipzig_Mind-Brain-Body-LEMON/) or NITRC
    (fcon_1000.projects.nitrc.org/indi/retro/MPI_LEMON).

-   Raw .vhdr header files internally reference an older anonymization
    ID (sub-010xxx) even though the surrounding folder/filename uses the
    public BIDS ID (sub-032xxx); header DataFile/MarkerFile fields need
    correcting before MNE will load them. Questionnaire files use the
    old ID scheme directly.

-   Recommended execution order: preprocessing EC epoch extraction
    forward model (built once, reused across subjects) source
    localization/label extraction band-limited connectivity and activity
    proxy table construction questionnaire merge correlation and FDR
    correction.

## 16. Key References

- Babayan, A., et al. (2019). A mind-brain-body dataset of MRI, EEG,
  cognition, emotion, and peripheral physiology in young and old adults.
  *Scientific Data*, 6, 180308.
- Pascual-Marqui, R. D. (2007). Discrete, 3D distributed, linear imaging
  methods of electric neuronal activity. Part 1: exact, zero error
  localization. *arXiv:0710.3341*.
- Fukushima, M., Betzel, R. F., et al. (2018). Structure–function
  relationships during segregated and integrated network states of human
  brain functional connectivity. *NeuroImage*, 180, 406–416.
- Menon, V. (2011). Large-scale brain networks and psychopathology: a
  unifying triple network model. *Trends in Cognitive Sciences*, 15(10),
  483–506.
- Schlossberg, N. K. (1981). A model for analyzing human adaptation to
  transition. *The Counseling Psychologist*, 9(2), 2–18.
- Buckner, R. L., Andrews-Hanna, J. R., & Schacter, D. L. (2008). The
  brain's default network: anatomy, function, and relevance to disease.
  *Annals of the New York Academy of Sciences*, 1124(1), 1–38.
- Andrews-Hanna, J. R. (2012). The brain's default network and its
  adaptive role in internal mentation. *The Neuroscientist*, 18(3),
  251–270.
- Gross, J. J., & John, O. P. (2003). Individual differences in two
  emotion regulation processes: implications for affect, relationships,
  and well-being. *Journal of Personality and Social Psychology*, 85(2),
  348–362.
- Gramfort, A., et al. (2013). MEG and EEG data analysis with MNE-Python.
  *Frontiers in Neuroscience*, 7, 267.

## 17. Author

*Eva Villar, evagarciavillar12@gmail.com*
