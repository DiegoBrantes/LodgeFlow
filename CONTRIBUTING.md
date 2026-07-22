# Contribuindo

Obrigado pelo interesse no LodgeFlow.

---

## O escopo deste repositório

Antes de tudo, uma distinção importante:

> Este repositório contém **apenas documentação técnica**. O código-fonte do LodgeFlow é proprietário
> e não está publicado aqui.

Isso significa que contribuições de **código da aplicação não são aceitas** — não há código de
aplicação neste repositório para modificar.

**São muito bem-vindas** contribuições à documentação.

---

## Como contribuir

### 1. Correções e melhorias na documentação

Erros de digitação, trechos confusos, links quebrados, diagramas incorretos, explicações que poderiam
ser mais claras — tudo isso é bem-vindo.

1. Faça um fork do repositório
2. Crie um branch a partir de `main`
3. Faça a alteração
4. Abra um Pull Request descrevendo o que mudou e por quê

### 2. Perguntas e discussões sobre arquitetura

Abra uma **issue** para:

- Perguntar sobre uma decisão arquitetural documentada
- Sugerir um tópico ainda não coberto
- Apontar uma inconsistência entre documentos
- Discutir trade-offs descritos em [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md)

Discussões técnicas são o principal valor deste repositório. Perguntas boas costumam melhorar a
documentação.

---

## O que não pode ser aceito

Pull Requests que incluam qualquer um destes itens serão **fechados sem merge**:

| Não aceito | Motivo |
|---|---|
| Código-fonte da aplicação | Fora do escopo deste repositório |
| Endpoints, hosts ou URLs reais da API | Informação não pública |
| Credenciais, chaves ou tokens — **de qualquer natureza** | Risco de segurança |
| Prompts de IA ou instruções de sistema | Propriedade intelectual |
| Esquemas reais de banco de dados | Informação estratégica |
| Regras de monetização, precificação ou entitlement | Informação comercial |
| Dados de usuários, mesmo anonimizados | Privacidade |
| Detalhes de infraestrutura interna | Segurança |

Se você tem dúvida se algo se enquadra, **assuma que sim** e pergunte em uma issue antes de abrir o
PR.

---

## Padrões de documentação

Ao editar arquivos deste repositório:

**Conteúdo**
- Explique o **porquê**, não apenas o quê. O valor está no raciocínio.
- Descreva conceitos e padrões, nunca implementação.
- Documente os trade-offs. Uma decisão sem custo declarado parece pouco pensada.
- Mantenha os avisos de conteúdo fictício nos exemplos.

**Forma**
- Markdown, com títulos em hierarquia consistente
- Diagramas em **Mermaid**, para que o GitHub renderize e o diff seja legível
- Português como idioma principal
- Linhas com cerca de 100 caracteres, para facilitar a revisão em diff
- Links relativos entre documentos do repositório

**Exemplos**
- Domínios `.invalid` ou `.example` — reservados por norma, nunca resolvem
- Identificadores obviamente sintéticos (`00000000-...`)
- Datas no futuro distante, para não parecerem reais
- Placeholders explícitos (`<TOKEN_AQUI>`) no lugar de valores

---

## Segurança

### Encontrou um problema de segurança?

**Não abra uma issue pública.**

Se você identificou uma vulnerabilidade no produto LodgeFlow, ou percebeu que este repositório expõe
inadvertidamente alguma informação sensível, reporte de forma privada:

- Use o **Security Advisory** privado do GitHub, na aba *Security* deste repositório
- Ou entre em contato diretamente com o mantenedor

Compromissos:

- Confirmação de recebimento assim que possível
- Você será mantido informado sobre o andamento
- Crédito público pela descoberta, se desejar

Por favor, dê um prazo razoável para correção antes de qualquer divulgação pública.

### Encontrou uma credencial exposta neste repositório?

Reporte imediatamente pelo canal privado acima. Não abra issue nem PR — isso apenas amplia a
exposição.

---

## Código de conduta

Seja respeitoso e construtivo. Críticas técnicas são bem-vindas; ataques pessoais não. Discorde de
ideias, não de pessoas.

---

## Licença das contribuições

Ao contribuir, você concorda que sua contribuição será licenciada sob a
[Licença MIT](LICENSE), nos mesmos termos da documentação deste repositório.
