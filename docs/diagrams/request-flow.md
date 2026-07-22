# Diagrama — Fluxo de Requisição

> Diagramas em Mermaid. Representação **conceitual** — nenhum endpoint real é mostrado.

---

## Caminho de uma requisição autenticada

```mermaid
sequenceDiagram
    autonumber
    actor U as 👤 Usuário
    participant A as 📱 Aplicativo
    participant L as 💾 Store local
    participant W as ☁️ Worker
    participant DB as 🗄 Banco

    U->>A: Ação
    A->>L: Grava localmente
    A-->>U: UI atualiza (otimista)

    A->>W: Requisição + JWT

    rect rgb(40, 40, 55)
        Note over W: Middleware — rejeita cedo
        W->>W: CORS
        W->>W: Limite de payload
        W->>W: Verifica assinatura do token
        W->>W: Rate limit por identidade e rota
        W->>W: Validação de esquema
    end

    rect rgb(35, 50, 40)
        Note over W,DB: Domínio
        W->>DB: Verifica propriedade do registro
        DB-->>W: OK
        W->>DB: Executa a operação
        DB-->>W: Resultado
    end

    W-->>A: Resposta
    A->>L: Reconcilia estado local
    A-->>U: Estado confirmado
```

---

## Caminho sem conectividade

```mermaid
sequenceDiagram
    autonumber
    actor U as 👤 Usuário
    participant A as 📱 Aplicativo
    participant Q as 📥 Fila durável
    participant W as ☁️ Worker

    U->>A: Ação (sem rede)
    A->>Q: Enfileira mutação
    A-->>U: UI atualiza + indicador offline

    Note over A,W: ⏳ minutos ou horas depois

    A->>A: Conectividade detectada
    loop Drena a fila em ordem
        Q->>W: Envia mutação
        alt sucesso
            W-->>Q: Confirmado
            Q->>Q: Remove da fila
        else falha transitória
            W-->>Q: Erro
            Q->>Q: Retry com backoff
        else conflito de versão
            W->>W: Resolução determinística
            W-->>Q: Estado convergido
        end
    end

    A-->>U: Tudo sincronizado
```

---

## Operação de IA

```mermaid
flowchart TB
    A["Pedido do usuário"] --> B{"Consentimento<br/>registrado?"}
    B -->|não| C["Solicita consentimento"]
    B -->|sim| D{"Quota do plano<br/>disponível?"}
    D -->|não| E["Mensagem + caminho de upgrade"]
    D -->|sim| F["Monta contexto mínimo"]
    F --> G{"Roteamento<br/>por adequação"}
    G --> H["Provedor A"]
    G --> I["Provedor B"]
    G --> J["Inferência local"]
    H -->|falha| I
    I -->|falha| K["Degradação graciosa"]
    H & I & J --> L{"Resposta válida<br/>no esquema?"}
    L -->|não| K
    L -->|sim| M["Persiste + registra consumo"]
    M --> N["Usuário revisa a proposta"]
    N --> O["✅ Aplicado após confirmação"]

    classDef ok fill:#1f4d2e,stroke:#4caf50,color:#fff
    classDef warn fill:#5f4a1e,stroke:#d9a84a,color:#fff
    class O,M ok
    class C,E,K warn
```

---

## Consumo de webhook

```mermaid
flowchart LR
    A["Webhook recebido"] --> B{"Assinatura<br/>válida?"}
    B -->|não| C["❌ Descarta<br/><i>sem processar</i>"]
    B -->|sim| D{"Evento já<br/>processado?"}
    D -->|sim| E["✅ 200<br/><i>idempotente</i>"]
    D -->|não| F["Registra o evento"]
    F --> G["Aplica o efeito"]
    G --> H["✅ 200"]

    classDef no fill:#5f1e1e,stroke:#d94a4a,color:#fff
    classDef ok fill:#1f4d2e,stroke:#4caf50,color:#fff
    class C no
    class E,H ok
```

A verificação de assinatura vem **antes** de qualquer processamento. Um payload sem assinatura válida
nunca chega a tocar o domínio.

---

## Entrega de lembrete

```mermaid
flowchart TB
    A["⏰ Job agendado"] --> B["Carrega lote de lembretes<br/><i>delimitado por janela de tempo</i>"]
    B --> C{"Criticidade?"}
    C -->|crítico| D["📞 Push VoIP<br/>→ chamada em tela cheia"]
    C -->|normal| E{"Dispositivo<br/>alcançável?"}
    E -->|sim| F["🔔 Push remoto"]
    E -->|não| G["📴 Notificação local<br/><i>já agendada no aparelho</i>"]
    D & F & G --> H["Registra a entrega"]
    H --> I{"Lote concluído?"}
    I -->|não| B
    I -->|sim| J["✅ Fim"]

    classDef ok fill:#1f4d2e,stroke:#4caf50,color:#fff
    class J,D,F,G ok
```

O laço de lote é o que mantém o job dentro do limite de execução do runtime e permite retomada em
caso de interrupção.
