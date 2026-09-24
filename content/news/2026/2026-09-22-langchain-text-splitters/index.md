---
title: 'LangChain Text Splitters, the Missing Link when Processing Long Texts with LLMs'
date: '2026-09-22'
tease: "Circumvent context window limits of LLMs by splitting long texts into manageable chunks."
hide_tease: false
subsites: [global, eu, us, freiburg]
tags: [tools, ai, humanities, llm]
contributions:
  authorship:
    - IvoLeist
    - arash77
    - Sch-Da
  funding:
    - ai4social
---

Imagine you have a very long text that you want to process with a large language model (LLM). You might want to summarize it, extract information, or translate it into another language. However, LLMs have limits on how much text they can process in a single request in a so-called context window. If your text exceeds those limits, you need a way to split it into smaller pieces (chunks) that the model can handle.

## Divide and Conquer:<br>LangChain Text Splitters to the Rescue

LangChain is a popular open-source framework for building LLM-powered applications. It provides a set of utilities
wrapped in standalone Python packages. One of these is [LangChain Text Splitters](https://github.com/langchain-ai/langchain/tree/master/libs/text-splitters), which provides different strategies for chunking as shown below:

<div id="split-modes-visual">
  <style>
    #split-modes-visual {
      --subtle: var(--muted-foreground, #59636e);
      --chunk: var(--viz-series-1, #2675c9);
    }
    #split-modes-visual .source {
      margin-bottom: 16px;
    }
    #split-modes-visual .controls {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-bottom: 20px;
      cursor: pointer;
    }
    #split-modes-visual .modes {
      display: grid;
      grid-template-columns: repeat(3, minmax(0, 1fr));
      gap: 24px;
    }
    #split-modes-visual .mode {
      min-width: 0;
    }
    #split-modes-visual h3 {
      margin: 0 0 8px;
    }
    #split-modes-visual .settings {
      margin-bottom: 12px;
    }
    #split-modes-visual .chunks {
      display: flex;
      flex-direction: column;
      gap: 8px;
      margin-bottom: 16px;
    }
    #split-modes-visual .chunk {
      padding: 10px 12px;
      background: color-mix(
        in srgb, var(--chunk) 14%, transparent
      );
      border-left: 3px solid var(--chunk);
      overflow-wrap: anywhere;
      white-space: pre-wrap;
    }
    #split-modes-visual .text-small {
      font-size: 0.85em;
    }
    #split-modes-visual .text-muted {
      color: var(--subtle);
    }
    @media (max-width: 580px) {
      #split-modes-visual .modes {
        grid-template-columns: 1fr;
      }
    }
  </style>

  <div class="source">
    <span class="text-small text-muted">Example:</span>
    <div>Dr. Smith studies photosynthesis in freshwater algae. She takes notes. She checks them. She shares her findings.</div>
  </div>

  <label class="controls">
    <input type="checkbox" data-overlap>
    With chunk overlap
  </label>

  <div class="modes">
    <section class="mode">
      <h3>Character (.)</h3>
      <div class="settings text-small text-muted"
           data-settings="character"></div>
      <div class="chunks" data-chunks="character"></div>
    </section>
    <section class="mode">
      <h3>Token (count=5)</h3>
      <div class="settings text-small text-muted"
           data-settings="token"></div>
      <div class="chunks" data-chunks="token"></div>
    </section>
    <section class="mode">
      <h3>Sentence</h3>
      <div class="settings text-small text-muted"
           data-settings="sentence"></div>
      <div class="chunks" data-chunks="sentence"></div>
    </section>
  </div>
</div>

<script>
(() => {
  const root = document.getElementById("split-modes-visual");
  if (!root) return;

  const toggle = root.querySelector("[data-overlap]");

  const firstSentence =
    "Dr. Smith studies photosynthesis in freshwater algae.";
  const shortSentences =
    "She takes notes. She checks them.";
  const lastSentence = "She shares her findings.";
  const repeatedSentence = "She checks them.";

  // Each chunk records its text and the length of the
  // prefix repeated from the preceding chunk.
  const chunk = (text, repeatedLength = 0) => ({
    text,
    repeatedLength
  });

  function textChunks(mode, overlap) {
    const chunks = mode === "character"
      ? [
          chunk("Dr."),
          chunk(
            "Smith studies photosynthesis in freshwater algae."
          )
        ]
      : [chunk(firstSentence)];

    chunks.push(chunk(shortSentences));

    chunks.push(
      overlap
        ? chunk(
            repeatedSentence + " " + lastSentence,
            repeatedSentence.length
          )
        : chunk(lastSentence)
    );

    return chunks;
  }

  // Verified cl100k_base token pieces for this exact source.
  // This is a fixed demonstration, not a browser tokenizer.
  const tokens = [
    "Dr", ".", " Smith", " studies", " photos",
    "ynthesis", " in", " freshwater", " algae", ".",
    " She", " takes", " notes", ".", " She",
    " checks", " them", ".", " She", " shares",
    " her", " findings", "."
  ];

  function tokenChunks(overlapEnabled) {
    const size = 5;
    const overlap = overlapEnabled ? 2 : 0;
    const stride = size - overlap;
    const chunks = [];

    for (let start = 0; start < tokens.length; start += stride) {
      const end = Math.min(start + size, tokens.length);
      const text = tokens.slice(start, end).join("");

      const repeatedLength = start === 0
        ? 0
        : tokens.slice(start, start + overlap).join("").length;

      chunks.push(chunk(text, repeatedLength));

      if (end === tokens.length) break;
    }

    return chunks;
  }

  function renderChunks(mode, chunks) {
    const container = root.querySelector(
      `[data-chunks="${mode}"]`
    );
    container.replaceChildren();

    chunks.forEach(({ text, repeatedLength }, index) => {
      const element = document.createElement("div");
      element.className = "chunk";
      element.setAttribute("aria-label", `Chunk ${index + 1}`);

      if (repeatedLength > 0) {
        const strong = document.createElement("strong");
        strong.textContent = text.slice(0, repeatedLength);
        element.append(strong);
      }

      element.append(
        document.createTextNode(text.slice(repeatedLength))
      );

      container.append(element);
    });
  }

  function update() {
    const overlap = toggle.checked;

    renderChunks(
      "character",
      textChunks("character", overlap)
    );
    renderChunks(
      "sentence",
      textChunks("sentence", overlap)
    );
    renderChunks("token", tokenChunks(overlap));

    for (const mode of ["character", "sentence"]) {
      root.querySelector(`[data-settings="${mode}"]`)
        .textContent =
          `Target: 50 characters · Overlap: ${
            overlap ? 20 : 0
          } characters`;
    }

    root.querySelector('[data-settings="token"]')
      .textContent =
        `cl100k_base · Overlap: ${overlap ? 2 : 0} tokens`;
  }

  toggle.addEventListener("change", update);
  update();
})();
</script>

As you can see, there is no splitting strategy which is universally better than the others, but you have
to choose the one that fits your text and downstream application the best. Also note the toggle for chunk
overlap which can be useful for tasks such as translation, summarization, or retrieval-augmented generation (RAG).

## LangChain Text Splitters in Galaxy

Insert GIF

## A Galaxy Workflow Example:<br>Transcribing & Translating long Video Transcripts

Like our users, we enjoy exploring the capabilities of various open-source LLMs available through the
[LLM Hub](https://usegalaxy.eu/?tool_id=llm_hub) ([Blogpost](https://galaxyproject.org/news/2025-10-10-llm-hub)).
While experimenting with the LLM Hub as an alternative to the [ChatGPT Galaxy tool](https://usegalaxy.eu/?tool_id=chatgpt_openai_api) in our [transcription & translation Galaxy workflow](https://usegalaxy.eu/published/workflow?id=a2284469005518e1), we faced a practical limitation: Translating long video transcripts results in truncated outputs or even timeouts!

Thanks to the [LangChain Text Splitters Galaxy tool](https://usegalaxy.eu/?tool_id=langchain_text_splitters) this now a problem of the past. See below how we feed manageable chunks to the LLM, which are then translated and concatenated into a single document:

<iframe title="Galaxy Workflow Embed" style="width: 100%; height: 400px; border: none;" src="https://usegalaxy.eu/published/workflow?id=0b5ab1b58eee9228&embed=true&buttons=true&about=false&heading=false&minimap=false&zoom_controls=false&initialX=-20&initialY=-20&zoom=1"></iframe>

#| Step                    |  Description |
-|-------------------------|--------------|
1| Speech to text |  [WhisperX](https://usegalaxy.eu/?tool_id=whisperx) transcribes the audio or video into subtitles (SRT): numbered cues with timestamps, so the translation can be used as subtitles again.  |
2| Split into chunks |  SRT is cut into chunks at paragraph boundaries using the recursive Text Splitter from [LangChain Text Splitters](https://usegalaxy.eu/?tool_id=langchain_text_splitters).
3| One LLM request per chunk |Every chunk goes to [LLM Hub](https://usegalaxy.eu/?tool_id=llm_hub) on its own, with the same prompt.
4| Join the answers | [awk](https://usegalaxy.eu/?tool_id=awk) ends each answer with exactly one blank line, so subtitle blocks don't stick together. [Concatenate](https://usegalaxy.eu/?tool_id=tp_cat) joins all answers in the original order into the final file.
