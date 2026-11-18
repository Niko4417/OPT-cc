# Local transcription of interviews and videos

Date: 2026-11-18. Ticket: [Find how to transcribe interviews and videos locally](https://github.com/Niko4417/OPT-cc/issues/28). Map: [Project memory reference installation](https://github.com/Niko4417/OPT-cc/issues/1). These are research findings, not owner decisions. The owner adopts, changes or rejects them in [Model the sources around repositories and project artifacts](https://github.com/Niko4417/OPT-cc/issues/29).

## Question

Which on-device speech-to-text setup turns audio and video project artifacts (interviews, recorded workshops) into transcripts on the V1 Mac (Apple Silicon, 64 GB)? The comparison covers German and English accuracy (word error rate, WER, on published benchmarks), speaker separation, word or segment timestamps for time-range citations, speed and memory next to the lab models, native install without Docker, and licenses that allow free commercial use. The output must fit the workshop result transcript format of ADR-0002: one speaker turn per line, with timestamps. The ticket also asks how video audio is extracted and whether slides shown in a video can be captured.

## Answer

Use two models in one transcription step of the document ingestion job:

1. **Speech-to-text: `nvidia/parakeet-tdt-0.6b-v3`** (CC-BY-4.0, 0.6 billion parameters), run on the Mac with `parakeet-mlx` (Apache-2.0). It has 4.12% German WER and 7.04% English WER on the Open ASR Leaderboard, against 4.00% and 8.24% for Whisper large-v3, and it runs about 12 times faster than Whisper large-v3 on the leaderboard GPU. It returns punctuated, cased text with word, segment and character timestamps, detects the language itself across 25 European languages, and handles up to 3 hours in one pass with local attention.
2. **Speaker separation: `pyannote/speaker-diarization-community-1`** (CC-BY-4.0) in `pyannote.audio` 4 (MIT). It has a diarization error rate (DER) of 17.0% on AMI headset audio and 11.2% on VoxConverse, beats pyannote 3.1 on 10 of 12 benchmark sets, and has no fixed speaker limit. Its "exclusive" output gives one speaker per moment, which is what word-to-speaker assignment needs.

A short glue step assigns each word to the speaker whose exclusive segment covers the word's midpoint, merges consecutive words of one speaker into a turn, and writes one `**Speaker N:**` line per turn with its start and end time. A human replaces the speaker labels with names. Video goes through `ffmpeg` first (`-vn -ac 1 -ar 16000`). Slides in a video are captured as one frame per scene change, OCR'd with the same Apple Vision OCR the document ingestion uses, and anchored to the scene's time range.

Both models together need about 2 to 3 GB (estimate, not measured), which fits in the 4.4 to 6.7 GB margin left by the [memory fit research](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md). Nothing needs Docker. All weights and code allow commercial use; CC-BY-4.0 requires attribution in a NOTICE file.

Two upgrades to test in the build phase, not defaults: `FluidVoice-Labs/fluid-speech-pro`, a CC-BY-4.0 fine-tune of Parakeet v3 with 3.76% German WER but released on 2026-11-13, and `Qwen/Qwen3-ASR-1.7B` (Apache-2.0) with 3.88% German WER but a 20-minute limit per call and timestamps only through a separate aligner. FluidAudio (Apache-2.0, Swift) runs the same two models on the Neural Engine at 200 to 300 times real time on M-series Macs and is the fallback if the Python path is too slow.

## Accuracy

### German and English WER

The [Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard) (version 1.16.1, last updated 2026-11-18, method in [Srivastav et al., arXiv 2510.06961](https://arxiv.org/abs/2510.06961)) is the only independent benchmark that runs all candidates with one script and one text normalizer. Its multilingual view scores German on the test splits of Mozilla Common Voice and FLEURS. Both are read speech, so these numbers are a floor: workshops with crosstalk and room microphones will score worse. The table shows the open-weight models filtered to German and English (read from the leaderboard's Multilingual tab with only those two languages selected). RTFx is seconds of audio per second of compute on an NVIDIA A100 at batch size 64 ([paper, section 3](https://arxiv.org/html/2510.06961v3)), so it ranks models but does not predict Mac speed.

| Model | Params (B) | Weights license | German WER | English WER | RTFx | Timestamps | Fits V1 |
| --- | ---: | --- | ---: | ---: | ---: | --- | --- |
| NVIDIA Nemotron 3.5 Transcribe 30B-A4B | 33.7 | OpenMDW-1.1 | 2.79 | 6.13 | 447 | not checked | No, 33.7B parameters do not fit next to the lab pair |
| Voxtral Small 24B 2507 | 24 | Apache-2.0 | 2.58 | 7.44 | 92 | not checked | No, too large |
| Cohere Transcribe 03-2026 | 2.07 | Apache-2.0 | 3.10 | 6.72 | 765 | none | No, the card states no timestamps and no diarization |
| Fluid Speech Pro (Parakeet v3 fine-tune) | 0.63 | CC-BY-4.0 | 3.76 | 6.06 | 3,472 | word, segment | Candidate, five days old |
| Qwen3-ASR-1.7B | 2.35 | Apache-2.0 | 3.88 | 6.15 | 602 | through Qwen3-ForcedAligner | Candidate, 20-minute calls |
| Whisper large-v3 | 1.54 | MIT | 4.00 | 8.24 | 399 | segment, word via DTW | Fallback |
| Canary 1B v2 | 1.0 | CC-BY-4.0 | 4.06 | 8.43 | 1,581 | via NeMo Forced Aligner | Slower than Parakeet, no gain |
| Parakeet TDT 0.6B v3 | 0.63 | CC-BY-4.0 | 4.12 | 7.04 | 4,795 | word, segment, character | **Recommended** |
| Voxtral Mini 3B 2507 | 4.68 | Apache-2.0 | 4.27 | 8.32 | 165 | not checked | No gain |
| Granite Speech 4.1 2B NAR | 2.25 | Apache-2.0 | 4.32 | 7.16 | 1,757 | not checked | No gain |
| Whisper large-v3-turbo | 0.81 | MIT | 6.12 | 8.75 | 574 | segment, word | Weaker in German |
| VibeVoice-ASR | 8.67 | MIT | 14.1 (secondary mirror) | not checked | not checked | yes, with speakers | No, weak German and 17 GB in bf16 |

Licenses and parameter counts come from the Hugging Face model API for each repository; the Whisper weights license is from the [Whisper README](https://github.com/openai/whisper#license) ("code and model weights are released under the MIT License"). The VibeVoice-ASR German figure is from a [secondary mirror of the leaderboard](https://paperswithcode.co/benchmark/open-asr-leaderboard-german), not the live table.

The papers behind the models agree with the leaderboard on German:

- The Parakeet v3 and Canary v2 paper ([Sekoyan et al., arXiv 2509.14128, Tables 11 and 13](https://arxiv.org/html/2509.14128v2)) reports German WER on FLEURS of 5.04 for Parakeet v3, 4.40 for Canary v2 and 4.30 for Whisper large-v3, and on CoVoST2 of 4.84, 5.53 and 5.94. Parakeet v3 is behind Whisper on FLEURS and ahead on CoVoST2, so the two are within about one point of each other on German read speech.
- The [Parakeet v3 model card](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) gives the same 5.04 (FLEURS) and 4.84 (CoVoST) for German. Its noise test, averaged over nine sets, rises from 6.34% WER on clean audio to 7.12% at 10 dB signal-to-noise and 8.23% at 5 dB.
- The [Qwen3-ASR technical report](https://arxiv.org/html/2601.21337) (Tables A.2a and A.2b) gives Qwen3-ASR-1.7B German WER of 3.92 on FLEURS, 6.05 on MLS and 5.85 on Common Voice, and 17.19 on the conversational MLC-SLM German set. The 0.6B model is much weaker in German (6.48 on FLEURS).
- Whisper large-v3-turbo loses more in German than its card's "minor quality degradation" suggests. On the leaderboard it has 6.12% German WER against 4.00% for large-v3. A German evaluation by primeline ([model card](https://huggingface.co/primeline/whisper-large-v3-turbo-german)) shows the same direction: 3.85 against 3.48 on Common Voice 19 and 8.30 against 7.88 on Tuda-De.

The only conversational German numbers found are on MLC-SLM: 16.30% WER for VibeVoice-ASR ([VibeVoice-ASR report, Table 2](https://arxiv.org/html/2601.18184v2)) and 17.19% for Qwen3-ASR-1.7B. No conversational German result exists for Parakeet v3. Expect roughly three to four times the read-speech WER on real workshop audio, and measure it (see "Not verified").

### Hallucination

Whisper can produce text that was never said. [Koenecke et al., "Careless Whisper", FAccT 2024](https://arxiv.org/abs/2402.08021) found hallucinated phrases in 1.4% of audio segments transcribed by Whisper in 2023, 38% of them harmful, more often after long pauses. The [Whisper paper](https://arxiv.org/abs/2212.04356) (section 6) names repetition loops and "complete hallucination" as known failure modes of its 30-second sliding window. The Cohere Transcribe card warns that its model transcribes non-speech noise and recommends voice activity detection. For a project memory that cites time ranges as evidence, invented text is worse than a missed word. Parakeet is a transducer model that emits tokens aligned to audio frames instead of generating free text from a 30-second window, so this failure mode is less likely, but no study measures it. The glue step drops words that fall outside the diarization's speech regions, which removes text produced during silence.

### Speaker separation

[pyannote community-1](https://huggingface.co/pyannote/speaker-diarization-community-1) reports DER (no collar, overlapping speech scored) of 17.0% on AMI headset mix, 19.9% on AMI single distant microphone, 11.2% on VoxConverse and 20.2% on DIHARD 3, all better than pyannote 3.1 except REPERE (8.9 against 7.9). pyannote's paid precision-2 model is 4 to 10 points better but runs only on pyannoteAI servers, so it breaks the on-device default. The [pyannote.audio README](https://github.com/pyannote/pyannote-audio) gives 31 seconds per hour of AMI audio on an NVIDIA H100.

Other speaker options fail a requirement:

- NVIDIA Streaming Sortformer v2 ([card](https://huggingface.co/nvidia/diar_streaming_sortformer_4spk-v2), CC-BY-4.0) detects at most 4 speakers. Workshops often have more.
- VibeVoice-ASR transcribes and separates speakers in one pass (DER 11.92% on AMI headset audio, [report Table 2](https://arxiv.org/html/2601.18184v2)), but its German WER is weak and its 8.67 billion parameters need about 17 GB in bf16.
- Granite Speech 4.1 2B Plus ([card](https://huggingface.co/ibm-granite/granite-speech-4.1-2b-plus), Apache-2.0) labels speakers inline, but was trained on clips of up to 10 minutes. Speaker numbers would restart in every chunk of a two-hour workshop, and the card publishes no German WER.
- Docling's video pipeline separates speakers with Resemblyzer clustering, for which no DER is published, and offers no speaker separation for audio files ([Docling audio docs](https://docling-project.github.io/docling/usage/processing_audio_media)).
- DiariZen may inherit CC BY-NC 4.0 from its segmentation weights ([Anuvani-Diarization, arXiv 2610.20327](https://arxiv.org/pdf/2610.20327)); not checked further.

No German diarization benchmark was found. Diarization depends on voices more than on language, and the community-1 table includes French (REPERE) and Chinese (AISHELL-4, AliMeeting) sets, but German workshop audio still needs a check.

## Timestamps and the ADR-0002 format

[ADR-0002](https://github.com/Niko4417/OPT-cc/blob/claude/mattpocock-skills-wayfinder-36692b/docs/adr/0002-prescribe-a-markdown-format-for-workshop-results.md) prescribes front matter `id`, `type: workshop-result`, `kind: transcript`, `date`, `language`, `title`, `participants` and `decisions`, and one `**Name:**` line per speaker turn with optional timestamps. [ADR-0006](https://github.com/Niko4417/OPT-cc/blob/claude/mattpocock-skills-wayfinder-36692b/docs/adr/0006-define-project-artifacts-by-origin-and-take-audio-and-video.md) requires that answers cite time ranges.

What each part delivers:

- Parakeet v3 returns word, segment and character timestamps ([card](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3)). `parakeet-mlx` exposes sentences and tokens with start and end times through its Python API and writes `txt`, `srt`, `vtt` or `json` from its CLI ([README](https://github.com/senstella/parakeet-mlx)).
- community-1 returns `exclusive_speaker_diarization`, a version with one speaker at a time that its card describes as made to align with transcription timestamps.
- WhisperX ([README](https://github.com/m-bain/whisperX)) does the same merge with wav2vec2 forced alignment and the same pyannote model. Its approach is the reference for the glue step; the package itself is not needed, because Parakeet already returns word times.

The glue step:

1. Run diarization and transcription on the same 16 kHz mono file.
2. Give each word the speaker whose exclusive segment contains the word's midpoint. Drop words that fall in no speech segment.
3. Merge consecutive words of one speaker into a turn. Split a turn at pauses longer than about 2 seconds or after about 60 seconds, so that a citation points at a short span.
4. Write one line per turn:

```markdown
---
id: WR-2026-11-12-interview-billing
type: workshop-result
kind: transcript
date: 2026-11-12
language: de
title: Interview on the billing batch
participants: [Speaker 1, Speaker 2]
decisions: []
source: interview-billing.m4a
source-sha256: 3f5a...
asr-model: nvidia/parakeet-tdt-0.6b-v3@541d1f99c6b0
diarization-model: pyannote/speaker-diarization-community-1@3533c8cf8e36
model-location: on-device
speakers-confirmed: false
---

**Speaker 1:** [00:12:31-00:13:05] Der Abrechnungslauf startet jede Nacht um zwei Uhr.
**Speaker 2:** [00:13:05-00:13:40] And it still reads the old customer table?
```

The keys after `decisions` are suggestions for provenance and the model policy, not part of ADR-0002. Citations then point to `interview-billing.m4a` at `00:12:31-00:13:05`. Diarization labels are anonymous, so the transcript keeps `Speaker 1` until a consultant maps labels to names and sets `speakers-confirmed: true`, the same human confirmation rule that ADR-0004 applies to decision logs. Speaker labels are not matched across recordings.

## Speed and memory on the V1 Mac

No transcription model was run for this note: `ffmpeg`, `parakeet-mlx` and `pyannote.audio` are not installed on the M5 Pro, and installing them was out of scope. The figures below come from published measurements on M-series Macs.

| Step | Published speed | Source | Weights |
| --- | --- | --- | --- |
| Parakeet v3, Core ML on the Neural Engine (FluidAudio) | German FLEURS 220.9x real time, 5.9% WER, on M4 Pro; 60-minute file at 335x on M5 Pro | [FluidAudio benchmarks](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Benchmarks.md) | about 480 MB download |
| Parakeet v3, MLX on the GPU (`parakeet-mlx`) | no long-file figure published; 120-second chunks with 15-second overlap by default | [README](https://github.com/senstella/parakeet-mlx) | 0.63B parameters, 1.25 GB in bf16 |
| community-1, PyTorch | about 1.5 to 2x real time on CPU, 20 to 25x on Apple MPS | FluidAudio benchmarks (vendor of a competing port) | segmentation and ResNet34 embedding, under 0.1 GB (estimate) |
| community-1, Core ML (FluidAudio) | 323x real time on M5 Pro; AMI DER 20.3% against pyannote's 17.0% under the pyannote protocol | FluidAudio benchmarks | same weights, converted |
| Whisper large-v3 in whisper.cpp | no long-file figure for M5 Pro found | [whisper.cpp README](https://github.com/ggml-org/whisper.cpp) | about 3.9 GB in memory |

For a one-hour workshop that means seconds for Parakeet on the Neural Engine, and about 2.5 to 3 minutes for pyannote on MPS. pyannote on the CPU would take 30 to 40 minutes per hour of audio, so the job must move the pipeline to MPS. The build phase should time `parakeet-mlx` and pyannote on MPS with a one-hour German recording.

Memory: Parakeet in bf16 (1.25 GB) plus pyannote and audio buffers is about 2 to 3 GB at peak (estimate). The memory fit research leaves 6.65 GB at typical load and 4.41 GB at the upper bound with the smaller prompt caches, so transcription fits next to the lab pair, one file at a time. If memory runs short, transcribe while the checker idles, as that research suggests for its own fallbacks. The Neural Engine path (FluidAudio) also leaves the GPU to `mlx_lm.server`, because Apple Silicon has one memory pool but separate GPU and Neural Engine compute.

Whisper large-v3 would need about 3.9 GB and is about 12 times slower than Parakeet on the leaderboard GPU, for a German WER within one point of Parakeet.

## Install without Docker

| Part | Install | License |
| --- | --- | --- |
| `parakeet-mlx` | `uv tool install parakeet-mlx` or `pip install parakeet-mlx` | Apache-2.0 |
| Parakeet v3 MLX weights | `mlx-community/parakeet-tdt-0.6b-v3` from Hugging Face | CC-BY-4.0 |
| `pyannote.audio` 4.x | `pip install pyannote.audio` | MIT |
| community-1 weights | Hugging Face, gated: accept the conditions once with an account and token, then cache or mirror (CC-BY-4.0 allows redistribution with attribution) | CC-BY-4.0 |
| `ffmpeg` | `brew install ffmpeg` | Homebrew build is GPL-3.0-or-later ([formula](https://formulae.brew.sh/formula/ffmpeg)) |
| PySceneDetect (optional) | `pip install scenedetect` | BSD-3-Clause |
| FluidAudio (alternative) | Swift package, `fluidaudiocli` on macOS | Apache-2.0 code; Core ML weights CC-BY-4.0 |

All of these run as native processes. The app calls `ffmpeg` as a separate program and does not link or ship it, so its GPL build does not affect the app's license. A product that ships FFmpeg would need an LGPL build linked dynamically, following [FFmpeg's license checklist](https://ffmpeg.org/legal.html). Docling's own audio and video support also needs `ffmpeg` on the `PATH`, so the document ingestion job installs it anyway.

For Linux and Windows later, the same two models run without the Mac-specific ports: Parakeet through NVIDIA NeMo, and pyannote through PyTorch on CPU or CUDA. Only `parakeet-mlx` and FluidAudio are Mac-only.

## Model policy

ADR-0006 says the model policy covers the transcription model like any other model. The transcription models run inside the ingestion job, not behind LiteLLM, so the job itself must declare its model location. The suggested front matter records `asr-model`, `diarization-model` and `model-location: on-device` per transcript. A later client-hosted or cloud transcription route would have to pass the project memory's model policy before the job may send audio to it. LiteLLM documents an `audio_transcription` mode for OpenAI and a few cloud providers ([LiteLLM docs](https://docs.litellm.ai/docs/audio_transcription)); whether it can route to a local transcription server was not checked, and V1 does not need it.

## Video: audio extraction and slides

### Audio

`ffmpeg -i workshop.mp4 -vn -ac 1 -ar 16000 -c:a pcm_s16le workshop.wav` drops the video stream and writes 16 kHz mono PCM, the input format both models expect. The same command reads MP4, MOV, MKV and WEBM. Keep the original file as the cited source; the WAV is a temporary file.

### Slides

Slides shown in a recorded workshop can be captured, with one caveat: no benchmark for slide detection in screen recordings was found, so quality needs a prototype.

1. Detect slide changes with PySceneDetect's content detector or `ffmpeg`'s `select='gt(scene,0.3)'` filter. A shared screen changes little between slides, so the threshold needs tuning on real recordings.
2. Keep one sharp frame per scene and drop near-duplicates with a perceptual hash.
3. OCR each frame with Apple Vision, the OCR that [Find how to ingest folders and documents beyond the repository](https://github.com/Niko4417/OPT-cc/issues/16) picked for German.
4. Store each slide as a chunk anchored to its time range (`workshop.mp4` at `00:41:10-00:44:55`), next to the transcript turns of the same range.

Docling's video pipeline already samples one frame per scene change (`--video-sampling-mode scene`, at most 200 frames by default) and stores frames as pictures in the document, but its docs do not say it OCRs them ([Docling audio docs](https://docling-project.github.io/docling/usage/processing_audio_media), [video pipeline example](https://docling-project.github.io/docling/_generated/examples/video_pipeline/)). Its frame sampler can replace step 1; steps 3 and 4 stay glue code. If the slide deck itself is available, add it as a project artifact: its text is exact and the Docling job already reads slides.

## Options not recommended

- **Docling's ASR pipeline as the transcriber.** On Apple Silicon it runs Whisper turbo through mlx-whisper by default (6.12% German WER on the leaderboard), keeps segment times but not word times, and separates speakers only in video, with Resemblyzer. The transcript Markdown should still go through the Docling job like any other workshop result.
- **WhisperX.** BSD-2-Clause and widely used, but it runs Whisper through faster-whisper (CTranslate2), whose Apple GPU backend is experimental and never chosen automatically ([CTranslate2 README](https://github.com/OpenNMT/CTranslate2)). Its diarization is the same pyannote model.
- **Apple SpeechAnalyzer.** On-device and free, but it needs macOS 26, runs only on Apple platforms, and Apple publishes no WER. The only independent-looking English figure is 14.0% on earnings calls against 12.8% for Whisper small.en, from Argmax, a competitor ([Argmax blog](https://www.argmaxinc.com/blog/apple-and-argmax)). German figures exist only on blogs without method.
- **Cohere Transcribe.** Best German WER of the small models (3.10%), Apache-2.0, but no timestamps ([card](https://huggingface.co/CohereLabs/cohere-transcribe-03-2026)). A forced aligner would have to add them afterwards.
- **TheStage AI Whisper large-v3-turbo.** Strong German WER on the leaderboard (3.04%), but its Mac SDK needs an API token from app.thestage.ai ([card](https://huggingface.co/TheStageAI/thewhisper-large-v3-turbo)), which conflicts with offline, local-first use.
- **Large speech LLMs** (Nemotron 3.5 Transcribe 30B-A4B, Voxtral Small 24B, VibeVoice-ASR 9B). Better or similar German WER, but each needs 17 GB or more and would not fit next to the lab pair.

## Upgrades worth testing

- **Fluid Speech Pro** ([card](https://huggingface.co/FluidVoice-Labs/fluid-speech-pro)) has the same architecture and size as Parakeet v3, so it is a drop-in swap in `parakeet-mlx` and FluidAudio. It has 3.76% German and 6.06% English WER on the leaderboard against 4.12% and 7.04% for Parakeet v3. It was released on 2026-11-13, its training data is not described and no paper exists, so it is not yet proven.
- **Qwen3-ASR-1.7B** ([report](https://arxiv.org/html/2601.21337)) has 3.88% German WER on the leaderboard. It accepts at most 20 minutes per call, and timestamps need `Qwen3-ForcedAligner-0.6B` (Apache-2.0), which handles at most 5 minutes per call but has the lowest German timestamp shift in the report (46.5 ms against 122.7 ms for the NeMo Forced Aligner and 165.1 ms for WhisperX, Table 9). In bf16 the pair needs about 6.5 GB, more than the memory margin at the upper bound.

The gold questions and a German workshop recording decide whether either upgrade is worth it.

## Not verified

- No model was run on the target Mac. Speed and peak memory of `parakeet-mlx` and of pyannote on MPS need a measurement with a one-hour German recording.
- Accuracy on German conversational workshop audio. All German WER figures for Parakeet are on read speech. The test data must stay public or synthetic, so the evaluation needs a public German conversational recording or a recorded synthetic workshop.
- German and English switching within one recording. Parakeet v3 detects the language itself, but no benchmark covers code-switching.
- Timestamp accuracy of Parakeet's word times in German. Turn-level citations need seconds, not milliseconds, so this is low risk.
- Diarization quality on German workshops, and on rooms with one table microphone.
- Slide detection on shared-screen recordings.
- The scope of FluidAudio's `NOTICE.md` for its converted community-1 weights, if the FluidAudio path is used.

## Sources

- Benchmarks: [Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard) (multilingual, English and long-form tabs, version 1.16.1, 2026-11-18); [Open ASR Leaderboard paper, arXiv 2510.06961](https://arxiv.org/html/2510.06961v3); [secondary mirror of the German leaderboard](https://paperswithcode.co/benchmark/open-asr-leaderboard-german).
- Papers: [Canary-1B-v2 and Parakeet-TDT-0.6B-v3, arXiv 2509.14128](https://arxiv.org/html/2509.14128v2); [Qwen3-ASR technical report, arXiv 2601.21337](https://arxiv.org/html/2601.21337); [VibeVoice-ASR technical report, arXiv 2601.18184](https://arxiv.org/html/2601.18184v2); [Whisper, arXiv 2212.04356](https://arxiv.org/abs/2212.04356); [Careless Whisper, arXiv 2402.08021](https://arxiv.org/abs/2402.08021); [WhisperKit, arXiv 2507.10860](https://arxiv.org/abs/2507.10860).
- Model cards: [parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3), [mlx-community/parakeet-tdt-0.6b-v3](https://huggingface.co/mlx-community/parakeet-tdt-0.6b-v3), [fluid-speech-pro](https://huggingface.co/FluidVoice-Labs/fluid-speech-pro), [Qwen3-ASR-1.7B](https://huggingface.co/Qwen/Qwen3-ASR-1.7B), [whisper-large-v3-turbo](https://huggingface.co/openai/whisper-large-v3-turbo), [cohere-transcribe-03-2026](https://huggingface.co/CohereLabs/cohere-transcribe-03-2026), [VibeVoice-ASR](https://huggingface.co/microsoft/VibeVoice-ASR), [granite-speech-4.1-2b-plus](https://huggingface.co/ibm-granite/granite-speech-4.1-2b-plus), [thewhisper-large-v3-turbo](https://huggingface.co/TheStageAI/thewhisper-large-v3-turbo), [primeline whisper-large-v3-turbo-german](https://huggingface.co/primeline/whisper-large-v3-turbo-german), [speaker-diarization-community-1](https://huggingface.co/pyannote/speaker-diarization-community-1), [speaker-diarization-3.1](https://huggingface.co/pyannote/speaker-diarization-3.1), [diar_streaming_sortformer_4spk-v2](https://huggingface.co/nvidia/diar_streaming_sortformer_4spk-v2), [FluidInference/speaker-diarization-coreml](https://huggingface.co/FluidInference/speaker-diarization-coreml). Licenses, parameter counts and revisions from the Hugging Face model API on 2026-11-18.
- Tools: [parakeet-mlx](https://github.com/senstella/parakeet-mlx), [pyannote.audio](https://github.com/pyannote/pyannote-audio), [FluidAudio](https://github.com/FluidInference/FluidAudio) and its [benchmarks](https://github.com/FluidInference/FluidAudio/blob/main/Documentation/Benchmarks.md), [whisper.cpp](https://github.com/ggml-org/whisper.cpp), [WhisperX](https://github.com/m-bain/whisperX), [CTranslate2](https://github.com/OpenNMT/CTranslate2), [Whisper](https://github.com/openai/whisper), [PySceneDetect](https://github.com/Breakthrough/PySceneDetect), [Docling audio and video](https://docling-project.github.io/docling/usage/processing_audio_media), [Docling video pipeline](https://docling-project.github.io/docling/_generated/examples/video_pipeline/), [LiteLLM audio transcription](https://docs.litellm.ai/docs/audio_transcription). Repository licenses from the GitHub API on 2026-11-18.
- Licensing: [FFmpeg legal page](https://ffmpeg.org/legal.html), [Homebrew ffmpeg formula](https://formulae.brew.sh/formula/ffmpeg).
- Apple: [Argmax on Apple SpeechAnalyzer](https://www.argmaxinc.com/blog/apple-and-argmax).
- Project context: [ADR-0002](https://github.com/Niko4417/OPT-cc/blob/claude/mattpocock-skills-wayfinder-36692b/docs/adr/0002-prescribe-a-markdown-format-for-workshop-results.md), [ADR-0006](https://github.com/Niko4417/OPT-cc/blob/claude/mattpocock-skills-wayfinder-36692b/docs/adr/0006-define-project-artifacts-by-origin-and-take-audio-and-video.md), [memory fit research](https://github.com/Niko4417/OPT-cc/blob/research/memory-fit/docs/research/memory-fit.md).
