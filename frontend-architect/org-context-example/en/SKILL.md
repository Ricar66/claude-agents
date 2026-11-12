---
name: frontend-architect-org-context
description: >-
  Visual standard and frontend context of Example Agro Ltd. (fictitious
  company) for the frontend-architect agent: colors, typography, logos, images,
  screen patterns, systems and internal rules. Preloaded by
  frontend-architect. Internal.
---

# Visual standard and frontend — Example Agro Ltd. (FICTITIOUS EXAMPLE)

> This file is a **template**. The brand, the colors and the systems are made up. Copy the structure to `~/.claude/skills/frontend-architect-org-context/SKILL.md` and replace it with your organization's real standard.

## 0. How the agent uses this file

- In this organization's systems, the standard below is **mandatory**, unless the project already has an approved theme of its own.
- It is a snapshot of 2026-03-01: check the repository's tokens before changing anything.

## 1. Brand colors

| Token | Hex | Use |
|---|---|---|
| `--cor-primaria` | `#1F5F3A` | main actions, header |
| `--cor-primaria-contraste` | `#FFFFFF` | text on the primary color |
| `--cor-destaque` | `#E0A526` | occasional accent, never text on white |
| `--cor-fundo` | `#F6F7F5` | page background |
| `--cor-texto` | `#1C2420` | main text |
| `--cor-erro` | `#B3261E` | errors and destructive actions |

Contrast checked: `#1C2420` text on `#F6F7F5` passes AA. The `#E0A526` accent does not pass as text on white.

## 2. Typography and images

- Font: Inter (headings and body text), loaded locally; tabular numbers in tables.
- Logo: `assets/marca/logo-agro-exemplo.svg` (light and dark versions). Clear space equal to the height of the letter "A". Never distort or recolor it.
- Photos: only from the image bank approved by communications; no generated images that imitate the brand.

## 3. Screen patterns

- Login through the corporate provider, with the logo at the top and the system name.
- Header with the logo on the left, the system name and the user on the right.
- Dense tables with filters in the URL; cards on phones.
- Interface text in Brazilian Portuguese.

## 4. Systems

| System | Stack | Note |
|---|---|---|
| `portal-pedidos` | Next.js 15 App Router, Tailwind with the tokens above | used by customers on phones |
| `painel-interno` | Vite + React 19 | internal use, dense screens |
