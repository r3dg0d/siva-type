# SIVA-Type — AI-Enhanced Voice Typing

A **fully-local AI dictation rewriter** for [niri](https://github.com/YaLTeR/niri) on Wayland,
built on the same stack as [SIVA](https://github.com/vincentisvalid/siva).

Press **F10**, speak a rough draft, and SIVA-Type transcribes it, rewrites it in your chosen
personality — **professional**, **nerdy**, or **intelligent** — with the model's chain of
thought streaming live to an overlay, then types the polished result straight into whatever
text field you had focused. No cloud, no API keys.

## How it works

```
   F10 (niri bind)
        │
        ▼
  bin/siva-type ─────────► /tmp/siva-type.sock (newline-JSON)
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
           bin/siva-type-daemon      quickshell overlay
           (orchestrator)            (SivaType.qml: status,
                    │                 style chips, live CoT,
        ┌───────┬───┴────┬───────┐    rewritten text preview)
        ▼       ▼        ▼       ▼
    pw-record whisper  llama-  wtype
    (Scarlett  -cli    server  (types result into
     2i2 mic)  (STT)   :8090    the focused input)
                       gemma-4-31B
```

1. **First F10** — overlay opens, desktop audio ducks 50%, the mic starts recording
   (prefers a Focusrite Scarlett hardware source, falls back to the default).
2. **Second F10** — recording stops, whisper transcribes, and the LLM rewrites the
   dictation in the selected style. Its brief reasoning streams into the overlay's
   chain-of-thought panel; the final text streams into the preview box.
3. **Typing** — the rewritten text is typed into the focused input via `wtype`,
   and the overlay auto-hides a few seconds later. F10 while busy cancels.

Click a personality chip on the overlay to switch styles (sticky until changed):

| Style | Voice |
|---|---|
| `professional` | Polished business writing — courteous, clear, confident |
| `nerdy` | Playful geeky energy with tasteful tech / sci-fi flavor |
| `intelligent` | Articulate and erudite — precise vocabulary, elegant structure |

### Design note: keyboard focus

The overlay is a layer-shell surface that takes **no keyboard focus, ever**
(`WlrKeyboardFocus.None`). That's the core trick: the text field you were typing in stays
focused for the entire dictation, so `wtype`'s virtual keyboard lands the rewritten text
exactly where your cursor was. Style chips are mouse-only for the same reason.

## Requirements

- niri + [quickshell](https://quickshell.org/) (≥ 0.3.0)
- llama.cpp (`llama-server`) running an instruct model on port 8090
  (shared with SIVA's `siva-llm`: gemma-4-31B-it)
- whisper.cpp (`whisper-cli`) + a ggml model (`ggml-base.en.bin`)
- wtype, socat, PipeWire (`pw-record`, `wpctl`)

## Install

```bash
# 1. scripts
cp bin/* ~/.local/bin/ && chmod +x ~/.local/bin/siva-type*

# 2. overlay widget — register in your quickshell shell.qml
cp quickshell/SivaType.qml ~/.config/quickshell/
#    ...then add `SivaType {}` inside your ShellRoot

# 3. niri config.kdl
#    spawn-at-startup "/home/YOU/.local/bin/siva-type-daemon"
#    binds { F10 repeat=false { spawn "/home/YOU/.local/bin/siva-type"; } }
```

If you don't run SIVA, point `WHISPER_MODEL` in `bin/siva-type-daemon` at your ggml model
and start a `llama-server` on port 8090.

## Protocol

The daemon owns `/tmp/siva-type.sock` and speaks newline-delimited JSON. UI-bound events:
`status`, `style`, `mic`, `transcript`, `cot` (reasoning delta), `reply` (rewritten-text
delta), `error`, `ping` (liveness heartbeat). Daemon-bound: `toggle`, `set_style`,
`cycle_style`, `user_text` (skip STT, rewrite this text), `cancel`, `hide`. Any client can
drive it:

```bash
echo '{"type":"set_style","style":"nerdy"}' | socat - UNIX-CONNECT:/tmp/siva-type.sock
```

## License

MIT
