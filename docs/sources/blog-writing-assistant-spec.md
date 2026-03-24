# Blog Writing Assistant — Original Spec

> **Source:** Notion T-0 Projects DB → "Blog Writing Assistant"
> **Page ID:** `2ccc0fdf-942d-81ce-91ca-e2099c8862c3`
> **Status (in Notion):** Prepared
> **Owner:** Aaron Kokal
> **Created:** 2025-12-20
> **Last edited:** 2026-02-20

## Project Mission

Top-Priorität für Aaron. Ziel ist eine extrem flüssige Content-Produktion: Wir diskutieren News und Themen, und wenn wir auf etwas Interessantes stoßen, soll per Knopfdruck daraus ein lesbarer, hochwertiger Content entstehen. Die Quadratur des Kreises – Content extrem schnell erstellen und trotzdem hohe Qualität liefern.

## Full Description

AI-gestützter Workflow für die Erstellung von Blogposts: Von der ersten Idee über Research und iterative Verfeinerung bis zur Veröffentlichung auf Ghost – inklusive automatischer Bildgenerierung nach CI-Vorgaben.

## Workflow Phases

### Phase 1: Ideation & Ressourcen-Sammlung
- User erzählt dem Agent die Geschichte/Idee zum Thema
- Verknüpfung mit existierenden Notes (Status != Archived)
- Agent sucht eigenständig passende Notes aus Notes DB
- Externe Links als zusätzliche Ressourcen

### Phase 2: CI & Format-Vorgaben laden
- Resources DB: Brand Guidelines, Blogpost-Struktur, Tonalität
- Skill/Ressource sollte mit relevanten CI-Ressourcen verknüpft sein
- Klassifizierung der Ressourcen für zuverlässiges Auffinden definieren

### Phase 3: Content Creation DB – Erster Entwurf
- Neuer Eintrag in Content Creation DB
- Alle verwendeten Quellen/Notes an relevanter Stelle verlinkt
- Erste Idee/Outline als Entwurf

### Phase 4: Self-Review & Gap-Analysis
- System prüft: Passt die Idee? Ist sie vollständig?
- Identifiziert Lücken für weiteren Research
- Dokumentiert offene Punkte in der Content Creation Page

### Phase 5: Human-in-the-Loop Review
- User reviewed Entwurf + Gap-Analysis
- Entscheidung: Weiterer Research nötig? → Perplexity/Web-Recherche
- Oder: Weiterarbeiten mit vorhandenem Material
- Bei Research: Neue Notes anlegen und verknüpfen

### Phase 6: Finalisierung des Contents
- Klarer Break/Trenner in der Page
- Bestmögliche Version des Blogposts
- Iterative Feedback-Schleifen mit User möglich
- Nur aktuelle Version wird weiterverwendet

### Phase 7: Ghost Publishing
- User gibt Freigabe zur Veröffentlichung
- Blogpost wird auf Ghost erstellt (Markdown)
- Nur finale Version, keine alten Entwürfe

### Phase 8: Bildgenerierung & Platzierung
- Analyse: Welche Abschnitte brauchen Illustrationen?
- Bedeutungszusammenhänge identifizieren
- Anzahl und Platzierung der Bilder bewusst entscheiden
- Bildgenerierung via N8N-Workflow (Nano-Banana o.ä.)
- CI-Ressourcen als Kontext für konsistenten Stil

## Required Components
- Notes DB Integration (Suche, Status-Filter)
- Resources DB (CI, Brand Guidelines, Blogpost-Vorgaben)
- Content Creation DB (Workflow-Tracking)
- Web-Recherche Tools (Perplexity etc.)
- Ghost API Integration
- N8N-Workflow für Bildgenerierung

## Linked Notes (13 total, in Notion)

These notes are linked to the Blog Writing Assistant project in Notion and contain related discussions, ideas, and research. They should be reviewed when defining specs for this pipeline.

## Tags
Notion, Ghost, T-0 Hub, Claude Code
