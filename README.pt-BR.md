# claude-agents

[English](README.md) · **Português**

Coleção de subagentes para o [Claude Code](https://code.claude.com/docs/en/sub-agents), cada um com o agente pronto para instalar, documentação, avaliações e um jeito de receber o contexto da sua organização sem que esse contexto entre no repositório.

## Agentes

| Agente | O que faz | Status |
|---|---|---|
| [`security-sentinel`](security-sentinel/) | Auditoria de segurança somente leitura e baseada em evidência: código, APIs, rede por arquivos, infraestrutura, cadeia de suprimentos, CI e sistemas de IA (LLM, agentes, MCP, RAG). Onze modos, de `triage` a `release-gate` | Inicial (v0.1) |
| [`infra-guardian`](infra-guardian/) | Revisão de infraestrutura e confiabilidade somente leitura (SRE, DevOps, plataforma, banco, FinOps): VPS, Docker, Kubernetes, IaC, rede, CI/CD, observabilidade, backup e DR, capacidade, incidente e custo. Dezessete modos, de `discover` a `verify-change`; nunca aplica mudança, entrega plano com rollback. Vem com a variante `static`, sem Bash | Inicial (v0.1) |

| [`backend-architect`](backend-architect/) | Arquiteto e engenheiro de backend: APIs, regras de domínio, banco, integrações, migrações e testes. Lê o repositório inteiro antes de propor, preserva o que funciona e **implementa quando pedido**. Quinze modos, de `discover` a `verify` | Inicial (v0.1) |
| [`frontend-architect`](frontend-architect/) | Arquiteto e engenheiro de frontend: UX, design system, acessibilidade (WCAG 2.2 AA), responsividade, renderização, performance e testes. Segue a identidade visual existente e **implementa quando pedido**. Dezessete modos | Inicial (v0.1) |

Os agentes se complementam: os dois auditores (Sentinel e Guardian) só leem; os dois arquitetos desenham e, se você pedir, implementam. **Como usar e chamar cada um, e como eles trabalham juntos: [`GUIA.pt-BR.md`](GUIA.pt-BR.md).**

## Como cada agente é organizado

```text
<agente>/
├── agent/
│   ├── pt-BR/<agente>.md     ← copie UM idioma para ~/.claude/agents/
│   └── en/<agente>.md        (alguns agentes têm variantes, ex.: <agente>-static.md)
├── org-context-example/      ← modelo fictício do contexto organizacional
│   ├── pt-BR/SKILL.md
│   └── en/SKILL.md
└── docs/
    ├── pt-BR/
    └── en/
```

## Contexto organizacional fica na sua máquina

Agente genérico erra mais. Por isso cada agente pode pré-carregar uma skill local, `<agente>-org-context`, com o que só a sua organização sabe: sistemas, onde rodam, rede, contas, riscos conhecidos e regras internas.

- A skill fica em `~/.claude/skills/<agente>-org-context/SKILL.md`, **fora deste repositório**.
- Se ela não existir, o Claude Code pula o pré-carregamento e o agente roda no modo genérico.
- O `.gitignore` deste repositório ignora pastas `*-org-context/` por segurança, mas a regra principal é: contexto real nunca entra aqui.
- Nunca coloque senha, token ou chave no contexto. Escreva onde a credencial está e qual o risco.

## Instalação rápida

```powershell
# Windows (PowerShell)
Copy-Item .\security-sentinel\agent\pt-BR\security-sentinel.md "$env:USERPROFILE\.claude\agents\"
Copy-Item .\infra-guardian\agent\pt-BR\infra-guardian.md "$env:USERPROFILE\.claude\agents\"
```

```bash
# Linux e macOS
cp security-sentinel/agent/pt-BR/security-sentinel.md infra-guardian/agent/pt-BR/infra-guardian.md ~/.claude/agents/
```

Para instalar os quatro de uma vez, veja o [guia](GUIA.pt-BR.md), seção 2. Depois, numa sessão nova:

- `Use o agente security-sentinel para triage deste repositório, somente leitura.`
- `Use o agente infra-guardian para discover deste repositório, somente leitura.`

Detalhes, permissões recomendadas e limitações: [`security-sentinel/docs/pt-BR/SECURITY_SENTINEL_SETUP.md`](security-sentinel/docs/pt-BR/SECURITY_SENTINEL_SETUP.md) e [`infra-guardian/docs/pt-BR/INFRA_GUARDIAN_SETUP.md`](infra-guardian/docs/pt-BR/INFRA_GUARDIAN_SETUP.md).

## Idiomas

Todo agente e todo documento existem em português (`pt-BR`) e inglês (`en`). As duas versões mudam juntas.

## Licença

[Apache-2.0](LICENSE).

## Aviso

Estes agentes ajudam a revisar; não substituem pentest autorizado, revisão humana, processo de mudança nem parecer jurídico. Um modelo de linguagem pode errar: confira a evidência de cada achado antes de agir.
