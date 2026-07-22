# System Design — LodgeFlow

> Este documento explica **por que** o LodgeFlow foi construído do jeito que foi. Ele descreve o
> raciocínio de engenharia, as alternativas consideradas e o que foi abandonado no caminho.
>
> Nenhum código, endpoint, esquema real ou regra de negócio proprietária aparece aqui.

---

## 1. O problema de engenharia

O requisito de produto era simples de enunciar e desagradável de implementar:

> *Uma pessoa deve conseguir planejar o dia no celular no ônibus sem sinal, receber o lembrete como
> uma chamada mesmo com o telefone no silencioso, ajustar o plano por voz pela Alexa em casa, e
> encontrar tudo consistente ao abrir o desktop — sem nunca pensar em sincronização.*

Isso impõe quatro restrições que puxam em direções opostas:

| Restrição | Consequência técnica |
|---|---|
| **Cinco plataformas** | Reescrever cinco vezes é inviável para uma equipe pequena |
| **Rede opcional** | O servidor não pode ser pré-requisito para nenhuma operação de escrita |
| **Recursos genuinamente nativos** | CallKit, WidgetKit, EventKit e IAP não têm equivalente web |
| **Custo previsível na fase inicial** | Infraestrutura ociosa não pode custar caro |

O resto deste documento é a sequência de decisões que tenta satisfazer as quatro ao mesmo tempo.

---

## 2. Por que serverless

**Decisão:** todo o backend roda como funções efêmeras, sem servidores de aplicação persistentes.

### O que foi considerado

| Alternativa | Por que foi descartada |
|---|---|
| **VPS / servidor dedicado** | Custo fixo desde o dia zero, patching, escalonamento manual, ponto único de falha |
| **Kubernetes** | Complexidade operacional desproporcional ao tamanho do problema e da equipe |
| **Containers gerenciados** (ECS, Cloud Run) | Melhor, mas ainda exige gestão de imagens, escala e cold start relevante |
| **Serverless (adotado)** | Escala com o uso real, sem infraestrutura ociosa, sem plano de capacidade |

### O raciocínio

O padrão de carga de um app de produtividade é **fortemente irregular**: picos matinais quando as
pessoas planejam o dia, picos noturnos quando revisam, e vales longos de madrugada. Provisionar para
o pico significa pagar pelo vale.

Serverless inverte isso: o custo acompanha a demanda. Para um produto em fase de crescimento, isso
transforma um custo fixo de infraestrutura em um custo variável proporcional à receita.

### O que se paga por essa escolha

Serverless não é gratuito em complexidade. As limitações reais enfrentadas:

- **Sem estado entre requisições.** Nada de cache em memória de processo, nada de conexão persistente
  reaproveitada. Todo estado precisa morar explicitamente no banco, no KV ou no cliente.
- **Tempo de execução limitado.** Operações longas — uma sincronização completa de calendário, uma
  migração de dados — precisam ser **fatiadas em unidades retomáveis**, não executadas de uma vez.
  Isso obrigou a desenhar essas operações como incrementais desde o início, o que acabou sendo melhor
  de qualquer forma.
- **Sem processos em segundo plano.** Trabalho periódico precisa ser modelado como job agendado
  disparado externamente, e cada execução precisa ser idempotente porque pode reexecutar.

Essas restrições foram tratadas como **disciplina de design**, não como obstáculo. Um sistema que
sobrevive a elas é, por construção, mais fácil de escalar horizontalmente.

---

## 3. Por que Cloudflare Workers

**Decisão:** Cloudflare Workers como runtime, com D1, R2 e KV como camada de dados.

### O que foi considerado

| Alternativa | Avaliação |
|---|---|
| **AWS Lambda + API Gateway + RDS** | Maduro e completo, mas cold start perceptível, latência dependente de região e custo de saída de dados alto |
| **Vercel / Netlify Functions** | Ótima experiência de desenvolvimento, mas acoplado ao ciclo de deploy do frontend e menos controle sobre a camada de dados |
| **Supabase Edge Functions** | Boa integração com o banco, mas ecossistema de runtime mais restrito |
| **Cloudflare Workers (adotado)** | Isolates em vez de containers, execução global por padrão, dados no mesmo plano |

### O raciocínio

Três propriedades foram decisivas:

**1. Isolates, não containers.** Workers usam isolates V8 em vez de containers por invocação. A
diferença prática é a ausência de cold start relevante — o custo de subir um isolate é de
milissegundos, não de segundos. Para um app onde o usuário toca em algo e espera resposta imediata,
isso é a diferença entre parecer nativo e parecer web.

**2. Global por padrão, não por configuração.** Não existe "escolher a região". O código executa no
ponto de presença mais próximo de quem chamou. Para um produto com usuários em fusos diferentes, isso
elimina inteiramente a categoria de problema "escolher onde hospedar".

**3. Dados colocalizados com a computação.** D1 (SQLite distribuído), R2 (objetos) e KV (chave-valor)
vivem no mesmo plano de execução dos Workers. A consulta não atravessa a internet pública para chegar
ao banco — a maior fonte de latência em arquiteturas serverless tradicionais simplesmente não existe.

### O que se paga por essa escolha

- **Acoplamento de plataforma.** Migrar para outro provedor exigiria retrabalho real na camada de
  dados. A mitigação adotada foi manter a **lógica de domínio separada dos bindings da plataforma** —
  o acesso a dados fica atrás de uma camada própria, então trocar o motor abaixo é um trabalho
  delimitado, não uma reescrita.
- **D1 não é Postgres.** Não há extensões, tipos avançados nem o ecossistema maduro do Postgres. Isso
  foi aceito conscientemente: o modelo de dados do produto é predominantemente relacional simples, e
  SQLite dá conta. Se um dia exigir mais, a fronteira de acesso a dados existe justamente para isso.
- **Ecossistema mais novo.** Menos bibliotecas prontas, mais código próprio. Em troca, menos
  dependências carregadas e um bundle pequeno.

### Por que Hono no topo

Workers expõem uma API baixa demais para escrever um produto inteiro. Hono foi escolhido por ser
projetado para runtimes de edge desde a origem — bundle mínimo, sem dependência de APIs do Node,
tipagem forte e middleware componível. Alternativas como Express carregam premissas de Node que não
existem nesse ambiente.

O roteamento é organizado por **domínio** (autenticação, dados, IA, cobrança, notificações,
integrações), com middleware transversal aplicado de forma consistente: autenticação, CORS, limite de
requisições e teto de payload.

---

## 4. Por que Capacitor

**Decisão:** núcleo React compartilhado, empacotado por Capacitor no mobile, com plugins nativos em
Swift onde é necessário.

### O que foi considerado

| Alternativa | Avaliação |
|---|---|
| **Nativo puro** (Swift + Kotlin) | Melhor resultado possível por plataforma, custo proibitivo: 3 bases de código com o desktop |
| **React Native** | Bom equilíbrio, mas exigiria reescrever a interface inteira e manter uma ponte de UI própria |
| **Flutter** | Excelente, mas descartaria todo o investimento em React e obrigaria a uma linguagem a mais |
| **PWA puro** | Sem CallKit, sem widgets, sem IAP, sem Live Activities. Elimina metade do produto |
| **Capacitor (adotado)** | Preserva a base React, mantém acesso total ao nativo pela camada de plugins |

### O raciocínio

A pergunta certa não é *"qual framework é melhor?"*, e sim *"onde a diferença entre web e nativo é
percebida pelo usuário?"*.

Na maior parte do LodgeFlow — listas, formulários, calendário, gráficos, chat — a resposta é: **em
lugar nenhum**. Uma interface web bem construída, com animações corretas e feedback tátil, é
indistinguível de nativa nesses contextos.

A diferença aparece em um conjunto pequeno e bem delimitado de recursos:

- Chamada em tela cheia que atravessa o modo silencioso (**CallKit / PushKit**)
- Widget na home screen (**WidgetKit**)
- Atividade ao vivo na Dynamic Island (**Live Activities**)
- Acesso ao calendário do sistema (**EventKit**)
- Comandos de voz do sistema (**App Intents / Siri**)
- Compras dentro do app (**StoreKit**)

Capacitor permite exatamente essa divisão: **web onde não importa, Swift onde importa.** Cada recurso
nativo é um plugin com uma superfície pequena e uma responsabilidade única, chamado pela camada React
por uma interface tipada.

O resultado prático é que uma funcionalidade de produto nova custa uma implementação, não cinco. Só
recursos genuinamente nativos exigem trabalho por plataforma — e eles são a minoria.

### O que se paga por essa escolha

- **Teto de performance.** Animações muito pesadas ou listas com milhares de itens exigem cuidado
  extra que nativo puro não exigiria. Na prática, virtualização e renderização controlada resolveram.
- **A ponte é um limite real.** Chamadas entre JavaScript e nativo têm custo. Plugins foram desenhados
  para serem **conversas curtas e raras**, não canais de tráfego contínuo.
- **Dependência do ciclo do Capacitor.** Cada versão maior do iOS pode exigir espera. Mitigado por
  escrever os plugins críticos internamente, em vez de depender de plugins de terceiros.

### Desktop: por que Electron

Mesma lógica, aplicada ao desktop. Electron reaproveita o mesmo núcleo React e entrega instaladores
assinados para macOS e Windows. Alternativas mais leves (Tauri) foram consideradas e seriam
tecnicamente superiores em consumo de memória, mas exigiriam Rust na cadeia de build e ofereciam
menos maturidade nos fluxos de assinatura e notarização — que são, na prática, a parte difícil de
distribuir um app de desktop.

---

## 5. Offline-first: a decisão mais cara e mais importante

**Decisão:** a rede é tratada como opcional. O aplicativo escreve localmente primeiro, sempre.

### O raciocínio

A tentação em um app conectado é assumir a rede e tratar sua ausência como erro. Isso produz uma
experiência que falha exatamente nos momentos em que o produto seria mais útil: no metrô, no elevador,
no avião, em área de cobertura ruim.

O modelo adotado inverte a premissa:

1. **A escrita é local e imediata.** A interface reflete a mudança antes de qualquer chamada de rede.
2. **A mutação entra em uma fila durável.** Ela sobrevive ao fechamento do app e à reinicialização.
3. **A fila é drenada quando há conectividade.** Em ordem, com retry e backoff.
4. **Conflitos são resolvidos de forma determinística.** Cada mutação carrega metadados de versão e
   origem, então dois dispositivos que alteraram o mesmo registro sem se verem convergem para o mesmo
   resultado — não para o resultado de quem falou por último por acaso.

### Por que isso é caro

Offline-first contamina todas as camadas. Não dá para adicionar depois:

- Identificadores precisam ser gerados no **cliente**, não pelo banco — senão não há como referenciar
  um registro criado offline.
- Toda operação precisa ser **idempotente**, porque a fila pode reenviar.
- A interface precisa representar três estados por registro (local, em sincronização, confirmado), não
  dois.
- Os testes precisam cobrir cenários de partição de rede, não só o caminho feliz.

Foi assumido desde o primeiro dia justamente por isso — é o tipo de decisão que não se retrofita.

---

## 6. Como o produto pensa em escala

Escalabilidade aqui não significa "suportar um milhão de usuários amanhã". Significa **não ter um
gargalo estrutural** que exija reescrita quando o crescimento vier.

### Onde a escala já é resolvida pela arquitetura

| Dimensão | Como escala |
|---|---|
| **Requisições HTTP** | Isolates são criados sob demanda; não há pool de conexões a esgotar |
| **Distribuição geográfica** | Execução na edge, sem escolha de região a revisar |
| **Armazenamento de arquivos** | Object storage, sem limite prático e sem custo de saída |
| **Trabalho periódico** | Jobs agendados fatiados em lotes retomáveis, não varreduras totais |

### Onde a escala exige atenção deliberada

**1. Banco de dados.** É o limite mais próximo. As mitigações aplicadas:
- Índices desenhados a partir dos padrões de consulta reais, não por intuição
- Consultas sempre delimitadas por usuário e por janela de tempo, nunca varreduras globais
- Dados frios separados dos quentes, com rotina de limpeza
- Acesso isolado atrás de uma camada própria, para que particionar ou trocar o motor seja um trabalho
  delimitado

**2. Custo de IA.** Chamadas a modelos são a operação mais cara por unidade. As mitigações:
- Quota por usuário verificada **antes** da chamada, não depois
- Registro de consumo por operação, com métricas de custo e performance observáveis
- Roteamento por adequação: nem toda tarefa precisa do modelo mais capaz
- Inferência local no dispositivo onde a tarefa permite, tirando carga do servidor inteiramente
- Degradação graciosa quando um provedor falha, em vez de erro para o usuário

**3. Notificações em massa.** Disparar para toda a base ao mesmo tempo satura qualquer provedor. O
envio é feito em lotes, com controle de ritmo e retomada em caso de interrupção.

**4. Integrações externas.** Cada provedor tem os próprios limites de requisição. Cada um é isolado
atrás de sua fronteira, com timeout e retry próprios, de forma que a lentidão de um não se propague
para o resto do sistema.

### O princípio geral

> **Todo caminho quente é delimitado por usuário e por tempo.** Nenhuma operação de produto varre a
> base inteira. Trabalho em lote é sempre incremental e retomável.

Isso é o que permite que o mesmo desenho atenda mil ou cem mil usuários sem mudança estrutural.

---

## 7. Resiliência

Um sistema que integra mais de dez provedores externos vai ter um deles fora do ar em algum
momento. O desenho
assume isso.

| Padrão | Aplicação |
|---|---|
| **Timeout** | Toda chamada externa tem prazo. Nada espera indefinidamente |
| **Retry com backoff** | Falhas transitórias são reprocessadas com espaçamento crescente |
| **Idempotência** | Operações sensíveis carregam chave de idempotência; reentrega não duplica efeito |
| **Degradação graciosa** | A falha de um recurso secundário não derruba o principal |
| **Fallback entre provedores** | Quando um provedor de IA falha, o pedido é redirecionado |
| **Observabilidade** | Logging estruturado sem dados pessoais, com alertas nos caminhos críticos |

---

## 8. Decisões que foram revistas

Documentar o que deu errado é mais honesto — e mais útil — do que apresentar só o resultado final.

**Migração da camada de dados.** O produto começou sobre uma plataforma de banco gerenciada e migrou
para a camada de dados da própria edge. O motivo foi latência: cada consulta atravessava a internet
pública, e esse custo dominava o tempo de resposta. A migração foi feita de forma incremental, com
uma camada de compatibilidade mantendo os dois caminhos vivos durante a transição, e só então o
caminho antigo foi removido. **Lição:** a fronteira de acesso a dados que parecia excesso de zelo no
início foi exatamente o que tornou a migração viável.

**Fatiamento de operações longas.** As primeiras versões de sincronização de calendário tentavam
processar tudo de uma vez e esbarravam no limite de execução do runtime. Foram reescritas como
operações incrementais com ponto de retomada. **Lição:** a restrição da plataforma forçou um desenho
que também é melhor para o usuário — ele vê progresso em vez de esperar por um resultado binário.

**Plugins nativos próprios.** A primeira tentativa usou plugins de terceiros para recursos nativos
críticos. Vários ficaram desatualizados em relação ao ciclo do iOS ou não cobriam os casos de borda
necessários. Os plugins críticos passaram a ser escritos internamente. **Lição:** para o que é
central no produto, a dependência externa é um risco maior do que o custo de manter o próprio código.

---

## 9. Resumo das decisões

| Decisão | Escolha | Motivo principal |
|---|---|---|
| Modelo de execução | Serverless | Custo proporcional ao uso, sem infraestrutura ociosa |
| Plataforma de nuvem | Cloudflare Workers | Sem cold start, global por padrão, dados colocalizados |
| Framework HTTP | Hono | Projetado para edge, bundle mínimo, tipagem forte |
| Banco de dados | SQLite distribuído (D1) | Colocalizado com a computação; modelo relacional simples basta |
| Mobile | Capacitor + plugins Swift | Uma base de código, acesso nativo onde importa |
| Desktop | Electron | Reaproveita o mesmo núcleo; cadeia de assinatura madura |
| Frontend | React + TypeScript | Ecossistema, tipagem, reaproveitamento entre plataformas |
| Estado de servidor | TanStack Query | Cache, revalidação e atualização otimista prontos |
| Sincronização | Offline-first com fila durável | A rede é opcional, não pré-requisito |
| Pagamentos | Multi-provedor normalizado | Lojas exigem sistema próprio; o produto vê um estado só |
| IA | Multi-provedor com fallback | Resiliência e adequação de custo por tarefa |

---

<div align="center">
<sub>

Documento conceitual. Nenhum código-fonte, endpoint, credencial, prompt ou regra de negócio
proprietária do LodgeFlow é revelado aqui.

</sub>
</div>
