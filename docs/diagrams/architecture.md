# Diagrama — Arquitetura

> Diagramas em Mermaid, renderizados nativamente pelo GitHub. Representação **conceitual**.

---

## Visão geral em camadas

```mermaid
graph TB
    subgraph C["📱 Clientes"]
        direction LR
        C1["iOS"] ~~~ C2["Android"] ~~~ C3["macOS"] ~~~ C4["Windows"] ~~~ C5["Web"]
    end

    subgraph E["☁️ Edge Serverless"]
        direction TB
        E1["Roteamento por domínio"]
        E2["Middleware<br/>Auth · CORS · Rate limit · Payload"]
        E3["Regras de domínio"]
        E4["Orquestração de integrações"]
        E1 --> E2 --> E3 --> E4
    end

    subgraph D["🗄 Dados"]
        direction LR
        D1[("Relacional<br/>distribuído")] ~~~ D2[("Object<br/>storage")] ~~~ D3[("Chave-valor")]
    end

    subgraph X["🔌 Serviços externos"]
        direction LR
        X1["IA"] ~~~ X2["Pagamentos"] ~~~ X3["Push"] ~~~ X4["Mensageria"] ~~~ X5["Calendários"]
    end

    C -->|HTTPS · JWT| E
    E3 --> D
    E4 --> X
    X -.->|webhooks assinados| E1

    classDef cl fill:#1e3a5f,stroke:#4a90d9,color:#fff
    classDef ed fill:#7c3f00,stroke:#f38020,color:#fff
    classDef da fill:#1f4d2e,stroke:#4caf50,color:#fff
    classDef ex fill:#4a3010,stroke:#d99b4a,color:#fff
    class C1,C2,C3,C4,C5 cl
    class E1,E2,E3,E4 ed
    class D1,D2,D3 da
    class X1,X2,X3,X4,X5 ex
```

---

## Compartilhamento de código entre plataformas

```mermaid
graph TB
    CORE["<b>Núcleo compartilhado</b><br/>React · TypeScript · Tailwind<br/><i>Telas, estado, regras de interface</i>"]

    CORE --> WEB["<b>Web</b><br/>Navegador"]
    CORE --> CAP["<b>Capacitor</b>"]
    CORE --> ELE["<b>Electron</b>"]

    CAP --> IOS["<b>iOS</b><br/>+ plugins Swift"]
    CAP --> AND["<b>Android</b><br/>+ APIs da plataforma"]
    ELE --> MAC["<b>macOS</b>"]
    ELE --> WIN["<b>Windows</b>"]

    IOS -.-> N1["CallKit · WidgetKit<br/>Live Activities · EventKit<br/>App Intents · StoreKit"]
    AND -.-> N2["FCM · Play Billing"]

    classDef c fill:#1e3a5f,stroke:#4a90d9,color:#fff
    classDef w fill:#7c3f00,stroke:#f38020,color:#fff
    classDef n fill:#1f4d2e,stroke:#4caf50,color:#fff
    class CORE c
    class WEB,CAP,ELE,IOS,AND,MAC,WIN w
    class N1,N2 n
```

---

## Isolamento de integrações

```mermaid
graph LR
    DOM["<b>Domínio</b><br/><i>não conhece provedores</i>"]

    DOM --> F1["Fronteira · IA"]
    DOM --> F2["Fronteira · Pagamentos"]
    DOM --> F3["Fronteira · Push"]
    DOM --> F4["Fronteira · Mensageria"]
    DOM --> F5["Fronteira · Calendários"]

    F1 --> P1["OpenAI"] & P1b["Gemini"] & P1c["Local"]
    F2 --> P2["Stripe"] & P2b["Apple IAP"] & P2c["Play Billing"] & P2d["PIX"]
    F3 --> P3["APNs"] & P3b["FCM"]
    F4 --> P4["WhatsApp"] & P4b["Twilio"]
    F5 --> P5["Google"] & P5b["Apple"]

    classDef d fill:#7c3f00,stroke:#f38020,color:#fff
    classDef f fill:#1f4d2e,stroke:#4caf50,color:#fff
    classDef p fill:#4a3010,stroke:#d99b4a,color:#fff
    class DOM d
    class F1,F2,F3,F4,F5 f
    class P1,P1b,P1c,P2,P2b,P2c,P2d,P3,P3b,P4,P4b,P5,P5b p
```

Cada fronteira encapsula credenciais, formato de requisição, limites do provedor e tratamento de
erro. Trocar, adicionar ou perder um provedor é um trabalho contido — o domínio não é tocado.
