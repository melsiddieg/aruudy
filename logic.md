# Feet Inference Logic & Anomaly Handling

This document outlines how Aruudy infers the rhythmic feet (تفعيلات) of an Arabic hemistich and detects common prosodic deviations (زِحاف and عِلَّة).

## 1. From Text to Prosody Units

1. **Normalization**: remove tatweel, fix diacritics, etc. (`prosody.normalize`).
2. **Prosody Form**: apply _prosody_ transformations (_prosody_del, _prosody_add_) to get the canonical prosodic string. (`prosody.prosody_form`).
3. **Arabic Meter (ameter)**: scan the prosody form letter‑by‑letter; each vowelled letter (`watad`) becomes `w`, each unvowelled or non‑vowelled letter (`sabab`) becomes `s`, yielding the `ameter` string. (`meter.get_ameter`).
4. **Western‑style Meter (emeter)**: convert `ameter` by mapping each `ws`→`-` (long), each lone `w`→`u` (short). (`meter.a2e_meter`).

## 2. Bahr Matching & Foot Segmentation

### 2.1. Bahr Search
The library has a registry of all 16 Arabic meters (buḥūr), each represented by one or more `BahrForm` sequences of feet.

- **search_bahr**: for each registered meter, try to validate the entire `emeter`+unit list against one of its `BahrForm`s. (`meter.search_bahr`)

### 2.2. BahrForm Validation
A `BahrForm` is a specific sequence of feet classes (e.g. `WWSWS`, `WSWWSWS`, …).

- **BahrForm.validate**: iterate through its feet in order; for each foot:
  1. Attempt to match the beginning of the remaining `emeter` via `Tafiila.process`.
  2. If matched, wrap the chosen `TafiilaComp` (foot form) in a `Part` and consume that many prosodic units.
  3. Proceed to the next foot with the leftover `emeter` & units.
  4. If all feet match and consume the entire `emeter`, the meter is found.

### 2.3. Foot Matching (`Tafiila.process`)
Each foot class (`WSWWS`, `WWSWS`, …) defines multiple `TafiilaComp` variants, each with:
- a TafiilaType (e.g. `SALIM`, `QABDH`, `HADF`, …),
- a mnemonic (Arabic pattern),
- an English‑meter snippet (`emeter`).

On initialization, a foot filters its forms to only those TafiilaTypes allowed in the current meter (i.e. which deviations are permitted in that meter form).

`Tafiila.process(text_emeter)` tries each allowed form in order. If `text_emeter` starts with that form’s `emeter` string, it returns a copy of the matching `TafiilaComp` plus the leftover `emeter` tail.

## 3. Zihaf & Illa (زوْحَاف وعِلَّة)
All common prosodic anomalies (زِحاف, عِلَّة) are encoded as members of the `TafiilaType` enum (e.g. `QABDH`, `KHABN`, `HADF`, `QASR`, `IDHMAR`, …).  

- **Zihaf (زِحاف)**: changes within the foot that shorten or alter its standard pattern (e.g. `قبض` → dropping a vowel).
- **Illa (عِلَّة)**: changes at the word’s boundary that add or shift a vowel (e.g. `إضمار`).

Whether a given meter form allows a particular anomaly is determined by the variant list passed when constructing its feet in `meter.py`. For example:
```python
# allow only the سالم (plain) and قبض (zihaf) forms of فَعُولُنْ
foot.WWSWS([FT.SALIM, FT.QABDH])
```

With this setup, the segmentation algorithm automatically recognizes deviations: if the `emeter` matches the `QABDH` variant of `WWSWS`, the resulting `Part.type` will be `QABDH` and the mnemonic will reflect the altered foot.

## 4. Putting It All Together
1. Convert text → `ameter` & `emeter`.
2. For each Bahr, run `search_bahr` → gives the best matching `Bahr` + list of `Part` objects.
3. Each `Part` object carries:
   - its foot type (`Part.type` → a `TafiilaType`, i.e. which anomaly, if any),
   - the Arabic & English meter slices,
   - the mnemonic pattern,
   - the exact substring of the prosody text that formed this foot.

This modular design cleanly separates core prosody conversion, meter lookup, and foot‑level anomaly recognition.
