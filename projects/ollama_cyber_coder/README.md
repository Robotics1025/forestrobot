# Local Cyber Coder

This profile uses the locally installed `qwen2.5-coder:3b` model and is tuned
for coding, secure-code review, defensive security, authorized labs, and CTFs.

## Create or update the profile

```bash
ollama create cyber-coder -f Modelfile
```

## Run it

Interactive:

```bash
ollama run cyber-coder
```

Single prompt:

```bash
ollama run cyber-coder "Review this Python function for security bugs: ..."
```

## Hardware note

This computer has 8 GB RAM and no NVIDIA GPU. Close memory-heavy applications
before running the model. The first response is slower while the model loads;
later prompts in the same session are faster.

## License note

The `qwen2.5-coder:3b` model displays the Qwen Research License in Ollama.
Review that license before distributing or commercializing a product based on
the model.
