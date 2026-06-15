# Nuxt 4 com DDD e Clean Architecture

Arquitetura de referência para aplicações **Nuxt 4** organizadas por **bounded contexts**, utilizando **Domain-Driven Design**, **Clean Architecture**, **Nuxt Layers**, **TypeScript** e **Pinia**.

> Objetivo: manter as regras de negócio independentes de Vue, Nuxt, Pinia, HTTP e detalhes externos, permitindo evolução modular, testes isolados e divisão clara de responsabilidades.

![Arquitetura técnica do Nuxt 4 com DDD](./docs/architecture/nuxt4-ddd.png)

## Sumário

- [Visão geral](#visão-geral)
- [Princípios](#princípios)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Estrutura de um bounded context](#estrutura-de-um-bounded-context)
- [Responsabilidades das camadas](#responsabilidades-das-camadas)
- [Fluxos da arquitetura](#fluxos-da-arquitetura)
- [Instalação](#instalação)
- [Configuração principal do Nuxt](#configuração-principal-do-nuxt)
- [Configuração de cada layer](#configuração-de-cada-layer)
- [Aliases e imports](#aliases-e-imports)
- [Runtime config](#runtime-config)
- [Regras de dependência](#regras-de-dependência)
- [Testes](#testes)
- [Checklist para novos domínios](#checklist-para-novos-domínios)

## Visão geral

O projeto é dividido em duas dimensões:

1. **Domínios do negócio:** `auth`, `orders`, `budgets`, `payments` e outros bounded contexts.
2. **Camadas internas de cada domínio:** `domain`, `application`, `infrastructure` e `presentation`.

No Nuxt 4, as pastas dentro de `layers/` são registradas automaticamente. Cada layer deve possuir seu próprio `nuxt.config.ts`, mesmo quando não precisar de configuração adicional.

## Princípios

- Organização prioritária por domínio, e não apenas por tipo técnico.
- `domain` não depende de Vue, Nuxt, Pinia, `$fetch`, navegador ou banco de dados.
- `application` coordena casos de uso e depende do domínio.
- `infrastructure` implementa portas e contratos definidos pelas camadas internas.
- `presentation` contém pages, components, composables e stores.
- Páginas devem ser finas e atuar como composição da interface.
- Componentes devem receber dados por propriedades e emitir eventos.
- DTOs externos não devem circular diretamente pelo domínio.
- Mappers convertem dados da API em objetos de domínio.
- Código entre frontend e Nitro só deve ir para `shared/` quando for realmente agnóstico ao ambiente.

## Estrutura do projeto

```text
project/
├── app/                            # Shell principal da aplicação
│   ├── app.vue
│   ├── app.config.ts
│   ├── assets/
│   │   └── styles/
│   ├── components/
│   │   └── ui/                     # Design system compartilhado
│   ├── composables/                # Composables globais
│   ├── layouts/
│   ├── middleware/
│   ├── pages/                      # Rotas globais ou cross-domain
│   ├── plugins/
│   └── utils/
│
├── layers/                         # Bounded contexts locais
│   ├── auth/
│   │   └── nuxt.config.ts
│   ├── orders/
│   │   └── nuxt.config.ts
│   ├── budgets/
│   │   └── nuxt.config.ts
│   └── payments/
│       └── nuxt.config.ts
│
├── shared/                         # Código compartilhado entre app e Nitro
│   ├── contracts/
│   ├── types/
│   ├── constants/
│   └── validation/
│
├── server/                         # Nitro/BFF global
│   ├── api/
│   ├── middleware/
│   ├── plugins/
│   ├── routes/
│   └── utils/
│
├── public/                         # Arquivos servidos sem processamento
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .env.example
├── nuxt.config.ts
├── package.json
├── tsconfig.json
└── README.md
```

## Estrutura de um bounded context

Exemplo para `orders`:

```text
layers/orders/
├── nuxt.config.ts
├── app/
│   ├── domain/
│   │   ├── entities/
│   │   ├── value-objects/
│   │   ├── repositories/           # Interfaces/ports do domínio
│   │   ├── services/               # Domain services puros
│   │   ├── policies/
│   │   ├── events/
│   │   └── errors/
│   │
│   ├── application/
│   │   ├── use-cases/
│   │   ├── dto/
│   │   ├── ports/
│   │   └── errors/
│   │
│   ├── infrastructure/
│   │   ├── http/
│   │   ├── repositories/
│   │   ├── mappers/
│   │   ├── serializers/
│   │   ├── adapters/
│   │   └── factories/
│   │
│   ├── components/                 # Presentation
│   ├── composables/                # Presentation/orquestração de UI
│   ├── layouts/
│   ├── middleware/
│   ├── pages/
│   ├── plugins/                    # Composition root do domínio
│   ├── stores/                     # Pinia: estado de interface e sessão
│   └── utils/
│
└── server/
    └── api/                         # Endpoints Nitro do domínio
```

## Responsabilidades das camadas

### Domain

Contém as regras essenciais do negócio:

- entidades;
- value objects;
- invariantes;
- políticas;
- eventos de domínio;
- interfaces de repositório;
- erros de negócio.

O domínio deve ser TypeScript puro.

```ts
export class Order {
  constructor(
    public readonly id: string,
    private status: 'draft' | 'approved' | 'cancelled',
  ) {}

  cancel(): void {
    if (this.status === 'approved') {
      throw new Error('Approved orders cannot be cancelled')
    }

    this.status = 'cancelled'
  }
}
```

### Application

Coordena o fluxo da aplicação:

- casos de uso;
- DTOs de entrada e saída;
- portas para serviços externos;
- transações e orquestração;
- autorização contextual quando fizer parte do caso de uso.

```ts
export class CancelOrderUseCase {
  constructor(private readonly repository: OrderRepository) {}

  async execute(orderId: string): Promise<void> {
    const order = await this.repository.findById(orderId)

    if (!order) {
      throw new Error('Order not found')
    }

    order.cancel()
    await this.repository.save(order)
  }
}
```

### Infrastructure

Implementa detalhes técnicos:

- REST, GraphQL ou WebSocket;
- implementações de repositório;
- serializers e mappers;
- persistência local;
- observabilidade;
- adapters de serviços externos.

```ts
export class HttpOrderRepository implements OrderRepository {
  constructor(
    private readonly apiBase: string,
    private readonly fetcher: typeof $fetch = $fetch,
  ) {}

  async findById(id: string): Promise<Order | null> {
    const response = await this.fetcher<OrderResponse>(
      `${this.apiBase}/orders/${id}`,
    )

    return OrderMapper.toDomain(response)
  }
}
```

### Presentation

No Nuxt, esta camada é formada por:

```text
pages/
components/
composables/
stores/
layouts/
middleware/
```

Ela pode depender de `application`, mas não deve implementar regras críticas de negócio.

## Fluxos da arquitetura

### Fluxo de dependência

```text
Presentation ─────▶ Application ─────▶ Domain
       │                                      ▲
       └──────── Infrastructure ──────────────┘
                 implementa contratos
```

### Fluxo de execução

```text
Page
  └──▶ Composable ou Store
        └──▶ Use Case
              └──▶ Repository Port
                    └──▶ Repository Adapter
                          └──▶ API externa ou Nitro
```

## Instalação

Crie um projeto Nuxt 4:

```bash
pnpm dlx nuxi@latest init nuxt-ddd-app
cd nuxt-ddd-app
pnpm install
```

Instale Pinia:

```bash
pnpm add pinia @pinia/nuxt
```

Instale as dependências de validação de tipos:

```bash
pnpm add -D typescript vue-tsc
```

Crie as layers iniciais:

```bash
mkdir -p layers/{auth,orders,budgets,payments}/app
mkdir -p layers/{auth,orders,budgets,payments}/server

touch layers/auth/nuxt.config.ts
touch layers/orders/nuxt.config.ts
touch layers/budgets/nuxt.config.ts
touch layers/payments/nuxt.config.ts
```

## Configuração principal do Nuxt

Arquivo `nuxt.config.ts`:

```ts
import { fileURLToPath } from 'node:url'

export default defineNuxtConfig({
  compatibilityDate: '2026-06-15',

  devtools: {
    enabled: true,
  },

  ssr: true,

  modules: [
    '@pinia/nuxt',
  ],

  css: [
    '~/assets/styles/main.css',
  ],

  runtimeConfig: {
    // Disponível apenas no servidor.
    apiToken: '',

    public: {
      // Disponível no servidor e no navegador.
      apiBase: '/api',
      appName: 'Nuxt DDD',
    },
  },

  alias: {
    // Aliases globais somente para módulos realmente transversais.
    '#core': fileURLToPath(new URL('./app/core', import.meta.url)),
  },

  pinia: {
    storesDirs: [
      'app/stores/**',
      'layers/*/app/stores/**',
    ],
  },

  typescript: {
    strict: true,
    typeCheck: true,
    tsConfig: {
      compilerOptions: {
        noUncheckedIndexedAccess: true,
        exactOptionalPropertyTypes: true,
      },
    },
  },

  app: {
    head: {
      htmlAttrs: {
        lang: 'pt-BR',
      },
      titleTemplate: '%s · Nuxt DDD',
      meta: [
        {
          name: 'viewport',
          content: 'width=device-width, initial-scale=1',
        },
      ],
    },
  },
})
```

### Por que não configurar `extends` para as layers locais?

Layers colocadas diretamente em `layers/` são detectadas automaticamente pelo Nuxt. Use `extends` somente quando precisar carregar uma layer externa, uma layer publicada em pacote, uma layer remota ou uma pasta fora de `layers/`.

Exemplo para uma layer externa:

```ts
export default defineNuxtConfig({
  extends: [
    '../company-design-system',
  ],
})
```

## Configuração de cada layer

Cada bounded context precisa de um `nuxt.config.ts`.

Exemplo `layers/orders/nuxt.config.ts`:

```ts
export default defineNuxtConfig({
  $meta: {
    name: 'orders',
  },
})
```

A configuração pode permanecer vazia:

```ts
export default defineNuxtConfig({})
```

O Nuxt mescla a configuração da layer com a configuração principal.

## Aliases e imports

O Nuxt cria aliases automáticos para layers locais:

```ts
import { Order } from '#layers/orders/domain/entities/Order'
import type { OrderRepository } from '#layers/orders/domain/repositories/OrderRepository'
```

Aliases úteis já fornecidos pelo Nuxt:

```text
~/           → app/
@/           → app/
~~/          → raiz do projeto
@@/          → raiz do projeto
#shared/     → shared/
#server/     → server/
#layers/...  → srcDir da layer correspondente
```

### Regra recomendada

Não configure auto-import global para `domain`, `application` ou `infrastructure`. Imports explícitos deixam as dependências visíveis e reduzem acoplamento acidental.

## Runtime config

Arquivo `.env`:

```dotenv
NUXT_API_TOKEN=server-only-secret
NUXT_PUBLIC_API_BASE=https://api.example.com
NUXT_PUBLIC_APP_NAME=Nuxt DDD
```

Uso no servidor ou em um plugin:

```ts
const config = useRuntimeConfig()

console.log(config.apiToken)
console.log(config.public.apiBase)
```

Somente propriedades dentro de `runtimeConfig.public` podem ser enviadas ao navegador.

## Regras de dependência

### Permitido

```text
presentation    → application
application     → domain
infrastructure  → domain
infrastructure  → application
server          → application
```

### Proibido

```text
domain          → Vue
 domain         → Nuxt
 domain         → Pinia
 domain         → $fetch
 domain         → localStorage
 application    → componentes Vue
 application    → stores de interface
```

### Comunicação entre domínios

Evite importar detalhes internos de outra layer:

```ts
// Evitar
import { PaymentInternalService } from '#layers/payments/infrastructure/internal'
```

Prefira contratos públicos, eventos, facades ou use cases expostos pelo bounded context:

```text
orders/application/ports/PaymentGateway.ts
payments/infrastructure/adapters/OrderPaymentGateway.ts
```

## App shell

Arquivo `app/app.vue`:

```vue
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

O shell principal deve concentrar apenas recursos transversais:

- layout global;
- design system;
- providers globais;
- tratamento global de erro;
- telemetria;
- autenticação transversal;
- navegação principal.

## Pinia

Use Pinia para estado de interface, cache temporário, filtros, paginação, sessão e coordenação de tela.

Evite colocar no store:

- invariantes do domínio;
- validações de negócio críticas;
- serialização da API;
- regras de cálculo reutilizáveis;
- acesso HTTP espalhado.

Exemplo:

```ts
export const useOrdersStore = defineStore('orders', () => {
  const selectedOrderId = ref<string | null>(null)
  const filters = reactive({
    status: null as string | null,
    search: '',
  })
  const isLoading = ref(false)

  return {
    selectedOrderId,
    filters,
    isLoading,
  }
})
```

## Server/Nitro

Rotas globais ficam em `server/api`:

```text
server/api/health.get.ts
server/api/session.get.ts
```

Rotas pertencentes a um domínio podem ficar dentro da própria layer:

```text
layers/orders/server/api/orders/index.get.ts
layers/orders/server/api/orders/[id].get.ts
```

O Nitro pode atuar como BFF para:

- proteger credenciais privadas;
- agregar várias APIs;
- normalizar respostas;
- aplicar cache;
- converter erros externos;
- manter contratos consistentes para o frontend.

## Testes

Estrutura recomendada:

```text
tests/
├── unit/
│   ├── domain/
│   └── application/
├── integration/
│   ├── repositories/
│   └── server/
└── e2e/
```

Prioridades:

1. Testar entidades, value objects e políticas sem montar o Nuxt.
2. Testar casos de uso com repositories fake ou in-memory.
3. Testar adapters HTTP separadamente.
4. Testar pages e componentes apenas para comportamento de interface.
5. Usar E2E para fluxos críticos completos.

## Scripts recomendados

```json
{
  "scripts": {
    "dev": "nuxt dev",
    "build": "nuxt build",
    "preview": "nuxt preview",
    "generate": "nuxt generate",
    "typecheck": "nuxt typecheck",
    "postinstall": "nuxt prepare"
  }
}
```

## Checklist para novos domínios

- [ ] Criar `layers/<dominio>/nuxt.config.ts`.
- [ ] Definir entidades e value objects.
- [ ] Criar contratos de repositório no domínio.
- [ ] Implementar casos de uso na application.
- [ ] Criar DTOs de entrada e saída.
- [ ] Implementar adapters e repositories na infrastructure.
- [ ] Criar mapper entre DTO externo e domínio.
- [ ] Criar composition root em plugin, factory ou composable específico.
- [ ] Criar pages, components e stores da apresentação.
- [ ] Adicionar testes unitários do domínio e dos casos de uso.
- [ ] Evitar dependência direta entre bounded contexts.

## Decisões arquiteturais

### Use Nuxt Layers quando

- o projeto possui múltiplos domínios relevantes;
- diferentes times trabalham em áreas separadas;
- cada módulo possui pages, components, stores e endpoints próprios;
- existe possibilidade de publicar ou reutilizar uma parte da aplicação;
- o projeto tende a crescer por vários anos.

### Não aplique DDD completo quando

- a aplicação é pequena e essencialmente CRUD;
- não existem regras de negócio significativas;
- há apenas poucas páginas e um único fluxo;
- a complexidade da estrutura supera a complexidade do problema.

Nesse caso, use uma organização mais simples por features e evolua gradualmente.

## Referências oficiais

- Nuxt 4 — Directory Structure: https://nuxt.com/docs/4.x/directory-structure
- Nuxt 4 — Layers Directory: https://nuxt.com/docs/4.x/directory-structure/layers
- Nuxt 4 — Authoring Layers: https://nuxt.com/docs/4.x/guide/going-further/layers
- Nuxt 4 — Configuration Reference: https://nuxt.com/docs/4.x/api/nuxt-config
- Nuxt 4 — Runtime Configuration: https://nuxt.com/docs/4.x/getting-started/configuration
- Pinia with Nuxt: https://pinia.vuejs.org/ssr/nuxt.html
