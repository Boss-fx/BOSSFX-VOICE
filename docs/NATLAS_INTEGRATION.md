# N-ATLaS Integration Evidence: BossFx Voice

**Problem Statement 02 (Voice-First Access)** requires that voice input uses the official N-ATLaS ASR service for the relevant language. This document shows exactly how BossFx Voice meets that requirement and how the integration can be verified.

## 1. Models used

| # | Model | Purpose | Loaded with |
|---|---|---|---|
| 1 | `NCAIR1/N-ATLaS` | Generates every answer | `AutoModelForCausalLM.from_pretrained` (4-bit NF4 on T4) |
| 2 | `NCAIR1/Yoruba-ASR` | Transcribes Yoruba voice notes | `transformers.pipeline("automatic-speech-recognition")` |
| 3 | `NCAIR1/Hausa-ASR` | Transcribes Hausa voice notes | same |
| 4 | `NCAIR1/Igbo-ASR` | Transcribes Igbo voice notes | same |
| 5 | `NCAIR1/NigerianAccentedEnglish` | Transcribes Nigerian-accented English voice notes | same |

No other language or speech model is used anywhere in the system.

## 2. Code path (notebook `BossFx_Voice_NATLaS.ipynb`)

**Cell 3: loading.** The model IDs are hard-coded:

```python
LLM_ID = "NCAIR1/N-ATLaS"
ASR_IDS = {
    "yo": "NCAIR1/Yoruba-ASR",
    "ha": "NCAIR1/Hausa-ASR",
    "ig": "NCAIR1/Igbo-ASR",
    "en": "NCAIR1/NigerianAccentedEnglish",
}
```

**Cell 4: voice → text.** Each voice note goes to the ASR model for the language the user selected:

```python
def transcribe(path, lang):
    audio, _ = librosa.load(to_wav16k(path), sr=16000)
    result = asr[lang]({"raw": audio, "sampling_rate": 16000})
    return result["text"].strip(), round(len(audio) / 16000, 1)
```

**Cell 4: text → answer.** The transcript is answered by N-ATLaS using its own chat template, as documented on the model card:

```python
text = tokenizer.apply_chat_template(
    messages, add_generation_prompt=True, tokenize=False,
    date_string=datetime.datetime.now().strftime("%d %b %Y"))
out = llm.generate(**inputs, max_new_tokens=350, do_sample=False,
                   repetition_penalty=1.12, use_cache=True)
```

**Cell 6: bot handler.** `on_voice()` calls `transcribe()` and then `ask_natlas()` for every Telegram voice note.

## 3. Per-request proof in the validation log

Every interaction record in `interactions.jsonl` / `validation_interactions.csv` stores the exact models used for that request. Example record (text abbreviated, timings illustrative):

```json
{
  "type": "interaction",
  "lang": "yo",
  "input_type": "voice",
  "transcript": "mo fẹ́ kọ́ fọ́rẹ̀ẹ̀sì ni, ...",
  "response": "Bẹẹni! Lati bẹrẹ iṣowo Forex, ...",
  "asr_model": "NCAIR1/Yoruba-ASR",
  "llm_model": "NCAIR1/N-ATLaS",
  "asr_seconds": 3.1,
  "total_seconds": 14.8
}
```

## 4. How a reviewer can verify

1. Open the live bot at [t.me/BossFxVoiceBot](https://t.me/BossFxVoiceBot), choose a language and send a voice note. The reply shows the transcript (🗣️) produced by the N-ATLaS ASR model, followed by the N-ATLaS answer (💬).
2. Run the notebook: cell 3 prints each model as it loads from Hugging Face (`Loaded ASR: NCAIR1/...`), and cell 5 shows N-ATLaS answering in all four languages.
3. Check `asr_model` / `llm_model` on any row of the validation CSV.

## 5. Why 4-bit quantisation is still N-ATLaS

To fit the 8B LLM and four ASR models on a single 16 GB T4, N-ATLaS is loaded with bitsandbytes 4-bit NF4 quantisation at load time. The weights come directly from `NCAIR1/N-ATLaS`. No other base model is involved, and nothing is distilled or swapped. On a larger GPU, `USE_4BIT = False` runs the original fp16 weights unchanged.

---

*N-ATLaS is an initiative of the Federal Ministry of Communications, Innovation and Digital Economy, and powered by Awarri Technologies.*
