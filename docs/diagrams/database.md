# Diagrama — Modelo de Dados

> ⚠️ **Representação ilustrativa.** Não corresponde ao esquema real do LodgeFlow. Nomes, campos e
> relacionamentos foram generalizados de propósito. Consulte [../database.md](../database.md) para os
> princípios de modelagem.

---

## Entidades e relacionamentos (simplificado)

```mermaid
erDiagram
    USUARIO ||--o{ PROJETO : possui
    USUARIO ||--o{ TAREFA : possui
    USUARIO ||--o{ TRANSACAO : registra
    USUARIO ||--|| ASSINATURA : tem
    USUARIO ||--o{ DISPOSITIVO : registra
    USUARIO ||--o{ INTEGRACAO : conecta
    USUARIO ||--o{ USO_IA : gera

    PROJETO ||--o{ TAREFA : agrupa
    TAREFA ||--o{ SUBTAREFA : contem
    TAREFA ||--o{ COMENTARIO : recebe
    TAREFA ||--o{ ANEXO : referencia
    TAREFA ||--o{ LEMBRETE : dispara
    TRANSACAO }o--|| CATEGORIA : classifica
```

---

## Agrupamento por domínio

```mermaid
graph TB
    subgraph ID["🔐 Identidade"]
        U["Usuário"]
        S["Assinatura"]
        DV["Dispositivo"]
    end

    subgraph PL["📋 Planejamento"]
        P["Projeto"]
        T["Tarefa"]
        ST["Subtarefa"]
        CM["Comentário"]
        AX["Anexo"]
        LB["Lembrete"]
    end

    subgraph FI["💰 Finanças"]
        TR["Transação"]
        CT["Categoria"]
    end

    subgraph IN["🔌 Plataforma"]
        IG["Integração"]
        AI["Uso de IA"]
        PF["Preferência"]
    end

    U --> P & T & TR & S & DV & IG & AI & PF
    P --> T
    T --> ST & CM & AX & LB
    TR --> CT

    classDef i fill:#1e3a5f,stroke:#4a90d9,color:#fff
    classDef p fill:#1f4d2e,stroke:#4caf50,color:#fff
    classDef f fill:#7c3f00,stroke:#f38020,color:#fff
    classDef n fill:#3d2452,stroke:#8e75b2,color:#fff
    class U,S,DV i
    class P,T,ST,CM,AX,LB p
    class TR,CT f
    class IG,AI,PF n
```

Note que **toda entidade tem caminho até o usuário**. Isso não é acidente de modelagem — é a base
simultânea do modelo de autorização (a propriedade é sempre verificável) e do modelo de performance
(toda consulta pode ser delimitada por usuário).

---

## Ciclo de vida de um registro sincronizável

```mermaid
stateDiagram-v2
    [*] --> Local: criado no dispositivo<br/>(id gerado no cliente)
    Local --> Pendente: mutação enfileirada
    Pendente --> Enviando: há conectividade
    Enviando --> Confirmado: aceito pelo servidor
    Enviando --> Pendente: falha transitória (backoff)
    Enviando --> Conflito: divergência de versão
    Conflito --> Confirmado: resolução determinística
    Confirmado --> Alterado: editado novamente
    Alterado --> Pendente
    Confirmado --> MarcadoRemovido: excluído
    MarcadoRemovido --> [*]: expurgo pela rotina de limpeza
```

Dois detalhes explicam decisões descritas em [../database.md](../database.md):

- O identificador nasce **no dispositivo**, senão um registro criado offline não poderia ser
  referenciado por outros registros locais.
- A exclusão é **lógica antes de física**, senão um dispositivo offline não distinguiria "foi
  apagado em outro aparelho" de "ainda não chegou até mim" — e recriaria o registro.

---

## Armazenamento por tipo de dado

```mermaid
graph LR
    A["Dado a persistir"] --> B{"Tem relação com<br/>outros dados?"}
    B -->|sim| C[("🗄 Banco relacional")]
    B -->|não| D{"É binário<br/>grande?"}
    D -->|sim| E[("📦 Object storage")]
    D -->|não| F{"Tem tempo de<br/>vida definido?"}
    F -->|sim| G[("⚡ Chave-valor")]
    F -->|não| C

    classDef s fill:#1f4d2e,stroke:#4caf50,color:#fff
    class C,E,G s
```
