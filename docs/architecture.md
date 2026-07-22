# Arquitetura

> Documento conceitual. Não contém código-fonte, endpoints reais, esquemas de produção nem lógica de
> negócio proprietária.

---

## Modelo arquitetural

O LodgeFlow segue uma arquitetura **serverless em camadas, distribuída na edge**, com clientes
offline-first e um núcleo de interface compartilhado entre plataformas.

```mermaid
graph TB
    subgraph L1["1 · Apresentação"]
        UI["Componentes de UI<br/><i>React · Tailwind · Radix</i>"]
        NAT["Camada nativa<br/><i>Plugins Swift · Electron</i>"]
    end

    subgraph L2["2 · Estado do Cliente"]
        SRV["Estado de servidor<br/><i>Cache · revalidação · otimismo</i>"]
        LOC["Estado local durável<br/><i>Fila de mutações offline</i>"]
    end

    subgraph L3["3 · Transporte"]
        HTTP["Cliente HTTP<br/><i>JWT · retry · backoff</i>"]
    end

    subgraph L4["4 · Borda / API"]
        RT["Roteamento por domínio"]
        MW["Middleware<br/><i>Auth · CORS · Rate limit · Payload</i>"]
    end

    subgraph L5["5 · Domínio"]
        BIZ["Regras de negócio"]
        ORCH["Orquestração de integrações"]
    end

    subgraph L6["6 · Acesso a Dados"]
        DAL["Camada de acesso<br/><i>Fronteira de persistência</i>"]
    end

    subgraph L7["7 · Persistência"]
        DB[("Banco relacional<br/>distribuído")]
        OBJ[("Object storage")]
        CACHE[("Cache chave-valor")]
    end

    UI --> SRV --> HTTP
    UI --> NAT
    SRV <--> LOC
    HTTP --> RT --> MW --> BIZ
    BIZ --> ORCH
    BIZ --> DAL --> DB & OBJ & CACHE

    classDef a fill:#1e3a5f,stroke:#4a90d9,color:#fff
    classDef b fill:#7c3f00,stroke:#f38020,color:#fff
    classDef c fill:#1f4d2e,stroke:#4caf50,color:#fff
    class UI,NAT,SRV,LOC,HTTP a
    class RT,MW,BIZ,ORCH b
    class DAL,DB,OBJ,CACHE c
```

---

## Camada 1 — Apresentação

Interface construída em React com TypeScript, estilizada por Tailwind CSS sobre primitivas acessíveis
do Radix UI.

A mesma camada de apresentação serve todas as plataformas. O que muda entre elas é o **invólucro**:
navegador na web, Capacitor no mobile, Electron no desktop.

**Adaptação por plataforma**, não duplicação:
- Layout responsivo cobre celular, tablet e desktop com o mesmo componente
- Navegação inferior no mobile, navegação lateral no desktop
- Atalhos de teclado ativos apenas onde há teclado
- Feedback tátil ativo apenas onde há motor háptico

### Camada nativa

Recursos sem equivalente web entram por plugins com superfície pequena e responsabilidade única:
calendário do sistema, chamadas nativas, widgets, atividades ao vivo, comandos de voz e compras
dentro do app. A camada React os consome por uma interface tipada, sem saber como estão implementados.

---

## Camada 2 — Estado do Cliente

Dois tipos de estado, com papéis distintos:

**Estado de servidor.** Gerenciado por uma biblioteca de cache com revalidação, que trata dados
remotos como cache com política de invalidação — não como estado da aplicação. Isso remove a
necessidade de sincronizar manualmente o que veio da rede e habilita atualização otimista de forma
consistente.

**Estado local durável.** A fila de mutações offline. Sobrevive ao fechamento do app e à
reinicialização do dispositivo, e é drenada em ordem quando a conectividade retorna.

### Estratégia offline

```mermaid
stateDiagram-v2
    [*] --> Local: usuário age
    Local --> Fila: mutação enfileirada
    Local --> UI: interface atualiza (otimista)
    Fila --> Envio: há conectividade?
    Envio --> Confirmado: sucesso
    Envio --> Retry: falha transitória
    Retry --> Envio: backoff
    Envio --> Conflito: divergência de versão
    Conflito --> Resolvido: resolução determinística
    Resolvido --> Confirmado
    Confirmado --> [*]
```

Cada registro tem três estados visíveis: **local**, **em sincronização** e **confirmado**. A interface
representa os três, para que o usuário nunca fique em dúvida sobre o que já foi salvo.

---

## Camada 3 — Transporte

Cliente HTTP único, responsável por anexar o token de autenticação, renovar a credencial de forma
transparente quando ela expira, aplicar retry com backoff em falhas transitórias e detectar mudança
de conectividade.

Toda comunicação usa HTTPS. Nenhuma requisição sai da aplicação sem passar por esta camada.

---

## Camada 4 — Borda / API

O ponto de entrada do backend. Roda como funções efêmeras distribuídas globalmente, sem servidores
persistentes.

### Roteamento por domínio

As rotas são agrupadas por área funcional — identidade, dados, IA, cobrança, notificações,
integrações e administração — com um módulo por responsabilidade em vez de um arquivo monolítico.
Isso mantém cada rota pequena, testável e substituível.

### Middleware

Aplicado de forma consistente e na ordem correta, antes de qualquer lógica:

| Middleware | Função |
|---|---|
| **CORS** | Restringe origens permitidas |
| **Limite de payload** | Rejeita requisições acima do teto antes de processar o corpo |
| **Autenticação** | Valida o token e resolve a identidade |
| **Rate limiting** | Limita por identidade e por rota, mais estrito nas rotas caras |
| **Validação** | Verifica o payload contra esquema antes de chegar ao domínio |

A ordem importa: rejeitar cedo o que é inválido evita gastar computação com requisições que já
falharam.

### Trabalho agendado

Jobs periódicos rodam no mesmo runtime, disparados por agendador da plataforma: envio de lembretes,
sincronizações incrementais, limpeza de dados frios, reengajamento e reconciliação de assinaturas.

Todo job é **idempotente** e **fatiado em lotes retomáveis** — ele pode ser reexecutado sem efeito
duplicado, e uma interrupção não perde o progresso.

---

## Camada 5 — Domínio

Onde vivem as regras do produto. Isolada tanto do protocolo HTTP quanto dos detalhes de persistência,
o que a torna a parte mais estável e testável do sistema.

### Orquestração de integrações

Cada serviço externo fica atrás de sua própria fronteira, que encapsula credenciais, formato de
requisição, limites do provedor e tratamento de erro.

```mermaid
graph LR
    D["Domínio"] --> F1["Fronteira<br/>IA"]
    D --> F2["Fronteira<br/>Pagamentos"]
    D --> F3["Fronteira<br/>Push"]
    D --> F4["Fronteira<br/>Mensageria"]
    D --> F5["Fronteira<br/>Calendários"]

    F1 --> P1["Provedores de IA"]
    F2 --> P2["Provedores de pagamento"]
    F3 --> P3["APNs · FCM"]
    F4 --> P4["WhatsApp · Twilio"]
    F5 --> P5["Google · Apple"]

    classDef d fill:#7c3f00,stroke:#f38020,color:#fff
    classDef f fill:#1f4d2e,stroke:#4caf50,color:#fff
    classDef p fill:#4a3010,stroke:#d99b4a,color:#fff
    class D d
    class F1,F2,F3,F4,F5 f
    class P1,P2,P3,P4,P5 p
```

O benefício prático: trocar um provedor, adicionar um segundo ou lidar com a indisponibilidade de um
deles é um trabalho contido em uma fronteira, sem tocar no domínio.

---

## Camada 6 — Acesso a Dados

Uma fronteira explícita entre o domínio e o banco. Nenhuma regra de negócio escreve consulta
diretamente.

Essa camada pareceu excesso de zelo no início do projeto e provou seu valor na migração da plataforma
de dados: foi possível manter dois destinos vivos simultaneamente durante a transição e remover o
antigo depois, sem reescrever o domínio.

---

## Camada 7 — Persistência

| Recurso | Uso |
|---|---|
| **Banco relacional distribuído** | Dados estruturados do produto, colocalizado com a computação |
| **Object storage** | Anexos e arquivos enviados pelo usuário |
| **Cache chave-valor** | Dados efêmeros, estados temporários e controle de limites |

Migrações são versionadas em SQL e aplicadas de forma controlada, com índices desenhados a partir dos
padrões de consulta reais.

---

## Padrões transversais

| Padrão | Onde se aplica |
|---|---|
| **Timeout** | Toda chamada externa tem prazo definido |
| **Retry com backoff** | Falhas transitórias, no cliente e no servidor |
| **Idempotência** | Operações sensíveis e todo consumo de webhook |
| **Degradação graciosa** | Falha de recurso secundário não derruba o principal |
| **Fallback de provedor** | Redirecionamento quando um provedor de IA falha |
| **Logging estruturado** | Sem dados pessoais, com alertas nos caminhos críticos |
| **Feature flags** | Ativação gradual de funcionalidades |

---

## Ver também

- [../SYSTEM_DESIGN.md](../SYSTEM_DESIGN.md) — por que cada decisão foi tomada
- [backend.md](backend.md) — detalhamento da camada serverless
- [mobile.md](mobile.md) — detalhamento da camada mobile
- [database.md](database.md) — modelo de dados conceitual
- [diagrams/](diagrams/) — diagramas adicionais
