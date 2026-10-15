# Technical-Challenge-Distributed-Wagering-Processor

# Distributed Wagering Processor

Serviço para processar transações de apostas recebidas de provedores de jogos. O projeto foi desenvolvido como solução para o *Technical Challenge — Distributed Wagering Processor*, da Jungle Gaming, com foco nas partes mais delicadas do domínio: dinheiro, operações concorrentes, mensagens duplicadas e recuperação quando um processo falha.

A aplicação expõe uma API HTTP e também consome mensagens de uma fila. Nos dois caminhos, as operações financeiras passam pelo mesmo caso de uso. O PostgreSQL mantém o estado definitivo da carteira e do ledger; o SQS transporta comandos e eventos sem ser usado como substituto das garantias do banco.

## Tecnologias

- **Bun 1.x** e **TypeScript** em modo estrito
- **NestJS** para a API e os módulos da aplicação
- **PostgreSQL** como fonte persistente dos dados financeiros
- **MikroORM** para persistência e migrations
- **AWS SQS**, com **LocalStack** no ambiente local
- **Docker Compose** para subir os serviços de desenvolvimento
- **decimal.js** para cálculos monetários exatos
- **Prometheus** para métricas operacionais

## Como o processamento funciona

Uma transação é identificada pela chave de idempotência e pela identidade externa do provedor. Antes de aplicar seus efeitos, o serviço valida o payload, a carteira, a moeda e as regras da operação. O banco coordena o acesso concorrente à carteira e aplica as constraints necessárias para preservar os invariantes mesmo quando há mais de uma instância ativa.

Quando uma operação altera o saldo, a atualização da carteira e o lançamento correspondente no ledger são gravados na mesma transação SQL. A transação de aposta, o registro de inbox (para mensagens SQS) e os eventos de outbox também participam dessa transação. O publisher envia eventos apenas depois do commit.

A entrega por SQS é *at-least-once*: uma mensagem ou evento pode reaparecer. Por isso, a deduplicação é persistida no banco. Se o processo cair depois do commit e antes do ACK, a nova entrega encontra o resultado já registrado e não repete o efeito financeiro.

### Operações suportadas

| Operação | Efeito no saldo | Regra principal |
|---|---|---|
| `BET` | Débito | É rejeitada quando o saldo disponível é insuficiente. |
| `WIN` | Crédito | Pode ser enviada com uma referência opcional a uma aposta. |
| `LOSS` | Nenhum | Registra o resultado sem criar lançamento de saldo. |
| `REFUND` | Crédito | Reverte uma aposta processada, uma única vez. |
| `ROLLBACK` | Inverso da referência | Reverte uma transação processada, respeitando as regras do domínio. |

`REFUND` e `ROLLBACK` exigem referência. A referência precisa pertencer ao mesmo provedor, jogador, carteira, moeda e rodada; o valor da reversão deve ser igual ao da transação referenciada. Uma referência ainda não recebida ou ainda não processada pode deixar a operação em `PENDING_REFERENCE`, que será reavaliada pelo worker de referências pendentes. `OPENING` é uma operação interna: ela registra no ledger o saldo inicial positivo da carteira e não pode ser enviada pela API.

## Regras que guiam a implementação

- **Dinheiro não usa `number` como representação monetária.** Valores entram e saem como strings decimais com duas casas; a aritmética é feita com `decimal.js`, e o banco usa `numeric(20,2)`.
- **Idempotência é persistente.** A mesma chave e o mesmo payload de negócio devolvem o resultado original. A mesma chave com payload diferente é conflito (`409`). O hash SHA-256 usa um JSON canônico dos campos de negócio, com chaves ordenadas.
- **O banco é a última linha de defesa.** Locks por carteira, constraints, índices e triggers protegem contra saldo negativo, lançamentos inconsistentes, atualização ou remoção do ledger e alterações indevidas em transações já finalizadas.
- **Inbox e outbox são transacionais.** A inbox identifica mensagens já consumidas; a outbox guarda eventos que precisam ser publicados depois do commit. Publishers concorrentes reclamam registros com `FOR UPDATE SKIP LOCKED` e lease.
- **Referências fora de ordem têm tratamento próprio.** O worker tenta novamente com backoff e encerra com um código de falha específico quando o limite configurado é atingido.
- **A reconciliação não corrige silenciosamente.** Ela compara o saldo materializado com o saldo reconstruído do ledger e informa divergências.

As decisões e os trade-offs estão descritos em [`ARCHITECTURE.md`](./ARCHITECTURE.md). O relatório das verificações realizadas e das limitações conhecidas está em [`VALIDATION_REPORT.md`](./VALIDATION_REPORT.md).

## Requisitos para executar localmente

Instale:

- [Bun 1.x](https://bun.sh/)
- [Docker](https://www.docker.com/) com Docker Compose

Não é necessário instalar PostgreSQL ou LocalStack separadamente: o Compose sobe ambos para o ambiente de desenvolvimento.

## Subir a aplicação

Na raiz do projeto, execute:

```bash
cp .env.example .env
bun install
docker compose up --build -d
```

O Compose inicia PostgreSQL e LocalStack, cria as filas FIFO, aplica as migrations e sobe a API e os três workers. A API fica disponível em `http://localhost:3000`.

Confira o estado dos serviços e os logs:

```bash
docker compose ps
docker compose logs -f api worker outbox-worker pending-reference-worker
```

Para parar os serviços sem apagar os dados:

```bash
docker compose down
```

Para recriar o banco local do zero, removendo também os dados persistidos, use `docker compose down -v`. **Esse comando apaga o ledger e os demais dados locais.**

### Executar a API fora do Compose

Também é possível rodar a API no host, mantendo PostgreSQL e LocalStack em containers:

```bash
cp .env.example .env
bun install
docker compose up -d postgres localstack
bun run init:sqs
bun run migration:up
bun run start:dev
```

Nesse modo, os endereços do `.env.example` usam `localhost`. Para rodar a aplicação em um container, use os nomes dos serviços da rede Docker, como `postgres` e `localstack`.

## API HTTP

Os principais endpoints são:

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/wallets` | Cria uma carteira. Um saldo inicial positivo gera a transação interna `OPENING` e seu crédito no ledger. |
| `GET` | `/wallets/:walletId` | Consulta saldo, moeda e versão da carteira. |
| `GET` | `/wallets/:walletId/ledger?limit=50&cursor=...` | Lista lançamentos em ordem estável, com paginação por cursor. O limite aceito é de 1 a 100. |
| `POST` | `/wallets/:walletId/reconciliation` | Compara o saldo atual com o saldo reconstruído a partir do ledger. |
| `POST` | `/wagering/transactions` | Submete uma operação `BET`, `WIN`, `LOSS`, `REFUND` ou `ROLLBACK`. |
| `GET` | `/wagering/transactions/:transactionId` | Consulta uma transação pelo UUID interno. |
| `GET` | `/providers/:providerId/wagering/transactions/:externalTransactionId` | Consulta uma transação pela identificação do provedor. |
| `GET` | `/health/live` | Verifica se o processo está ativo. |
| `GET` | `/health/ready` | Verifica a disponibilidade do PostgreSQL e do SQS. |
| `GET` | `/metrics` | Expõe métricas no formato Prometheus. |

Os endpoints de carteira e apostas têm um ponto de extensão para autenticação de provedor. Por decisão de escopo, o `ProviderIdentityGuard` é um no-op; endpoints de health permanecem públicos. Antes de expor o serviço a tráfego real, substitua esse guard por autenticação integrada a um provedor OIDC externo.

### Criar uma carteira

O exemplo cria uma carteira com saldo inicial de `100.00 BRL`:

```bash
curl -X POST http://localhost:3000/wallets \
  -H 'content-type: application/json' \
  -d '{
    "playerId": "0192f28f-5dc0-7d58-bdb2-814ad6a0f4a1",
    "initialBalance": { "amount": "100.00", "currency": "BRL" }
  }'
```

Guarde o `id` retornado: ele será usado como `walletId` nas operações seguintes. O `playerId` precisa ser um UUID válido.

### Enviar uma aposta

Substitua `COLOQUE-UUID-DA-CARTEIRA` pelo ID retornado na criação da carteira. O `Idempotency-Key` é obrigatório e deve ser reutilizado ao repetir a mesma operação.

```bash
curl -X POST http://localhost:3000/wagering/transactions \
  -H 'content-type: application/json' \
  -H 'Idempotency-Key: provider-a:bet-1' \
  -d '{
    "providerId": "provider-a",
    "externalTransactionId": "bet-1",
    "playerId": "0192f28f-5dc0-7d58-bdb2-814ad6a0f4a1",
    "walletId": "COLOQUE-UUID-DA-CARTEIRA",
    "roundId": "round-1",
    "gameId": "game-1",
    "kind": "BET",
    "money": { "amount": "25.00", "currency": "BRL" }
  }'
```

A resposta inclui `transactionId`, `status`, `balance` e `idempotentReplay`. Reenviar a mesma chave com os mesmos dados retorna o resultado persistido; alterar os dados mantendo a chave resulta em conflito. Valores monetários devem ser strings com exatamente duas casas decimais, como `"25.00"`.

### Estados e respostas

- `PROCESSED`: operação finalizada. Em `LOSS`, o saldo não muda.
- `PENDING` ou `PENDING_REFERENCE`: processamento ainda não concluído; a API retorna `202`.
- `REJECTED`: a operação foi rejeitada por uma regra de negócio; a API retorna `422` e informa `failureCode` quando aplicável.
- Chave de idempotência reutilizada com outro payload: `409 Conflict`.
- Entrada inválida: `400 Bad Request`.
- Falha transitória de infraestrutura, como banco indisponível ou conflito de transação após as tentativas: `503 Service Unavailable`, para que o chamador possa tentar novamente usando a mesma chave.

## Filas e workers

O ambiente local usa quatro filas FIFO:

- `wager-transactions.fifo`: comandos de transação recebidos dos provedores;
- `wager-transactions-dlq.fifo`: comandos que excederam o limite de entrega configurado;
- `wager-events.fifo`: eventos publicados pela outbox;
- `wager-events-dlq.fifo`: eventos que excederam o limite de entrega configurado.

Os processos têm responsabilidades distintas:

- `worker`: consome comandos, usa o mesmo caso de uso da API e só confirma a mensagem depois do commit;
- `outbox-worker`: publica os eventos confirmados no banco e agenda novas tentativas em caso de falha transitória;
- `pending-reference-worker`: reavalia transações que aguardam uma referência.

A outbox garante que um evento não seja publicado antes do commit, mas não transforma a integração PostgreSQL/SQS em entrega exatamente uma vez. Um evento pode ser publicado novamente se o processo falhar no intervalo entre o envio e a marcação de publicação; consumidores desses eventos devem deduplicar pelo `eventId`.

## Testes

Os testes unitários cobrem `Money`, invariantes de carteira e ledger, estados das transações e regras de referência. Execute-os com:

```bash
bun run test:unit
```

Os testes de integração utilizam PostgreSQL e LocalStack reais em containers; não substituem os serviços por mocks. Para executá-los em ambiente local dedicado:

```bash
cp .env.example .env
bun install
docker compose up -d postgres localstack
bun run init:sqs
bun run migration:up
bun run test:integration
```

A suíte de integração inclui cenários para 50 envios concorrentes da mesma aposta, duas apostas de `80.00` disputando uma carteira com `100.00`, execução por três processos, redelivery após commit, falha transacional da outbox, referências fora de ordem, publishers concorrentes, DLQ e imutabilidade do ledger.

Para rodar a checagem de tipos e todos os testes disponíveis:

```bash
bun run test:all
```

**Use recursos descartáveis ao executar a suíte de integração.** Alguns cenários limpam dados de teste da outbox e manipulam filas. Não execute essa suíte contra um banco ou filas compartilhados com dados importantes.

> A existência dos testes no repositório não significa que eles já passaram no seu ambiente. O estado das validações feitas durante a preparação do projeto está registrado em [`VALIDATION_REPORT.md`](./VALIDATION_REPORT.md); a compilação e os testes com serviços reais devem ser executados antes de considerar a entrega validada.

## Estrutura do projeto

```text
src/
├── domain/                 # Dinheiro, carteira, ledger, estados e eventos
├── migrations/             # Schema e constraints versionados
├── modules/
│   ├── auth/               # Ponto de extensão para identidade do provedor
│   ├── health/             # Liveness e readiness
│   ├── messaging/          # SQS e publicação da outbox
│   ├── metrics/            # Métricas Prometheus
│   ├── persistence/        # Entidades MikroORM
│   ├── wallets/             # API e casos de uso de carteira
│   └── wagering/            # DTOs, API e processamento de apostas
├── scripts/                # Inicialização das filas e migrations
├── worker.ts               # Consumidor de comandos SQS
├── outbox-worker.ts        # Publicação de eventos
└── pending-reference-worker.ts

test/
├── unit/                   # Testes de domínio
└── integration/            # Testes com PostgreSQL e LocalStack
```

## Comandos úteis

| Comando | Para que serve |
|---|---|
| `bun run start:dev` | Inicia a API em modo de desenvolvimento. |
| `bun run build` | Compila o projeto para `dist/`. |
| `bun run typecheck` | Executa a checagem de tipos sem emitir arquivos. |
| `bun run init:sqs` | Cria ou configura as filas locais. |
| `bun run migration:up` | Aplica migrations pendentes. |
| `bun run migration:down` | Reverte a última migration; use com cuidado. |
| `bun run test:unit` | Executa os testes unitários. |
| `bun run test:integration` | Compila e executa os testes de integração. |
| `bun run test:all` | Executa a checagem de tipos e a suíte geral de testes. |

## Escopo e próximos passos antes de produção

O projeto foi preparado para demonstrar as garantias financeiras e de mensageria exigidas pelo desafio. Ele ainda não é uma plataforma financeira certificada nem deve ser exposto à internet sem adaptação. Antes de um uso real, é necessário integrar um IdP OIDC, gerenciar segredos e permissões AWS adequadamente, definir retenção e políticas de dados, configurar alertas, executar testes de carga e ensaios de backup e restauração.
