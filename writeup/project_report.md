# Which parts of a flood map should a responder trust?

## Summary

After a flood, satellite flood maps are produced within hours and feed emergency response and digital
twins of the affected area. Some parts of these maps are wrong, and responders have little time to check
them. I asked whether a flood-mapping model can tell a reviewer where to look. I trained a geospatial
foundation model (TerraMind-tiny) on ImpactMesh-Flood, tested it on flood events it never saw, and ranked
320 m map cells for human review. On unseen floods, reviewing the 20% of cells where the model is least
certain finds about three quarters of the map's errors (77% in cross-validation over 58 events, 79% on a
12-event holdout), against 20% for random review. A learned error spotter adds a few points (81% and 84%),
but the gain over the model's own uncertainty is not statistically clear. On the August 2026 glacial lake
outburst flood in Nepal, the model misses most of the flooded valley and is confident while doing so. Model
uncertainty alone therefore cannot act as the reliability check for an event unlike the training data.

## Data and evaluation

ImpactMesh-Flood provides Sentinel-1 radar, Sentinel-2 optical imagery, elevation and flood masks for
Copernicus EMS flood activations, as 256 x 256 patches at 10 m. Each mask labels pixels as no water,
permanent water or flood.

The official splits keep areas of interest apart but not flood events: 52 of the test events also appear in
training. Because the question is how a model behaves on a new disaster, I built an event-disjoint split.
Events within 50 km and 30 days of each other are kept together, and the official 12 holdout events are left
untouched until the final evaluation. Model training uses 123 events, checkpoint selection 11 events, the
error spotter 60 events in five folds, and the final test the 12 holdout events. Results on the holdout are
reported with and without three events from one September 2024 storm.

Exploring the data shaped the set-up:
- About half of flood-time optical images are mostly cloud (47% of patches over half cloudy, against 6%
  before the flood), so the main model uses radar and elevation only.
- In radar, flood and permanent water look alike during the flood; they differ only in the change from the
  image before the flood (flood darkens by about 8 dB). The model therefore receives both time steps.
- Flood pixels are concentrated in a few events (7 of 123 hold half of them), so scores are reported per
  event, not pooled over pixels.
- One event held 839 of 1,435 validation patches; validation was capped at 150 patches per event so the
  checkpoint choice is not driven by a single flood.

## Flood maps on unseen events

TerraMind-tiny was fine-tuned with the ImpactMesh recipe (batch 16, learning rate 1e-4, dice loss, early
stopping on validation loss). The baseline is a radar threshold (VV below -14.6 dB, chosen on non-holdout
data), scored by its own rule, under which permanent water is not counted against it.

| Mean flood IoU per event, holdout | all 11 events with flood | without the storm (9) |
|---|---|---|
| Radar threshold | 0.258 | 0.238 |
| Radar + elevation model | 0.377 | 0.313 |
| Optical + radar + elevation model | 0.408 | 0.366 |

Both models beat the threshold: radar + elevation by +0.120 (95% interval +0.020 to +0.215, better in 9 of
11 events); with optical images by +0.150 (+0.076 to +0.224). Adding optical images does not give a clear
average gain over radar alone (+0.034, interval -0.053 to +0.142), but it rescues the two events where the
radar model fails almost completely (EMSR867: 0.03 to 0.46; EMSR251: 0.02 to 0.30).

## Where should a reviewer look?

Each patch is split into 32 x 32 pixel cells (320 m). Within an event, cells are ranked and a reviewer checks
the top 20%; the score is the share of the event's map errors inside those cells. The set-up was fixed before
any results were seen.

| Errors caught by reviewing 20% of cells | cross-validation (58 events) | holdout (12 events) |
|---|---|---|
| Random | 19% | 19% |
| Predicted flood fraction | 71% | 68% |
| Model uncertainty (entropy) | 77% | 79% |
| Error spotter (gradient-boosted trees on cell features) | 81% | 84% |
| Perfect ranking (upper bound) | 96% | 95% |

Errors are concentrated: three quarters of cells contain none. The model's own uncertainty already points to
most of them. The spotter's gain over entropy is +3.5 points in cross-validation, with an interval that
includes zero (-0.9 to +7.2), so it is not established. Where the map is wrong over more than 20% of the area
(EMSR867, EMSR845), even a perfect ranking catches only about 70% of the errors.

## The Nepal glacial lake outburst flood

On the night of 25 August 2026, a glacial lake outburst sent a debris flood down the Bhote Koshi and
Trishuli valleys. No Sentinel-2 image after the flood was usable for more than three weeks (54-97% cloud on
every pass), so responders had radar only, and the radar + elevation model was applied to the Sentinel-1
images of 16 and 28 August.

The case is far outside the training data: 63% of the mapped flood area is steeper than 15 degrees (training
floods: 0.6%), and two days after the event the radar shows little open water in the mapped corridor. The
model's map agrees with the UNOSAT reference where it marks water (precision 0.95) but finds only 6% of the
reference area. Reviewing 20% of cells catches 33% of the errors with entropy and 38% with the spotter,
against 22% at random. Most importantly, the model is confident: its average uncertainty on Nepal is lower
than on 69% of the other events, and across all events average uncertainty does not track map quality.

## What this means

For flood events like those it was trained on, the model's uncertainty is a useful guide for human review:
a fifth of the map holds most of its mistakes. For a disaster unlike the training data, the model fails
silently. A reliability layer for emergency mapping or digital-twin updates therefore needs a second
component that asks whether the input itself resembles what the model has seen, for example terrain,
sensor conditions or event type, before any map is accepted.

## Limitations

- The holdout has 12 events, so several differences have wide intervals.
- Nepal is a single event, and its reference outlines the damaged valley floor, not only water.
- The DEMs used for training and for Nepal are different products, so terrain comparisons are approximate.
- Models were trained once each; run-to-run variation was not measured.

## Reproducibility

All notebooks, with their outputs, are at https://github.com/gjoshi22/flood-triage. Data: ImpactMesh-Flood
(IBM/DLR/ESA), Copernicus Sentinel-1 and Sentinel-2 via Microsoft Planetary Computer, Copernicus DEM GLO-30,
the UNOSAT flood extent for Nepal and Copernicus EMS EMSR927.
