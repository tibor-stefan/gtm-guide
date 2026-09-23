# GTM Guide, a production voice agent for hiring

**[gtmguide.ai](https://gtmguide.ai)** · live · built and operated by one person

GTM Guide interviews candidates for go-to-market roles with a **conversational voice agent**
instead of a form. The candidate talks, the agent listens, asks the next question from a
generated plan, and the answers are written into a structured profile while the conversation
is still running on screen.

> This repository is documentation only. The product code is private.

## Why this exists

Hiring for sales and marketing roles still runs on CVs that say very little and screening
calls that recruiters do not have time for. The signal is in *how* someone talks about their
pipeline, which is exactly what a text form destroys. So the intake is a conversation.

That made voice the product rather than a feature, and it set the bar: if the agent sounds
like a phone tree, the candidate leaves.

I spend my working life in talent acquisition software, so I had the unfair advantage of
knowing which part of the process was broken. What I did not have was any experience shipping
realtime voice. That part I learned the expensive way, below.

## The stack that shipped

**ElevenLabs Conversational AI** runs the interview. One agent, one connection: listening,
turn detection, response generation and speech all happen inside it. The browser holds a
signed session URL, and the backend builds the agent's prompt at runtime from the question
plan generated for that specific role.

Two things turned out to matter more than the choice of model.

**One voice for the whole journey.** The guided screens before the interview are narrated, and
the interview itself is spoken. Originally those came from two different providers, and it
broke the illusion instantly. For the candidate this is *one* conversation, so it has to be
one voice. Narration moved onto the same provider and the same voice as the agent.

**Read the voice, do not hardcode it.** The voice identifier is fetched from the agent
configuration at startup rather than sitting in an environment file. Two sources of truth for
"which voice am I" is a bug waiting for the next edit someone makes in the dashboard.

Measured after the switch: **first audible audio in 0.12 to 0.24 seconds**, down from 0.6 to
0.9 on the self-built stack it replaced. Narration renders complete in both German and English.

## What I learned by getting it wrong first

Before ElevenLabs there was a self-built stack on Gemini Live. It failed in ways that were
genuinely hard to see, and those failures are the most useful thing I can show.

**Silence is not silence.** Live would speak for a second and a half, then keep streaming
near-silent audio for minutes instead of closing the stream. Every downstream guard broke at
once. The idle watchdog never fired, because chunks were still arriving. The audio budget
counted padding as speech. And the client, which revealed text along the audio timeline, kept
typing for minutes with nothing audible. The reported symptom was "text runs, no sound." The
fix was trivial once seen: speech and padding sit two orders of magnitude apart in level, so
classify each chunk, end the utterance once the padding runs long, and never cache a render
that contained too little actual speech.

**The model talks past the script.** If a narration line ended in a question mark, the model
answered its own question, producing roughly five times the audio the sentence called for, and
stopped emitting its output transcript while doing it, so no transcript-based guard could
catch it. That needed a hard audio ceiling derived from the character count.

**`requestAnimationFrame` stops in background tabs. Web Audio does not.** "First audible
sample" was detected only from rAF, so a hidden tab never flipped to *started*, a watchdog
declared playback blocked, and it killed audio that was playing perfectly well. Never hang
audio state on rAF alone.

**Cut PCM on sample boundaries.** A chunk that ends mid-sample sounds like noise in the
browser. Carry the remainder into the next one.

**The cache key has to contain the provider.** Otherwise a render made in one voice gets
served under another one's name, silently, with a 200 and the correct text.

None of these are model problems. They are integration problems, and they are why I have a
strong opinion about a provider that owns the whole turn rather than handing me five seams to
get wrong.

## The head to head

In September I added **OpenAI Realtime** as a second engine running the same question plan,
selectable per session, through the full candidate journey including narration. Not a demo
switch: both paths go the whole way.

Findings worth keeping:

- `gpt-live-1` is listed in `/v1/models` and mints a realtime session without complaint. Only
  `/v1/realtime/calls` reveals it is not supported in realtime mode. The offer check runs
  *before* the model check, so a malformed SDP masks the real error entirely.
- Instructions and voice have to be set when the session is minted. Without them the model
  interviews with its own persona instead of the question plan.
- `response.instructions` *replaces* the session instructions for that response. The opening
  line came out without the question plan, and the model then restarted at question one.
- The realtime path emits `conversation.item.input_audio_transcription.completed` but no
  `.delta`. Anything built on deltas never sees the candidate's side at all.
- Screen answers only reach the model through explicit context updates, the direct equivalent
  of ElevenLabs' `sendContextualUpdate`.
- Same nominal voice, two models, two noticeably different accents. A TTS sample tells you
  nothing about how a realtime model will render that same voice. Only a realtime recording does.

The honest summary: a raw realtime API is cheaper per minute and gives you every knob. The
conversational agent gives you the turn. For a candidate interview, where an awkward pause
costs you the candidate, owning the turn was worth more than owning the knobs.

## Beyond voice

The same platform runs a CV enhancer, and I built a blind benchmark to prove it was actually
better rather than assuming it: anonymised outputs, randomised ordering, several independent
judges per case, and hallucinated or misattributed figures counted explicitly rather than
eyeballed.

It beat a leading commercial CV tool across every case in the final round, with no invented
figures on our side and a great many on theirs.

I mention it because it is the same discipline as the voice work. The interesting question is
never "does it run", it is "is it measurably better, and by whose judgement".

## Operating facts

- React and Vite on the front, Fastify and Postgres behind it, systemd units, release
  directories with symlink rollback
- German and English throughout, voice included
- Voice is testable without a microphone: a mock seam replaces the hardware only, while
  session, speech and turns run against the real backend
- Automated test coverage across both frontend and backend

*Built by [Tibor Stefan](https://github.com/tibor-stefan). Code private, happy to walk through
any of it in a conversation.*
