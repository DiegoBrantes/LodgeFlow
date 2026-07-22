# Changelog

Todas as mudanças relevantes desta **documentação** são registradas aqui.

O formato segue [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/), e o versionamento segue
[Semantic Versioning](https://semver.org/lang/pt-BR/).

> Este changelog documenta o **repositório de documentação**, não o histórico de versões do aplicativo
> LodgeFlow.

---

## [1.0.0] — 2026-07-22

Publicação inicial da documentação técnica do LodgeFlow.

### Adicionado

**Documento principal**
- `SYSTEM_DESIGN.md` — decisões arquiteturais, alternativas consideradas, trade-offs assumidos,
  estratégia de escala e decisões que foram revistas ao longo do projeto

**Documentação técnica**
- `docs/system-overview.md` — visão geral do sistema e seus domínios
- `docs/architecture.md` — arquitetura em sete camadas, detalhada
- `docs/tech-stack.md` — stack completa por categoria, incluindo o que foi deliberadamente evitado
- `docs/backend.md` — runtime serverless, roteamento, autenticação, persistência e trabalho agendado
- `docs/mobile.md` — estratégia multiplataforma e camada nativa iOS
- `docs/database.md` — modelo de dados conceitual, princípios de modelagem e estratégia de migrations
- `docs/ai.md` — arquitetura multi-provedor, controle de custo e conformidade com a LGPD
- `docs/security.md` — modelo de segurança, autenticação, autorização e privacidade
- `docs/deployment.md` — estratégia de publicação nos cinco alvos de distribuição

**Diagramas** (Mermaid, renderizados nativamente pelo GitHub)
- `docs/diagrams/architecture.md` — camadas, compartilhamento de código e isolamento de integrações
- `docs/diagrams/request-flow.md` — requisição autenticada, caminho offline, operação de IA, webhook
  e entrega de lembrete
- `docs/diagrams/database.md` — entidades, agrupamento por domínio e ciclo de vida de registro

**Exemplos e templates** — todos fictícios
- `api/openapi-example.yaml` — contrato de API ilustrativo
- `examples/example-api-request.json` — requisição com idempotência
- `examples/example-api-response.json` — respostas de sucesso e de erro
- `examples/example-webhook.json` — evento de webhook com verificação de assinatura
- `examples/example-jwt-payload.json` — estrutura de claims de token
- `templates/env.example` — nomes de variáveis, sem valores
- `templates/docker-compose.example.yml` — ambiente local ilustrativo

**Repositório**
- `README.md` — apresentação, funcionalidades, arquitetura e índice da documentação
- `LICENSE` — MIT para a documentação, com aviso de propriedade sobre o produto
- `CONTRIBUTING.md` — escopo de contribuição e política de divulgação responsável
- `.gitignore`

### Nota sobre o conteúdo

Toda a documentação foi revisada para garantir que **não contém**: código-fonte proprietário,
endpoints reais, credenciais, chaves, tokens, prompts de IA, esquemas de banco de produção, regras de
monetização ou qualquer detalhe de infraestrutura interna.

---

[1.0.0]: https://github.com/DiegoBrantes/LodgeFlow/releases/tag/v1.0.0
