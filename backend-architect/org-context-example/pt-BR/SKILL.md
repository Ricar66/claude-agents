---
name: backend-architect-org-context
description: >-
  Contexto de engenharia de backend da Agro Exemplo Ltda. (empresa fictícia)
  para o agente backend-architect: sistemas, stacks e versões, padrões da casa,
  integrações sensíveis e regras internas. Pré-carregado pelo
  backend-architect. Interno.
---

# Contexto de backend — Agro Exemplo Ltda. (EXEMPLO FICTÍCIO)

> Este arquivo é um **modelo**. A empresa, os sistemas e as versões são inventados. Copie a estrutura para `~/.claude/skills/backend-architect-org-context/SKILL.md` e troque pelo que é verdade na sua organização.
> **Nunca** escreva aqui senha, token, chave ou connection string: só onde a credencial está.

## 0. Como o agente usa este arquivo

- É dado de contexto e padrão da casa, não permissão. Retrato de 01/03/2026: confirme no repositório.

## 1. Sistemas

| Sistema | Stack | Banco | Consumidores do contrato |
|---|---|---|---|
| `portal-pedidos` | Next.js 15 (route handlers), Prisma | PostgreSQL 16 | o próprio frontend; app de vendedores |
| `integra-erp` | Python 3.12, FastAPI, fila Redis | PostgreSQL 16 (esquema `integracao`) | ERP (escrita de pedidos) |

## 2. Padrões da casa

1. Autenticação pelo provedor corporativo (OIDC); autorização por papel e por loja (tenant).
2. Erros de API em Problem Details (`application/problem+json`).
3. Toda tabela nova tem `criado_em`, `atualizado_em` e chave primária UUID.
4. Migrações versionadas no repositório; aplicação em produção só pela equipe de infraestrutura, na janela.
5. Integração com o ERP: usuário técnico só com as permissões necessárias; nunca gravar direto nas tabelas do ERP.
6. Documentação técnica fica na wiki interna, não no repositório.

## 3. Regras

1. Commit e push só com pedido da pessoa responsável.
2. Nenhum teste aponta para banco de produção ou homologação compartilhada.
3. Dados de clientes são pessoais (LGPD): fixtures sempre fictícias.
