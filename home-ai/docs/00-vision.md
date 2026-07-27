# 00 — Vision

## North star

> An email client that makes you feel like you have a brilliant chief of staff
> who has already read your entire inbox — running on hardware you own, with
> data that never leaves the house.

Email is the highest-signal, highest-friction stream most people manage. The
friction is not reading individual messages; it is the *aggregate* cognitive
load: deciding what matters, remembering context across threads, and composing
replies. A local AI assistant that carries that load — without the privacy cost
of a cloud service reading your mail — is the product.

## The three bets

1. **Local inference is good enough now.** A home AI inference machine can run
   models capable of high-quality summarization, classification, and drafting.
   We do not need frontier cloud models to deliver a superhuman experience for
   *mail*.

2. **Privacy is a feature, not a constraint.** Keeping mail on-device is not a
   compromise we tolerate; it is a primary reason the product exists and a
   durable differentiator against cloud AI-mail products.

3. **Speed is the experience.** "Superhuman" is as much about latency and
   keyboard flow as intelligence. The assistant should pre-compute so the user
   never waits on a model.

## What it feels like

- You open the app. The inbox is already sorted into **Now / Later / FYI /
  Noise**, each with a one-line summary.
- A thread you were CC'd on has a "Sam approved the Q3 budget; no action needed
  from you" note at the top.
- You hit `r`, and a draft in your voice is waiting; you edit two words and
  send.
- You type "what did legal say about the vendor contract?" and get the exact
  paragraph, linked to its thread.
- None of this ever touched a server you don't own.

## Explicit non-goals (vision level)

- Not a webmail *provider* — it connects to existing mailboxes (IMAP/JMAP/Gmail
  API), it does not host mail.
- Not a team collaboration tool (at least not initially) — single-user, personal.
- Not a cloud SaaS — the deployment target is one person's home machine.

## Success, stated simply

The author stops using their previous mail client because this one is better,
and recommends it to a privacy-conscious friend who also has capable home
hardware.
