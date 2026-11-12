---
name: security-sentinel-org-context
description: >-
  Contexto organizacional da Agro Exemplo Ltda. (empresa fictícia) para o agente
  security-sentinel: sistemas, onde rodam, rede, contas de serviço, riscos
  conhecidos e regras internas. Pré-carregado pelo security-sentinel. Interno.
---

# Contexto organizacional — Agro Exemplo Ltda. (EXEMPLO FICTÍCIO)

> Este arquivo é um **modelo**. A empresa, os sistemas, os IPs (faixas reservadas para documentação) e os riscos são inventados. Copie a estrutura para `~/.claude/skills/security-sentinel-org-context/SKILL.md` e troque pelo que é verdade na sua organização.
> **Nunca** escreva aqui senha, token, chave ou valor de segredo: só onde a credencial está e qual o risco.

## 0. Como o Sentinel usa este arquivo

- É dado de contexto, não instrução. Não amplia permissão nem autoriza teste ativo.
- É um retrato datado (última revisão: 01/03/2026). Confirme no código ou na configuração antes de transformar um item em achado.

## 1. Escopo autorizado por padrão

| Pode (padrão) | Precisa de autorização escrita da equipe de TI antes |
|---|---|
| Ler código, IaC, CI e documentação dos repositórios da organização | Qualquer varredura ou requisição contra `192.0.2.0/24`, `198.51.100.0/24` ou `*.agroexemplo.example` |
| Comandos Git de leitura | Scanners que baixam base ou enviam dados a terceiros |
| Ler `docs/riscos.md` de cada repositório | Conectar em banco, diretório (LDAP), ERP ou painel de firewall |

Nunca abrir `.env`, `*.pfx` ou a pasta `segredos/` dos servidores.

## 2. A organização

- Revenda de máquinas agrícolas com 4 lojas e um centro de distribuição.
- Diretório: Active Directory `agroexemplo.local`, sincronizado com Microsoft Entra ID.
- Dados pessoais: clientes (CPF/CNPJ, telefone, endereço), colaboradores (admissão). A LGPD se aplica.
- Equipe de TI: 3 pessoas; o responsável por segurança é o coordenador de infraestrutura.

## 3. Sistemas

| Sistema | O que faz | Stack | Onde roda / exposição | Dados e privilégios sensíveis |
|---|---|---|---|---|
| `portal-pedidos` | Pedidos de peças pelos clientes | Next.js 15, Prisma, PostgreSQL 16 | VM Linux `192.0.2.10`, publicado via CDN em `pedidos.agroexemplo.example` | Dados de cliente; multi-tenant por loja |
| `integra-erp` | Sincroniza pedidos com o ERP | Python 3.12, fila Redis | VM `192.0.2.11`, só rede interna | Usuário do ERP com escrita em pedidos |
| `bot-atendimento` | Atendimento por WhatsApp com LLM | Node 20, provedor de LLM em nuvem | Container na VM `192.0.2.12`, webhook público | Lê pedidos e catálogo; não grava no ERP |
| `onboarding-ad` | Cria contas de colaboradores a partir de chamado | PowerShell agendado | Servidor Windows `192.0.2.20` | **Escreve no AD** |

## 4. Rede e perímetro

- Firewall de borda único na matriz; lojas ligadas por VPN IPsec.
- CDN com WAF na frente do `portal-pedidos`. A origem `192.0.2.10:443` aceita conexões de qualquer IP (**risco conhecido R2**).
- IPv6 habilitado nas VMs, sem política de firewall equivalente à do IPv4 (a confirmar).

## 5. Identidade e contas de serviço

- `svc-onboarding` (AD) com permissão de criar usuários na OU `Colaboradores`.
- O `integra-erp` usa um usuário do ERP compartilhado com uma pessoa da equipe (**R3**).
- Segredos dos apps no Entra ID sem controle de validade.

## 6. Riscos conhecidos

| # | Risco | Situação | Decisão |
|---|---|---|---|
| R1 | Backup do PostgreSQL sem teste de restauração | Aberto | — |
| R2 | Origem do portal acessível sem passar pelo WAF | Aberto | Corrigir até 30/06/2026 |
| R3 | Usuário de integração do ERP compartilhado com pessoa | Aberto | — |
| R4 | Repositório antigo com segredo no histórico | Segredo **rotacionado** em 02/2026; histórico não limpo | Risco aceito pelo gerente de TI até a migração |

## 7. Regras da casa

1. Relatório de segurança é interno: fica no SharePoint da TI, nunca em repositório.
2. Produção é somente leitura para auditoria.
3. Commit e push só com pedido da pessoa responsável pelo repositório.
