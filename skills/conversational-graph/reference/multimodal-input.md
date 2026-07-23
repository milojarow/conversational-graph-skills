# Multimodal input — accepting voice and images in a graph bot

The model driving the nodes is very often already `text+image+audio+video → text` while the engine's endpoint is purely textual (`{session, message}`). The capability is paid for and unused: enabling it is **endpoint + prompt, zero engine changes** — the graph, transitions and tools are untouched. Four traps show up on the way.

## 1. The history NEVER stores the base64

12 seconds of WAV audio is ~730 KB of base64. Putting that in the session record blows it up and poisons every following turn. Store a **marker** (`[voice note]`, `[image]` plus whatever text came with it) and treat the media as **one-shot for the current turn only**.

Corollary: anything that serializes the history — notably the transition classifier — must tolerate a non-string `content`.

## 2. The documented format list is NOT the real one — measure before you restrict

A whitelist built by copying the provider's documented audio formats (wav, mp3, m4a, ogg, aac, flac, aiff) rejected `webm` with an explicit 400 — and the frontend was told it would have to transcode to WAV in the browser. **False:** tested against the actual provider, `audio/webm;codecs=opus` (the Chrome MediaRecorder default) works perfectly, as do `audio/mp4` (Safari) and `audio/ogg` (Firefox). An incomplete doc turned into a whitelist manufactures useless work downstream: **input validation must come from a probe against the real provider, not from reading its table.**

- **Design corollary:** accept the **raw MIME** (`blob.type`) and normalize in the backend (`split(";")[0]`, strip `audio/`, map `mpeg→mp3`, `x-m4a→m4a`). The client sends what it recorded and nobody translates.
- **Size corollary:** the browser's compressed default is ~11× smaller than WAV (48 KB vs 551 KB for 12 s of speech). Forcing WAV "for compatibility" would have multiplied the payload by 11 for no gain.

## 3. Domain guardrails are REINFORCED, not inherited

A textual rule ("don't diagnose") does not by itself cover "the customer sends a photo of their injury and asks what's wrong with them". Add an **image-specific rule to BASE**: describe cautiously, never interpret medical studies or estimate severity, redirect to an in-person assessment. Test it with an image that **invites** the violation, not a neutral one.

The general form: for each new input modality, re-derive every domain guardrail — a rule written for text does not automatically bind the same behavior when the trigger arrives as media.

## 4. Order and caps in the content array

Put **text before media** in the content array (the usual provider recommendation), and **cap the size** at the endpoint (~3.5 MB image, ~6 MB audio as base64) — otherwise one big attachment turns a cheap turn into a slow, expensive one.

Measured cost/latency: a turn with a voice note ~6.1 s vs ~4 s for pure text; with an image ~5.9 s. Acceptable for a support chat.

## Testing it for real, without devices

Generate the voice note with a TTS model from the same provider and post it to the endpoint exactly as the client would. Gotchas seen doing this:

- audio output in a chat-completions API may require `stream: true` (it fails with a 400 "Audio output requires stream: true" in normal mode);
- it arrives as `delta.audio.data` chunks in base64 that you concatenate and decode;
- if it comes out as raw PCM, wrap it in a WAV container (`ffmpeg -f s16le -ar 24000 -ac 1`).

With that, the test is genuinely end-to-end: synthesized audio → endpoint → assert the bot answered the SPOKEN content.
