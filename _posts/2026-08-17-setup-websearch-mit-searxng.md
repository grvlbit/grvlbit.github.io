---
title: "Setup von Websearch mit SearXNG"
date: 2026-08-17 08:20:44 +0200
categories: [AI, DevTools]
tags: [searxng, opencode, docker, agentic-ai, llm, local-ai]
image:
  path: assets/img/posts/2026-08-17/searxng-prompt.png
  alt: "searxng läuft lokal im Docker Container"
---

> **Disclaimer:** Dieser Artikel ist keine offizielle Anleitung der Informatikdienste. Vorausgesetzt werden grundlegende Kenntnisse im Bereich KI und idealerweise auch KI-Agenten. Ich empfehle, KI-Tools nur für Aufgaben einzusetzen, die man sich auch selbst zutrauen würde.
>
> *Only let the AI do what you would feel comfortable doing yourself!*
{: .prompt-info }

## Was ist SearXNG?

SearXNG ist eine freie, dezentrale Suchmaschine, die Suchanfragen an verschiedene Suchmaschinen weiterleitet, ohne dabei Daten zu sammeln oder zu speichern. Im Gegensatz zu kommerziellen Suchmaschinen wie Google oder Bing bietet SearXNG eine anonyme und datenschutzfreundliche Alternative.

Die Einrichtung ist besonders nützlich, um KI Tools wie Open WebUI oder opencode den Zugang zu aktuellen Informationen aus dem Internet zu ermöglichen. Dadurch werden die KI-Agenten noch leistungsfähiger.

## Vorbereitung des Umgebungsverzeichnisses

Bevor wir mit der Installation beginnen, erstellen wir zunächst das notwendige Verzeichnisstruktur:

```bash
mkdir -p ./searxng/core-config/
cd ./searxng/
```

Das Verzeichnis `core-config` wird später für die Konfigurationsdateien verwendet, um die Einstellungen von SearXNG zu speichern.

## Herunterladen der Docker-Konfiguration

Als Nächstes laden wir die Docker Compose Konfigurationsdatei herunter, die für die Installation von SearXNG benötigt wird:

```bash
curl -fsSL \
  -O https://raw.githubusercontent.com/searxng/searxng/master/container/docker-compose.yml \
  -O https://raw.githubusercontent.com/searxng/searxng/master/container/.env.example
```

Diese Befehle laden zwei Dateien herunter:
1. `docker-compose.yml` - Die Hauptkonfigurationsdatei für Docker Compose
2. `.env.example` - Eine Beispieldatei für Umgebungsvariablen

## Erstellen der Konfigurationsdatei

Nachdem wir die Beispielkonfigurationsdatei heruntergeladen haben, kopieren wir sie und passen sie an:

```bash
cp -i .env.example .env
```

Die `-i` Option sorgt dafür, dass wir gefragt werden, bevor eine bestehende Datei überschrieben wird.

Anschließend bearbeiten wir die `.env` Datei um die gewünschten Einstellungen vorzunehmen. Hier die wichtigsten Parameter:

- `SEARXNG_HOST`: Die Hostadresse auf der SearXNG verfügbar sein soll
- `SEARXNG_PORT`: Der Port, auf dem SearXNG lauschen wird (Standard ist 8080)
- `SEARXNG_BASE_URL`: Die Basis-URL für SearXNG

## Starten und Stoppen der Dienste

Nun können wir die Docker-Container starten:

```bash
docker compose up -d
```

Das `-d` Flag sorgt dafür, dass die Container im Hintergrund gestartet werden.

Um die Dienste zu stoppen:

```bash
docker compose down
```

Dies beendet alle Container und entfernt sie.

## Konfigurieren von SearXNG

Nach dem ersten Start wird automatisch eine Standard-Konfigurationsdatei namens `settings.yml` erstellt. Diese finden wir unter `core-config/settings.yml`.

Wir öffnen diese Datei und passen Sie die Einstellungen an:

```yaml
server:
  # Wird durch ${SEARXNG_PORT} und ${SEARXNG_BIND_ADDRESS} überschrieben
  port: 8080
  bind_address: "0.0.0.0"

search:
  formats:
    - html
    - json
```

Diese Einstellungen sind wichtig:
- `port`: Der Port, auf dem SearXNG auf Anfragen hört
- `bind_address`: Die IP-Adresse, auf der SearXNG horcht (0.0.0.0 bedeutet alle Netzwerkschnittstellen)
- `formats`: Die Formate, in denen Suchergebnisse zurückgegeben werden sollen (HTML und JSON sind am nützlichsten)

## Sicherheit der Container

Nach dem ersten Durchlauf sollten Sie das Sicherheitsniveau erhöhen, indem Sie die Containerbeschränkungen konfigurieren:

```yaml
services:
  core:
    container_name: searxng-core
    image: docker.io/searxng/searxng:${SEARXNG_VERSION:-latest}
    restart: always
    ports:
      - ${SEARXNG_HOST:+${SEARXNG_HOST}:}${SEARXNG_PORT:-8080}:${SEARXNG_PORT:-8080}
    env_file: ./.env
    volumes:
      - ./core-config/:/etc/searxng/:Z
      - core-data:/var/cache/searxng/
    cap_drop:
      - ALL
```

Die Zeile `cap_drop: - ALL` entfernt alle Linux-Systemrechte vom Container, was die Sicherheit erheblich erhöht, da der Container nur noch die minimal benötigten Rechte hat.

## Starten von SearXNG

Abschließend starten wir SearXNG erneut:

```bash
docker compose up -d
```

Sie können nun in Ihrem Browser zur URL `http://localhost:8080` navigieren, um SearXNG zu verwenden.

## Integration mit Open WebUI und opencode

Nach erfolgreicher Einrichtung können Sie SearXNG in Open WebUI oder opencode integrieren, um Ihren KI-Agenten mit aktuellen Informationen aus dem Internet zu versorgen. Dies ermöglicht es den Agenten, aktuelle Daten zu recherchieren und präzisere Antworten zu liefern.

Durch diese Einrichtung erhalten Sie ein vollständiges, sicherheitsbewusstes und datenschutzfreundliches Websearch-System, das perfekt für KI-Anwendungen geeignet ist.

Detaillierte Informationen zur Integration in opencode finden sich um [Opencode
Blogbeitrag](../local-ai-opencode/).
