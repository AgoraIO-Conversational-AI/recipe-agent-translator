# Deep Dive — Translation Config & Vendor Chain

> **When to Read This:** You are changing the language pair (`SOURCE_LANG`, `TARGET_LANG`, `TTS_VOICE`), the translation system prompt, the OpenAI model or temperature, the Deepgram STT model, the MiniMax TTS voice, the VAD config, or any session parameter around the cascading vendor chain. For the high-level picture, start at [02_architecture](../02_architecture.md).

This recipe replaces a single MLLM with a **cascading vendor chain**: Deepgram STT transcribes the speaker, a managed OpenAI LLM translates the transcript, and MiniMax TTS speaks the result. All vendors are Agora-managed; no API keys are required by default.

## The vendor chain (built in `Agent.start()`)

```python
llm = OpenAI(
    api_key=self.openai_api_key,           # None = Agora-managed (zero-key)
    model=self.openai_model,               # OPENAI_MODEL, default gpt-4o-mini
    system_messages=build_translation_system_messages(self.target_lang),
    greeting_message=self.greeting,
    temperature=0.3,
)
stt = DeepgramSTT(model="nova-3", language=self.source_lang)   # SOURCE_LANG
tts = MiniMaxTTS(model="speech_2_6_turbo", voice_id=self.tts_voice)  # TTS_VOICE

agora_agent = AgoraAgent(
    client=self.client,
    greeting=self.greeting,
    failure_message="Please wait a moment.",
    max_history=50,
    turn_detection={...},           # see VAD section below
    advanced_features={"enable_rtm": True},
    parameters=parameters,
).with_stt(stt).with_llm(llm).with_tts(tts)
```

## Translation system prompt (`translation_config.py`)

`build_translation_system_messages(target_lang)` returns a single system message:

```
You are a translation assistant. Translate the user's message into {TARGET_LANG}.
Output only the translation, with no extra commentary, quotation marks, or explanations.
```

This function is a pure builder (no `agora_agent` import) and is covered by `tests/test_translation_config.py`. Edit here to change the translation instruction; the `target_lang` argument comes from the `TARGET_LANG` env var.

## Language pair coupling

| Env var       | Maps to               | Example values                      |
| ------------- | --------------------- | ----------------------------------- |
| `SOURCE_LANG` | `DeepgramSTT.language`| `es`, `fr`, `zh`, `ja`              |
| `TARGET_LANG` | Translation prompt    | `English`, `French`, `Japanese`     |
| `TTS_VOICE`   | `MiniMaxTTS.voice_id` | `English_captivating_female1`, `French_expressivefemale` |

All three must be consistent. An inconsistent set (e.g. `TARGET_LANG=French` with an English voice) produces output in the right language but with an incorrect voice.

## VAD config (set on `AgoraAgent`)

```python
turn_detection={
    "config": {
        "speech_threshold": 0.5,
        "start_of_speech": {
            "mode": "vad",
            "vad_config": {
                "interrupt_duration_ms": 160,
                "prefix_padding_ms": 300,
            },
        },
        "end_of_speech": {
            "mode": "vad",
            "vad_config": {
                "silence_duration_ms": 480,
            },
        },
    },
}
```

VAD is set **on `AgoraAgent(...)`**, not on the `OpenAI` vendor. This is the **opposite** of the realtime recipe. Do not attempt to set `turn_detection` on the `OpenAI(...)` constructor.

## Session `parameters`

Set in `Agent.start()` and passed to `AgoraAgent`:

| Key                    | Value     | Why                                              |
| ---------------------- | --------- | ------------------------------------------------ |
| `audio_scenario`       | `chorus`  | Ultra-low-latency profile for web clients.       |
| `data_channel`         | `rtm`     | Transcript + metrics delivered over RTM.         |
| `enable_error_message` | `true`    | Surface agent-side errors to the client.         |
| `enable_metrics`       | `true`    | Emit pipeline metrics to the UI.                 |
| `output_audio_codec`   | optional  | Forwarded from `POST /startAgent` `parameters`.  |

## Keyless default

`OPENAI_API_KEY` defaults to `None` (unset). Passing `api_key=None` to `OpenAI(...)` signals Agora-managed: Agora's cloud uses its own OpenAI account. Set `OPENAI_API_KEY` only if your Agora account requires a BYO key.

## Related L1

- [02_architecture](../02_architecture.md) · [06_interfaces](../06_interfaces.md) · [07_gotchas](../07_gotchas.md)
