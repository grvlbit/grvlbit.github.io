---
title: "Lokale Agentic AI mit GPUstack und opencode"
date: 2026-08-17 08:20:44 +0200
categories: [AI, DevTools]
tags: [gpustack, opencode, docker, agentic-ai, llm, local-ai]
image:
  path: assets/img/posts/2026-08-17/opencode-prompt.png
  alt: "opencode läuft lokal gegen GPUstack"
---

> **Disclaimer:** Dieser Artikel ist keine offizielle Anleitung der Informatikdienste. Vorausgesetzt werden grundlegende Kenntnisse im Bereich KI und idealerweise auch KI-Agenten. Ich empfehle, KI-Tools nur für Aufgaben einzusetzen, die man sich auch selbst zutrauen würde.
>
> *Only let the AI do what you would feel comfortable doing yourself!*
{: .prompt-info }

## Was ist opencode?
**[Opencode](https://opencode.ai/)** ist ein Open-Source-Coding-Agent. Er ist als terminalbasiertes Interface, Desktop-App oder IDE-Erweiterung verfügbar – ich verwende hier nur das Terminal-Interface. Die Idee ist simpel: Du gibst dem Agenten eine Aufgabe, er liest deinen Code, schlägt Änderungen vor, führt Befehle aus – und du schaust dabei zu (oder greifst ein, wenn er mal wieder etwas zu *kreativ* wird).

## Schrit 1: API Key

Um Opencode wirklich sinnvoll zu nutzen, brauchen wir einen API Key für einen
AI-Anbieter unserer Wahl. Da wir unseren Local-AI-POC testen wollen, [erstellen wir einen API Key für GPUstack](../generating-gpustack-token/).

## Schritt 2: opencode installieren – aber sicher
Opencode lässt sich auf [verschiedene Arten installieren](https://opencode.ai/docs/de#installation). Da ich kein Fan von `curl \| bash`-Pipes bin (mehr dazu in einem anderen Post) und npm bekanntlich ein beliebtes Ziel für Supply-Chain-Angriffe ist, empfehle ich den Docker-Weg. Wer Docker hat, kann direkt loslegen:
```
docker run -it --rm ghcr.io/anomalyco/opencode

```
![Opencode Startup](/assets/img/posts/2026-08-17/opencode-startup.png)
_Das Interface ist bereit – in dieser Konfiguration spricht opencode aber noch mit den opencode.ai-Servern im Internet. Das ändern wir gleich._

> Wichtig: So gestartet, verbindet sich opencode mit den opencode.ai-Modellen im Internet, analog zu ChatGPT oder jedem anderen kostenlosen KI-Tool. Wir wollen aber unsere lokale GPUstack-Infrastruktur nutzen.
{: .prompt-danger }


## Schritt 3: Eigenes Docker Image bauen
Wir wollen zwei Dinge erreichen:
1. opencode gegen unsere lokale GPUstack-Instanz konfigurieren.
2. Sicherstellen, dass der Agent nur im aktuellen Verzeichnis Änderungen vornehmen kann – kein Wildwuchs im Home-Verzeichnis.

Dafür bauen wir uns ein eigenes Image, grob angelehnt an die Anleitung von [klikukliku](https://klikukliku.dev/posts/opencode-in-docer-container/).
```
FROM alpine:3.19
ARG UID=1000
ARG GID=1000
ARG OPENCODE_VERSION=latest

RUN apk add --no-cache curl ca-certificates bash libstdc++ libgcc \
    && if [ "$OPENCODE_VERSION" = "latest" ]; then \
         curl -fsSL https://opencode.ai/install | bash; \
       else \
         curl -fsSL https://opencode.ai/install | bash -s -- --version "$OPENCODE_VERSION"; \
       fi \
    && mv /root/.opencode/bin/opencode /usr/local/bin/opencode \
    && apk del curl

RUN addgroup -g $GID coder 2>/dev/null; \
	GROUP_NAME=$(getent group $GID | cut -d: -f1); \
	adduser -D -s /bin/sh -u $UID -G "$GROUP_NAME" coder \
    && mkdir -p /home/coder/.config/opencode \
    && mkdir -p /home/coder/.local/share/opencode /home/coder/.cache/opencode /home/coder/.local/state/opencode \
    && chown -R coder:"$GROUP_NAME" /home/coder

USER coder
WORKDIR /workspace
CMD ["opencode"]

```
Lokal bauen:
```
docker build --build-arg UID=$(id -u) --build-arg GID=$(id -g) --no-cache -t opencode-container .

```
> Abkürzung: Wer sich das Bauen sparen möchte, kann mein fertiges Image verwenden: ghcr.io/grvlbit/opencode-container:latest. Im Rest des Artikels verwende ich dieses Image.
>

## Schritt 4: Konfiguration erstellen
### Konfigurationsdatei anlegen
```
mkdir -p ~/.config/opencode
vim ~/.config/opencode/opencode.json

```
Inhalt der Konfigurationsdatei:
```
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["unibe-ai"],
  "provider": {
    "unibe-ai": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Unibe Local AI PoC Provider",
      "options": {
        "baseURL": "https://gpustack.unibe.ch/v1",
        "apiKey": "{env:UNIBE_AI_API_KEY}"
      },
      "models": {
        "qwen3-coder-30b-a3b-instruct": { "name": "Qwen3 Coder 30B" },
        "gpt-oss-120b":                  { "name": "GPT OSS 120B" },
        "minimax-m2.7":                  { "name": "MiniMax M2.7" },
        "qwen3-vl-30b-a3b-instruct":     { "name": "Qwen3 VL 30B" },
        "qwen3-vl-8b-instruct":          { "name": "Qwen3 VL 8B" },
        "internvl3-8b-instruct":         { "name": "InternVL3 8B" }
      }
    }
  },
  "model": "unibe-ai/qwen3-coder-30b-a3b-instruct",
  "permission": {
    "edit": "ask",
    "bash": "ask"
  }
}

```
Zwei Punkte sind hier besonders wichtig:
- **`baseURL`**: zeigt auf unsere lokale GPUstack-Instanz – Daten bleiben intern.
- **`permission`**: Bei allen Dateiänderungen und Shell-Befehlen wird standardmässig nachgefragt. Wer schon mal einem allzu enthusiastischen Agenten beim Aufräumen eines Repos zugeschaut hat, weiss, warum das keine schlechte Idee ist.

### API Key sicher übergeben
In der Konfiguration haben wir eingerichtet, dass der API Key in der Umgebungsvariable `UNIBE_AI_API_KEY` zu finden ist. Wir nutzen das `--env-file`-Feature von Docker, um den Key sauber in den Container zu befördern – nicht als Argument, nicht hardcoded:
```
echo "UNIBE_AI_API_KEY=dein-api-key-hier" > ~/.opencode.env
chmod 600 ~/.opencode.env

```

### Persistente Daten, Sessions & Cache

Der Standard-Workflow erstellt einen Docker-Container, der zwischen den Starts des Containers keine Session- und State-Informationen von opencode speichert. Dies mag zunächst in Ordnung sein, aber irgendwann ist es nützlich, eine Sitzung zu beenden und fortzusetzen.
Glücklicherweise kann diese Funktion recht einfach mit *named* Volumes hinzugefügt werden.

Diese Volumes werden beim Start des Containers hinzugefügt:
````
  -v opencode-data:/home/coder/.local/share/opencode \
  -v opencode-state:/home/coder/.local/state/opencode \
  -v opencode-cache:/home/coder/.cache/opencode \
````

### AGENTS.md

README.md-Dateien sind für Menschen gedacht: Kurzanleitungen, Projektbeschreibungen und Richtlinien für Mitwirkende.
AGENTS.md kann zusätzlichen, manchmal detaillierten Kontext enthalten, den Coding-Agents benötigen: Build-Schritte, Tests und Konventionen, die eine README durcheinanderbringen oder für menschliche Mitwirkende sogar irrelevant sein können.

Wir können globale Regeln zu `~/.config/opencode/AGENTS.md` hinzufügen. Diese Regeln gelten für alle opencode-Sitzungen. Da sich diese Datei nicht im Git-Repository des Projekts befindet, ist dies nützlich für persönliche Regeln, wie z.B.:

``` 
Treat all data as private, only look up information without leaking of private/project data (try to obfuscate infos in your searches with placeholders), if unsure ask
```

Die AGENTS.md wird ebenfalls in den Container mittels Volume-Mount eingebunden:

```
-v "$HOME/.config/opencode/AGENTS.md:/home/coder/.config/opencode/AGENTS.md:ro" \
```

### Websearch mit SearXNG

So richtig gut wird Opencode mit der Fähigkeit das Internet zu durchsuchen. Da die
Einrichtung von SearXNG etwas umfangreicher ist, wird diese in einem separaten Beitrag
[Setup websearch mit SearXNG](../setup-websearch-mit-searxng/) beschrieben.

> Dieser Schritt ist lohnenswert, aber optional auch ohne Websearch ist opencode
> bereits mächtig.
{: .prompt-tip }

Sobald der SearXNG container läuft, ist der Grossteil geschafft.
Die Fähigkeit das Internet zu durchsuchen bringen wir Opencode als *Tool* bei.

Zunächst erstellen wir ein `tools` Verzeichnis im `.config` Ordner von opencode:

```
mkdir -p ~/.config/opencode/tools
```

Als nächsten legen wir dort ein Datei `searxng.ts` mit der Beschreibung für opencode wie searxng zu verwenden ist:

```typescript
import { tool } from "@opencode-ai/plugin";

const SEARXNG_URL = process.env.SEARXNG_URL ?? "http://host.docker.internal:8080";

export default tool({
  description:
    "Search the web for up-to-date information. Use this when you need current docs, news, or anything not in your training data.",
  args: {
    query: tool.schema.string().describe("the search query"),
  },
  execute: async ({ query }) => {
    const url = new URL("/search", SEARXNG_URL);
    url.searchParams.set("q", query);
    url.searchParams.set("format", "json");

    const res = await fetch(url.toString());
    if (!res.ok) throw new Error(`SearxNG error: ${res.status}`);

    const data = await res.json();
    const results = (data.results ?? []).slice(0, 8).map((r: any) => ({
      title: r.title,
      url: r.url,
      snippet: r.content ?? "",
    }));

    return JSON.stringify(results, null, 2);
  },
});
```

Unser tool in der globalen Konfiguration von opencode mounten wir wieder in den Container:

```
  -v "$HOME/.config/opencode/tools/:/home/coder/.config/opencode/tools/:ro" \
```


### Vollständiger `docker run` Befehl

Aus den einzelnen Schritten ergibt sich der vollständige `docker run`-Befehl:
```
docker run --rm -it \
  --env-file "$HOME/.opencode.env" \
  -v "$HOME/.config/opencode/tools/:/home/coder/.config/opencode/tools/:ro" \
  -v "$HOME/.config/opencode/opencode.json:/home/coder/.config/opencode/opencode.json:ro" \
  -v "$HOME/.config/opencode/AGENTS.md:/home/coder/.config/opencode/AGENTS.md:ro" \
  -v opencode-data:/home/coder/.local/share/opencode \
  -v opencode-state:/home/coder/.local/state/opencode \
  -v opencode-cache:/home/coder/.cache/opencode \
  -v "$(pwd):/workspace:rw" \
  ghcr.io/grvlbit/opencode-container:latest opencode

```

![opencode mit GPustack](assets/img/posts/2026-08-17/opencode-prompt.png)
_Tadaa! opencode läuft nun vollständig lokal gegen GPUstack._

## Schritt 5: Alias anlegen
Weil niemand diesen Befehl jedes Mal ausschreiben möchte, wandert er als Alias in die `~/.bashrc` oder `~/.zshrc`:

```
alias occ='docker run --rm -it \
  --env-file "$HOME/.opencode.env" \
  -v "$HOME/.config/opencode/tools/:/home/coder/.config/opencode/tools/:ro" \
  -v "$HOME/.config/opencode/opencode.json:/home/coder/.config/opencode/opencode.json:ro" \
  -v "$HOME/.config/opencode/AGENTS.md:/home/coder/.config/opencode/AGENTS.md:ro" \
  -v opencode-data:/home/coder/.local/share/opencode \
  -v opencode-state:/home/coder/.local/state/opencode \
  -v opencode-cache:/home/coder/.cache/opencode \
  -v "$(pwd):/workspace:rw" \
  ghcr.io/grvlbit/opencode-container:latest opencode --port 4096 --hostname 0.0.0.0'

```
Ab sofort reicht ein `occ` im Terminal zum starten unseres Containers – vermutlich der wichtigste Schritt des ganzen Artikels.
Ergänzend kann die vorherige session mit `occ -c` wiederhergestellt oder eine spezifische Session mit `occ -s <session-id>` verbunden werden.

Für alle weiteren Details zur Bedienung empfehle ich die [offizielle Dokumentation](https://opencode.ai/docs/de).
