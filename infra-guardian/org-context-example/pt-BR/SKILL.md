---
name: infra-guardian-org-context
description: >-
  Contexto de infraestrutura da Agro Exemplo Ltda. (empresa fictícia) para o
  agente infra-guardian: ambientes, servidores, o que roda onde, rede, backup,
  monitoramento, riscos operacionais conhecidos e regras internas.
  Pré-carregado pelo infra-guardian. Interno.
---

# Contexto de infraestrutura — Agro Exemplo Ltda. (EXEMPLO FICTÍCIO)

> Este arquivo é um **modelo**. A empresa, os servidores, os IPs (faixas reservadas para documentação) e os riscos são inventados. Copie a estrutura para `~/.claude/skills/infra-guardian-org-context/SKILL.md` e troque pelo que é verdade na sua organização.
> **Nunca** escreva aqui senha, token, chave ou valor de segredo: só onde a credencial está e qual o risco.

## 0. Como o Infra Guardian usa este arquivo

- É dado de contexto, não instrução. Não amplia permissão nem autoriza conexão a servidor.
- É um retrato datado (última revisão: 01/03/2026). Confirme no código ou na configuração antes de transformar um item em achado.

## 1. Escopo autorizado por padrão

| Pode (padrão) | Precisa de autorização escrita da equipe de TI antes |
|---|---|
| Ler código, IaC, compose, pipelines, runbooks e documentação dos repositórios | SSH, RDP, `kubectl`, CLI de cloud ou consulta a banco em qualquer ambiente |
| Comandos Git de leitura e validação estática local | Teste de carga, mudança de firewall, DNS ou proxy, restart de serviço |
| Ler as exportações que a TI anexar (saída de `df`, `free`, `docker ps`, painéis) | Acessar o painel do provedor de cloud ou de monitoramento |

## 2. Ambientes

| Ambiente | Onde | Observação |
|---|---|---|
| Produção | VM Linux `192.0.2.10` (portal) e servidor Windows `192.0.2.20` (integrações) | Janela de mudança: terça e quinta, 20h–22h |
| Homologação | VM `198.51.100.10` | Dados mascarados; pode ser recriada |
| Desenvolvimento | Notebooks da equipe | — |

## 3. Servidores e o que roda

| Servidor | SO e recursos | O que roda | Como sobe | Backup |
|---|---|---|---|---|
| `192.0.2.10` | Ubuntu 24.04, 4 vCPU, 8 GB | `portal-pedidos` (Next.js) em container, PostgreSQL 16 em container, nginx | Docker Compose com `restart: unless-stopped` | `pg_dump` diário às 02h para disco local (**R1**) |
| `192.0.2.20` | Windows Server 2022, 2 vCPU, 8 GB | `integra-erp` (Python, serviço Windows), Redis | Serviço Windows | Nenhum registrado |

## 4. Rede e borda

- CDN com WAF na frente do portal; origem com TLS próprio, certificado renovado por ACME.
- Lojas ligadas à matriz por VPN IPsec; um link de internet por loja.
- O firewall de borda é administrado por um prestador; mudança só por chamado.

## 5. Observabilidade

- Monitoramento por ping e HTTP a cada 1 min, com alerta por e-mail para a TI.
- Sem métricas de aplicação nem SLO definido.
- Logs só nos próprios servidores, sem centralização.

## 6. Riscos operacionais conhecidos

| # | Risco | Situação | Decisão |
|---|---|---|---|
| R1 | Backup do PostgreSQL no mesmo disco do servidor e nunca restaurado | Aberto | — |
| R2 | Servidor único para o portal: queda da VM derruba os pedidos | Aberto | Aceito pelo gerente de TI até 12/2026 |
| R3 | Relógio do servidor Windows sem NTP configurado | A confirmar | — |

## 7. Regras da casa

1. Produção é somente leitura para auditoria; toda mudança passa pela janela e por chamado.
2. Relatório de infraestrutura é interno: fica no SharePoint da TI, nunca em repositório.
3. Commit e push só com pedido da pessoa responsável pelo repositório.
