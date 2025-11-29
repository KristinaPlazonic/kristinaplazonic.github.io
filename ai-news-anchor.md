---
title: "I Built a Personal AI News Anchor in 90 Minutes (by Arguing with an LLM)"
date: 2024-11-29
---

# I Built a Personal AI News Anchor in 90 Minutes (by Arguing with an LLM)

I have 11 years of experience in IT, but keeping up with the "firehose" of AI news is becoming a full-time job. I have exactly four 15-minute walking intervals a day to catch up. I didn't need another news app; I needed a signal filter.

Instead of searching for a pre-made tool, I decided to pair-program a custom solution with an AI. My goal wasn't just to write code, but to treat the LLM as a Junior Architect—debating trade-offs, refining requirements, and building a "News-to-Speech" pipeline.

## 🎯 The Objective
Convert high-signal AI news (Hacker News) into a 15-minute audio briefing for daily walks, filtering out fluff and strictly preserving technical depth. This system is designed to be "pull-on-demand" via a mobile bookmark.

## 🏗️ The Architecture

### The Stack
* **Orchestrator:** n8n (Self-hosted)
* **Intelligence:** OpenAI (GPT-4o & TTS-1)
* **Trigger:** Webhook (Browser Bookmark)

### Logic Flow
1.  **Trigger:** I click a bookmark on my phone 📱 → Hits n8n Webhook.
2.  **Ingest:** HTTP Request pulls **Hacker News RSS**.
3.  **Smart Filter (The Gatekeeper):** An LLM evaluates titles and outputs structured JSON (e.g., `{"relevant": true}`).
4.  **Extraction:** **HTML-to-Markdown** converts the article, stripping ads but keeping code blocks.
5.  **Fork (Parallel Execution):**
    * **Branch A (Audio):** An LLM writes a script describing the code logic, which OpenAI TTS converts to MP3. This streams directly to my phone.
    * **Branch B (Archive):** The raw Markdown (with code blocks) is saved to my GitHub repo for deep diving later.

## 🧠 Architectural Decision Log

| Challenge | The "Lazy" Way | Our "Engineered" Choice | Reason |
| :--- | :--- | :--- | :--- |
| **Filtering** | Keyword Search | **LLM Classifier** | Keywords miss context. The LLM catches "neural networks" even if "AI" isn't in the title. |
| **Cleaning** | Complex Scraping | **HTML-to-Markdown** | The "batteries included" MVP solution. Removes noise cheaply without hallucination risks. |
| **Delivery** | Mobile App | **Browser Stream** | Zero friction. No app to build; just a direct binary stream to the browser's media player. |

## Conclusion
This project wasn't just about saving time; it was about defining a workflow that fits *my* life. By treating the AI as a partner rather than just a code generator, I built a robust tool that keeps me informed without the burnout.

