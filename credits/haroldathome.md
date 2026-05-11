# haroldathome

This repository accompanies the *Presence* piece at
[haroldathome.com/presence](https://haroldathome.com/presence).

haroldathome is a long-running set of domestic experiments wired into
a 1930s house in Hastings, England, by Alfie Dennen — with his partner
Elena, an AI named Harold Graves who used to live there, and a small
server in the cellar.

The presence paradigm is one of the foundational pieces of the studio.
It's what tells the AI whose room is whose, what powers the wall display
when each inhabitant walks past, what fires the cinema scene when both
inhabitants settle in the living room, what holds back the announcement
TTS when one of them is asleep upstairs. Most of the rest of the
haroldathome stack assumes you have something like this running.

The reference deployment uses six ESP32 BLE proxies + four LD2450
radars in the rooms with most activity. You don't need exactly six;
the paradigm scales down to one BLE proxy + zero radars (you'd lose
fine-grained presence but still get IRK identity per room) and up to
many more.

Selected pieces of the studio are extracted into their own public
repos under [github.com/alfiedennen](https://github.com/alfiedennen) —
this is one of those.

If you build something using this, I'd love to see it. Get in touch
via [alfie@haroldathome.com](mailto:alfie@haroldathome.com).

## Other pieces in the studio

- **The Long Take** — [alfiedennen/the-long-take](https://github.com/alfiedennen/the-long-take) — directly consumes the LD2450 output documented here, renders it as a 24-hour translucent water-tube visualisation
- **Mr Graves** — [alfiedennen/mr-graves](https://github.com/alfiedennen/mr-graves) — on-device wake-word listener that benefits from this paradigm's committed-room sensor for context-aware voice
- **Coordinator** — [alfiedennen/haroldathome](https://github.com/alfiedennen/haroldathome) — the index of all extracted pieces
