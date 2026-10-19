# Can emergency responders trust a satellite flood map?

## Summary

On August 26, 2026, a deadly flash flood struck northern Nepal's Rasuwa district, where more than a thousand people died and about 4,700 were reported missing. Typically, hours after a flood, satellite maps guide where emergency responders and rescuers provide emergency support and supplies. But parts of these maps are wrong, and because floods like these affect such a huge area, thoroughly checking everything is time-consuming and demanding. Thus, I wanted to understand whether a flood-mapping model can help a reviewer identify its own mistakes, especially on floods it has never seen. To test this, I fine-tuned a geospatial foundation model, TerraMind-tiny, on radar and elevation data from 134 past floods and tested it on floods kept completely separate from training. On those floods, reviewing the 20% of the map where the model was least confident caught about three quarters of its errors (77–79%, vs 19% with random checking). Then I tested the model on the Rasuwa flash flood, a steep mountain disaster unlike its training floods. The model found only 6% of the affected area, yet it was more confident there than on most other floods. So the answer to my question depends on whether the flood resembles what the model has seen before. My takeaway: a model's own uncertainty helps on familiar floods, but on an unfamiliar flood it fails confidently, so it can't warn anyone.

## Data and evaluation

To perform this analysis, I used a publicly available flood dataset from IBM, DLR and ESA called ImpactMesh-Flood, built from 206 Copernicus emergency mapping activations. Each sample in the dataset is a 256 × 256 patch at 10 m resolution with Sentinel-1 radar, Sentinel-2 optical imagery, elevation and a mask marking each pixel as no water, permanent water or flood.

My curiosity was how reliable a model can be on a flood it has never seen. Thus, the test set had to contain new floods. Upon further exploring the official split, I found it keeps map areas apart but not flood events: 52 of the 67 test events appeared in training as well. As a result, I rebuilt the split by event, keeping any events within 50 km and 30 days of each other bundled together. This gave 123 events for training, 11 for checkpoint selection, 60 for the error spotter (five folds), and the official 12-event holdout, which was not touched until the end. Three holdout events came from the same September 2024 storm as some training events, so I report holdout results both including and excluding them.

When I performed the exploratory data analysis, my plan changed further. First was the presence of clouds: 47% of optical patches taken during the flood events were mostly cloud, against 6% before, so my main model uses only radar and elevation. Another observation I made was that flood water and permanent water look the same in radar during a flood. The only way to separate them is comparing the before and during images, so the model sees both dates. Finally, a few flood events dominate: 7 of 123 training events hold half the flood pixels, and one event held 839 of 1,435 validation patches. So I scored every event separately and capped validation at 150 patches per event.

## Flood maps on unseen events

Checking a model's mistakes only makes sense if its maps are decent to begin with. Thus, my first step was to train a model whose maps were worth checking. I fine-tuned TerraMind-tiny, a small geospatial foundation model, using the training recipe that comes with ImpactMesh. To see whether it actually learned something useful, I compared it against a simple baseline: a radar threshold, where any pixel darker than -14.6 dB in VV radar counted as water. I chose this threshold using only non-holdout data. To measure each map, I used flood IoU, the overlap between the predicted and true flood area, from 0 to 1, averaged per event.

| Mean flood IoU per event, holdout | All 11 events with flood | Without the 2024 storm (9) |
| --- | --- | --- |
| Radar threshold | 0.258 | 0.238 |
| Radar + elevation model | 0.377 | 0.313 |
| Optical + radar + elevation model | 0.408 | 0.366 |

The radar and elevation model beat the threshold by 0.12 on average and did better on 9 of the 11 events. When I added optical imagery, the average went up slightly, but the gain over radar alone was not clear (+0.03, with an interval from -0.05 to +0.14). Where optical imagery really helped was on the two events where the radar model failed almost completely: EMSR867 rose from 0.03 to 0.46 and EMSR251 from 0.02 to 0.30. Because optical imagery is often covered by clouds during floods, I kept the radar and elevation model as my main model for the rest of the study.

## Where should a reviewer look?

Once a working model was built, the main research question that I wanted to explore was: if a reviewer or emergency responder only has time to look at a part of the map, where should they focus? To do this, I split each patch into 32 × 32 pixel cells (320 m on a side) and, within each event, ranked the cells by different scores. The responder checks the top 20%, and the score is the share of that event's map errors inside those cells. I fixed this set-up before looking at any results, so I could not tune it to make one method look better.

| Errors caught by reviewing 20% of cells | Cross-validation (58 events) | Holdout (12 events) |
| --- | --- | --- |
| Random | 19% | 19% |
| Predicted flood fraction | 71% | 68% |
| Model uncertainty (entropy) | 77% | 79% |
| Error spotter (gradient-boosted trees) | 81% | 84% |
| Perfect ranking (upper bound) | 96% | 95% |

One thing I noticed was that the errors are very concentrated: three quarters of cells had no errors at all. The model's own uncertainty, measured as entropy (how torn the model is between answers for each pixel), already pointed to most of the rest, catching 77–79% of the errors against 19% for random checking. I also trained an error spotter, a small model that predicts where the map is wrong. It added a few points, but in cross-validation its gain over entropy (+3.5 points) had an interval that includes zero (-0.9 to +7.2), so I cannot claim it reliably helps. On the holdout without the 2024 storm, the gain was larger and clearer (+8.9 points, interval +3.1 to +15.4), which is promising but not conclusive. Some events stayed hard for every method: on EMSR867 and EMSR845, where the map is wrong over large areas, even a perfect ranking catches only about 70% of the errors.

## The Rasuwa flash flood, Nepal

The real test was the Rasuwa flash flood. On the morning of August 26, 2026, a glacier collapse triggered an ice-rock avalanche that sent a debris flood down the Bhote Koshi and Trishuli valleys. After the flood, no Sentinel-2 image was usable for more than three weeks (54–97% cloud on every pass), so a responder would have had radar only. Thus, I applied my radar and elevation model to the Sentinel-1 images from August 16 (before) and August 28 (after), and compared its map with UNOSAT's reference map, which outlines 37.4 km² of affected area inside a 363 km² analysis area.

This event is very different from what the model learned from. 63% of the mapped flood area is steeper than 15 degrees, against only 0.6% in the training floods, and the flood left mud and debris rather than open standing water. Where the model said water, it was almost always right (precision 0.95), but it found only 6% of the reference area. Reviewing 20% of the cells still helped a little, catching 33% of the errors with entropy and 38% with the error spotter, against 22% at random.

The most important observation I made was that the model was not worried. Its average uncertainty on Nepal was lower than on 69% of the other events. In other words, it missed almost the whole flooded valley while looking confident, so nothing in its output would have warned a responder to double-check.

## What this means

For floods similar to the ones the model was trained on, its uncertainty is a useful guide: reviewing a fifth of the map catches most of its mistakes. But for a disaster unlike the training data, the model can fail silently, being wrong and confident at the same time. So a reliability check for emergency mapping, or for updating a digital twin from satellite maps, should not rely on the model's uncertainty alone. It also needs a step that asks whether the input itself looks like what the model has seen before, for example the terrain, sensor conditions or type of event. Where the satellite map cannot be trusted, other evidence such as ground reports or geotagged social media becomes the most valuable.

## Limitations

- The holdout has only 12 events, so several differences have wide intervals. This is why I also report cross-validation over 58 events.
- Nepal is a single event, so it is a stress test, not a general claim. Its reference map outlines the damaged valley floor, including mud and rock, not only water.
- The elevation data used for training and for Nepal come from different products, so terrain comparisons are approximate.
- Each model was trained once, so run-to-run variation was not measured.
- The model uses two time steps (before and during the flood), not all four in the dataset.
- Event-disjoint splits prevent label leakage, but TerraMind's pretraining may still have seen imagery of some of these places.

## Reproducibility

All notebooks, with their outputs, are at https://github.com/gjoshi22/flood-triage. Data: ImpactMesh-Flood (IBM/DLR/ESA), Copernicus Sentinel-1 and Sentinel-2 via Microsoft Planetary Computer, Copernicus DEM GLO-30, the UNOSAT flood extent for Nepal, and Copernicus EMS EMSR927.
