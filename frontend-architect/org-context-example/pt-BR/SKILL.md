---
name: frontend-architect-org-context
description: >-
  Padrão visual e contexto de frontend da Agro Exemplo Ltda. (empresa
  fictícia) para o agente frontend-architect: cores, tipografia, logos,
  imagens, padrões de tela, sistemas e regras internas. Pré-carregado pelo
  frontend-architect. Interno.
---

# Padrão visual e frontend — Agro Exemplo Ltda. (EXEMPLO FICTÍCIO)

> Este arquivo é um **modelo**. A marca, as cores e os sistemas são inventados. Copie a estrutura para `~/.claude/skills/frontend-architect-org-context/SKILL.md` e troque pelo padrão real da sua organização.

## 0. Como o agente usa este arquivo

- Nos sistemas desta organização, o padrão abaixo é **obrigatório**, salvo tema próprio já aprovado no projeto.
- É retrato de 01/03/2026: confirme nos tokens do repositório antes de mudar algo.

## 1. Cores da marca

| Token | Hex | Uso |
|---|---|---|
| `--cor-primaria` | `#1F5F3A` | ações principais, cabeçalho |
| `--cor-primaria-contraste` | `#FFFFFF` | texto sobre a primária |
| `--cor-destaque` | `#E0A526` | destaque pontual, nunca texto sobre branco |
| `--cor-fundo` | `#F6F7F5` | fundo das páginas |
| `--cor-texto` | `#1C2420` | texto principal |
| `--cor-erro` | `#B3261E` | erros e ações destrutivas |

Contraste conferido: texto `#1C2420` sobre `#F6F7F5` passa em AA. O destaque `#E0A526` não passa como texto sobre branco.

## 2. Tipografia e imagens

- Fonte: Inter (títulos e texto), carregada localmente; números tabulares em tabelas.
- Logo: `assets/marca/logo-agro-exemplo.svg` (versão clara e escura). Área de proteção igual à altura da letra "A". Nunca distorcer nem recolorir.
- Fotos: só do banco de imagens aprovado pela comunicação; sem imagens geradas que imitem a marca.

## 3. Padrões de tela

- Login pelo provedor corporativo, com o logo no topo e o nome do sistema.
- Cabeçalho com logo à esquerda, nome do sistema e usuário à direita.
- Tabelas densas com filtros na URL; em celular, cartões.
- Textos de interface em português do Brasil.

## 4. Sistemas

| Sistema | Stack | Observação |
|---|---|---|
| `portal-pedidos` | Next.js 15 App Router, Tailwind com os tokens acima | usado por clientes no celular |
| `painel-interno` | Vite + React 19 | uso interno, telas densas |
