# Infra Guardian — modelos de documentos

Cinco modelos curtos que o agente usa ao entregar decisões e planos. Copie o bloco, preencha e guarde onde a sua organização guarda documentação operacional (não necessariamente no repositório).

Regras para todos: sem segredo; números só medidos ou com premissas; cada ação com dono e critério verificável.

## 1. ADR — registro de decisão de arquitetura

```markdown
# ADR-<nnn>: <decisão em uma frase>

- Data: <AAAA-MM-DD> · Status: proposta | aceita | substituída por ADR-<nnn>
- Decisores: <nomes ou papéis>

## Contexto
<O problema, com evidência: incidente, métrica, requisito, custo. O que acontece se nada mudar.>

## Opções consideradas
| Opção | Custo | Complexidade | Operação | Confiabilidade | Risco de migração |
|---|---|---|---|---|---|
| Manter o atual | | | | | |
| A | | | | | |
| B | | | | | |

## Decisão
<A opção escolhida e por quê, ligada aos requisitos reais (carga, SLO, equipe, orçamento).>

## Consequências
<O que melhora, o que piora, o que passa a ser preciso operar, quando revisar esta decisão.>
```

## 2. Plano de mudança

```markdown
# Mudança: <título> — <ambiente> — <data e janela, com fuso>

| Campo | Valor |
|---|---|
| Ativo e ambiente | |
| Dono / quem executa / quem aprova | |
| Objetivo e critério de sucesso | |
| Risco e raio de impacto | |
| Dependências | |
| Linha de base (métricas antes) | |
| Backup e restore disponíveis (testado em) | |
| Acesso fora de banda (se a mudança pode cortar acesso) | |

## Pré-checks
- [ ] <ex.: backup de hoje restaurável; espaço em disco; réplicas saudáveis>

## Passos
1. <passo — comando ou ação — tempo esperado — como saber que deu certo>

## Critérios de aborto
- <ex.: taxa de erro acima de X% por Y min; latência P95 acima de Z; perda de acesso>

## Rollback ou forward-fix
1. <passo — tempo esperado — como confirmar a volta>

## Pós-checks e evidências a guardar
- [ ] <ex.: health OK, métricas iguais ou melhores que a linha de base por N min>

## Aprovação
<nome — data e hora — o que foi aprovado (esta mudança, nesta janela)>
```

## 3. SLO

```markdown
# SLO: <jornada do usuário>

- Serviço e dono: <...>
- Jornada: <ex.: cliente envia mensagem e recebe resposta>
- SLI (especificação): <o que o usuário sente, ex.: proporção de mensagens respondidas em até N s>
- SLI (implementação): <de onde vem a medida e o que ela não enxerga>
- Meta e janela: <ex.: X% em janela móvel de 4 semanas> — base: <histórico medido ou "a definir após N semanas de medição">
- Error budget: <quanto de falha a meta permite na janela>
- Política de error budget: <o que muda quando o orçamento acaba; quem aprovou; quando revisar>
- Alertas: <consumo rápido e lento do orçamento; dono; runbook>
- Não é SLA: <compromissos contratuais, se existirem, ficam em documento próprio>
```

## 4. Incidente e postmortem

```markdown
# Incidente <AAAA-MM-DD> — <título curto>

- Severidade: <P0 | P1 | ...> · Duração: <início — fim, com fuso>
- Coordenação: <quem> · Comunicação: <quem> · Operação: <quem>
- Impacto: <quem foi afetado, o que deixou de funcionar, dados perdidos ou não>

## Linha do tempo (fuso: <...>)
| Hora | Fato | Fonte |
|---|---|---|

## Hipóteses
| Hipótese | A favor | Contra | Resultado |
|---|---|---|---|

## Estabilização
<O que foi feito, por quem, com que risco; o que foi preservado antes.>

## Postmortem (sem culpados)
- Causa: <...>
- Fatores contribuintes: <...>
- O que funcionou: <...>
- O que atrapalhou: <...>

## Ações
| Ação | Dono | Prazo | Critério verificável |
|---|---|---|---|
```

## 5. Plano de recuperação de desastre (DR)

```markdown
# DR: <sistema>

- Dono: <...> · Última revisão: <data> · Último teste: <data e resultado>
- RPO aceito: <valor> — aprovado por <quem>, em <data>
- RTO aceito: <valor> — aprovado por <quem>, em <data>

## O que precisa voltar
| Item | Onde está | Como é copiado | Destino da cópia | Frequência | Retenção | Último restore testado |
|---|---|---|---|---|---|---|
| Banco | | | | | | |
| Arquivos e uploads | | | | | | |
| Configuração (sem segredos) | | | | | | |
| Segredos e certificados (custódia) | | | | | | |
| DNS e IaC | | | | | | |

## Cenários
| Cenário | Estratégia | Passos resumidos | Tempo medido no último teste |
|---|---|---|---|
| Perda do servidor | | | |
| Exclusão ou corrupção de dados | | | |
| Perda do provedor ou da região | | | |
| Ransomware | | | |

## Teste de restore
<Ambiente isolado, passos, verificações de integridade e de aplicação, RTO e RPO medidos, problemas, próxima data.>
```
