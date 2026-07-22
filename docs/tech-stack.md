# Stack Tecnológica

> Documento conceitual. Lista as tecnologias utilizadas e o motivo de cada escolha. Não contém
> configurações de produção, versões exatas de dependências privadas nem detalhes de infraestrutura.

---

## Linguagens

| Linguagem | Onde | Por quê |
|---|---|---|
| **TypeScript** | Frontend, backend, plugins, tooling | Uma linguagem em toda a stack; tipagem que pega erro antes do runtime |
| **Swift** | Plugins nativos iOS, widgets, Live Activities | Único caminho para CallKit, WidgetKit, EventKit e StoreKit |
| **SQL** | Migrations e consultas | Explícito e versionável; sem camada mágica entre o código e o banco |

A escolha de TypeScript em toda a stack não é preferência estética. Ela permite compartilhar tipos
entre cliente e servidor, o que transforma incompatibilidade de contrato de API em erro de compilação
em vez de erro em produção.

---

## Frontend

| Tecnologia | Papel | Por quê |
|---|---|---|
| **React 18** | Biblioteca de UI | Ecossistema maduro; o mesmo núcleo serve web, mobile e desktop |
| **Vite** | Build e dev server | Build rápido, HMR instantâneo, saída otimizada |
| **SWC** | Transpilação | Substancialmente mais rápido que Babel no ciclo de desenvolvimento |
| **Tailwind CSS** | Estilização | Consistência visual sem CSS global crescendo sem controle |
| **Radix UI** | Primitivas de componente | Acessibilidade correta por padrão — foco, ARIA, teclado |
| **shadcn/ui** | Camada de componentes | Componentes que ficam no projeto, não em `node_modules` |
| **TanStack Query** | Estado de servidor | Cache, revalidação e atualização otimista prontos |
| **React Router** | Navegação | Roteamento declarativo compartilhado entre plataformas |
| **React Hook Form** | Formulários | Performance por renderização mínima em formulários grandes |
| **Zod** | Validação de esquema | O mesmo esquema valida no cliente e no servidor |
| **Recharts** | Gráficos | Composição em React; adequado às visualizações financeiras |
| **dnd-kit** | Arrastar e soltar | Acessível e performático no planner |
| **date-fns** | Datas | Modular e imutável; entra no bundle só o que é usado |
| **Lucide** | Ícones | Conjunto consistente, tree-shakeable |
| **Sonner** | Notificações em tela | Feedback não intrusivo |
| **next-themes** | Temas | Claro, escuro e preferência do sistema |

### Sobre shadcn/ui

Componentes copiados para dentro do projeto em vez de instalados como dependência. O trade-off é
explícito: perde-se a atualização automática, ganha-se controle total sobre comportamento e
aparência, sem lutar contra a API de uma biblioteca de terceiros. Para um produto com identidade
visual própria, esse é o lado certo do trade-off.

---

## Backend

| Tecnologia | Papel | Por quê |
|---|---|---|
| **Cloudflare Workers** | Runtime serverless | Isolates sem cold start relevante; global por padrão |
| **Hono** | Framework HTTP | Projetado para edge; bundle mínimo, tipado, componível |
| **Wrangler** | Tooling e deploy | Desenvolvimento local fiel ao runtime de produção |
| **jose** | JWT | Implementação sobre Web Crypto, compatível com edge |
| **bcrypt** | Hash de senha | Custo adaptativo contra força bruta |
| **Cron Triggers** | Jobs agendados | Trabalho periódico sem processo de longa duração |

---

## Dados

| Tecnologia | Papel | Por quê |
|---|---|---|
| **Cloudflare D1** | Banco relacional | SQLite distribuído, colocalizado com a computação |
| **Cloudflare R2** | Object storage | Anexos e arquivos, sem custo de saída de dados |
| **Cloudflare KV** | Chave-valor | Cache, estados efêmeros e controle de limites |
| **Migrations SQL** | Evolução do esquema | Versionadas, ordenadas e revisáveis |

---

## Inteligência Artificial

| Tecnologia | Papel | Por quê |
|---|---|---|
| **OpenAI** | Modelos de linguagem | Qualidade em raciocínio e saída estruturada |
| **Google Gemini** | Modelos de linguagem | Capacidades multimodais e alternativa de custo |
| **Transformers.js** | Inferência no dispositivo | Tarefas simples sem sair do aparelho nem custar servidor |

Detalhes da orquestração em [ai.md](ai.md).

---

## Mobile

| Tecnologia | Papel |
|---|---|
| **Capacitor 8** | Empacotamento nativo e acesso a recursos de dispositivo |
| **Swift** | Plugins nativos iOS |
| **CallKit / PushKit** | Lembretes críticos como chamada nativa em tela cheia |
| **WidgetKit** | Widgets na tela inicial |
| **Live Activities** | Sessão ativa na Dynamic Island e tela de bloqueio |
| **EventKit** | Sincronização com o calendário do sistema |
| **App Intents** | Comandos de voz pela Siri e atalhos |
| **StoreKit** | Compras dentro do aplicativo |
| **Google Play Billing** | Assinaturas no Android |

Plugins Capacitor em uso: aplicativo, navegador, háptico, teclado, notificações locais, rede,
notificações push, tela de abertura, barra de status, tarefas em segundo plano e autenticação social.

Detalhes em [mobile.md](mobile.md).

---

## Desktop

| Tecnologia | Papel | Por quê |
|---|---|---|
| **Electron** | Invólucro desktop | Reaproveita o núcleo React; cadeia madura de assinatura |
| **electron-builder** | Empacotamento | Instaladores assinados para macOS e Windows |

---

## Notificações e mensageria

| Tecnologia | Papel |
|---|---|
| **APNs** | Push no ecossistema Apple, incluindo canal VoIP |
| **Firebase Cloud Messaging** | Push no Android |
| **WhatsApp Business** | Lembretes e interação por mensagem |
| **Twilio** | Infraestrutura de comunicação |

---

## Pagamentos

| Tecnologia | Onde | Por quê |
|---|---|---|
| **Stripe** | Web e desktop | Assinaturas, portal do cliente e webhooks maduros |
| **Apple In-App Purchase** | iOS | Exigência da App Store para bens digitais |
| **Google Play Billing** | Android | Exigência da Google Play |
| **PIX** | Brasil | Meio de pagamento dominante no mercado principal |

Os quatro convergem para um único estado interno de acesso, resolvido no servidor.

---

## Integrações

| Serviço | Uso |
|---|---|
| **Google Calendar** | Sincronização bidirecional |
| **Apple Calendar (EventKit)** | Sincronização nativa no iOS |
| **Amazon Alexa** | Skill própria com account linking e notificações |
| **Siri / App Intents** | Comandos de voz nativos |
| **Google Sign-In / Sign in with Apple** | Autenticação social nativa |

---

## Cloud e infraestrutura

| Tecnologia | Papel |
|---|---|
| **Cloudflare Workers** | Execução do backend na edge |
| **Cloudflare Pages** | Hospedagem da aplicação web e do site |
| **Cloudflare D1 / R2 / KV** | Camada de dados |
| **Cron Triggers** | Agendamento de trabalho periódico |

---

## Qualidade e DevOps

| Tecnologia | Papel |
|---|---|
| **TypeScript strict** | Tipagem rigorosa em toda a base de código |
| **ESLint** | Padrões de código consistentes |
| **Vitest** | Testes unitários e de integração |
| **Wrangler** | Deploy e observação de logs em tempo real |
| **Migrations versionadas** | Evolução controlada do esquema |
| **Logging estruturado** | Diagnóstico sem exposição de dados pessoais |
| **Feature flags** | Ativação gradual e desativação rápida sem novo deploy |

---

## O que foi deliberadamente evitado

Decisões negativas explicam a stack tanto quanto as positivas:

| Não adotado | Motivo |
|---|---|
| **Kubernetes / containers** | Complexidade operacional desproporcional ao problema |
| **ORM pesado** | SQL explícito é mais previsível e evita abstração vazando em consulta quente |
| **Redux / estado global amplo** | Cache de servidor + estado local cobrem o caso sem cerimônia |
| **CSS-in-JS em runtime** | Custo de execução desnecessário; Tailwind resolve em build |
| **Micro-serviços** | Complexidade distribuída sem benefício nesta escala |
| **Múltiplas bases nativas** | Custo de manutenção incompatível com uma equipe pequena |
| **Dependência de um único provedor de IA** | Risco de disponibilidade e de preço |

---

## Ver também

- [../SYSTEM_DESIGN.md](../SYSTEM_DESIGN.md) — o raciocínio por trás das escolhas principais
- [architecture.md](architecture.md) — como as peças se encaixam
