# API — `audit-service`

Endpoints do `audit-service` (porta 8081) e o contrato da mensagem que ele consome. A aplicação principal (`permission-service`) publica cada validação de permissão na [fila `audit.events`](#mensageria--fila-auditevents-rabbitmq) e repassa ao `GET` daqui, por OpenFeign, cada consulta ao [`GET /audit-events` dela](https://github.com/Permission-SaaS/permission_saas_api/blob/main/docs/API.md#get-audit-events). O `POST` foi o caminho da gravação até 03/10/2026 e continua disponível para uso direto.

Os dois endpoints podem ser exercitados direto, pelo Swagger UI em `http://localhost:8081/swagger-ui/index.html` ou pela coleção Postman do [repositório guarda-chuva](https://github.com/Permission-SaaS/permission_saas) (pasta `3. permission_saas_audit (8081)`). O serviço não tem autenticação: é chamado pela rede interna dos serviços, não por clientes.

## Formato padrão de erro

Todo erro é tratado por `api/GlobalExceptionHandler` e devolvido no mesmo formato (`ErrorResponse`) da aplicação principal:

```json
{ "status": 400, "error": "Bad Request", "message": "routePath: must start with /", "timestamp": "..." }
```

| Situação | Status |
|---|---|
| Falha de validação `@Valid` no corpo ou nos filtros, categoria `InvalidDataException` (período inválido) | `400 Bad Request` |
| Corpo ilegível: JSON malformado, UUID ou data em formato inválido (mensagem fixa `"Malformed request body"`) | `400 Bad Request` |
| Qualquer erro não mapeado | `500 Internal Server Error` |

---

### `GET /audit-events`

Lista a trilha de auditoria, do evento mais recente para o mais antigo.

**Query params** (todos opcionais, `SearchAuditEventsRequest`): `type` (`PERMISSION_CHECK` ou `PROJECT_LIFECYCLE`, sem diferenciar maiúsculas), `projectId` (UUID), `onlyDenied` (`true` devolve só as validações negadas), `from` e `to` (ISO-8601 com fuso, ex.: `2026-09-01T00:00:00Z`; os dois limites entram no resultado). Um limite sozinho vale como "a partir de" ou "até". `onlyDenied=true` com `type=PROJECT_LIFECYCLE` devolve lista vazia — evento de ciclo de vida nunca é negado.

Os filtros são aplicados no banco, por consulta JPQL (ADR-009, no [log de ADRs](https://github.com/Permission-SaaS/permission_saas/blob/main/docs/ARCHITECTURE.md)).

**Response** `200 OK` — array de `AuditEventResponse`:
```json
[
  {
    "id": "e24dd619-da77-46c7-b8ab-956d90d0ad71",
    "type": "PERMISSION_CHECK",
    "projectId": "d13b3f47-8988-46bf-9061-f66dcecb3d04",
    "occurredAt": "2026-10-01T01:14:31.175821Z",
    "description": "NEGADO GET /passo-9 para o cargo 'ADMIN' — invalid or inactive api key (762.892476 ms)"
  }
]
```

**Um DTO para toda a hierarquia:** `AuditEvent` é abstrata e tem duas subclasses (`PermissionCheckEvent`, `ProjectLifecycleEvent`). O que as distingue sai pronto em `type` e `description`, ambos polimórficos (`type()` e `describe()`), então a API expõe a herança sem precisar de um DTO por subclasse.

**Erros:** `400 Bad Request` se `type` não for um dos valores conhecidos, se `from` estiver no futuro ou se `from` for posterior a `to`.

```bash
curl "http://localhost:8081/audit-events?onlyDenied=true&from=2026-09-01T00:00:00Z"
```

---

### `POST /audit-events/permission-checks`

Registra uma validação de permissão na trilha. O caminho é específico do tipo de evento porque o corpo só serve para validações de permissão; `GET /audit-events` continua listando todos os tipos juntos.

**Request** (`RegisterPermissionCheckRequest`):
```json
{
  "projectId": "11111111-1111-1111-1111-111111111111",
  "occurredAt": "2026-09-28T20:00:00Z",
  "routePath": "/produtos",
  "httpMethod": "GET",
  "roleName": "ADMIN",
  "granted": false,
  "reason": "invalid or inactive api key",
  "durationMs": 3.5
}
```

| Campo | Regra |
|---|---|
| `projectId` | obrigatório, UUID |
| `occurredAt` | obrigatório, ISO-8601 com fuso. **Sem** `@PastOrPresent`: a data é gerada por outro serviço, com outro relógio, e uma diferença de milissegundos entre eles recusaria eventos legítimos |
| `routePath` | obrigatório, começa com `/`, até 255 caracteres |
| `httpMethod` | obrigatório, `GET`/`POST`/`PUT`/`PATCH`/`DELETE` sem diferenciar maiúsculas; gravado em maiúsculas |
| `roleName` | obrigatório, até 80 caracteres |
| `granted` | obrigatório. É `Boolean`, não `boolean`: um campo ausente vira `400`, e não "negado" em silêncio |
| `reason` | opcional (só as negadas têm motivo), até 255 caracteres |
| `durationMs` | obrigatório, `>= 0` |

Os limites de tamanho seguem as colunas da migration `V1`, para que um valor grande demais responda `400` em vez de estourar no banco.

**Response** `201 Created` — o `AuditEventResponse` gravado:
```json
{
  "id": "3848fce2-7590-4fbc-b802-ff41b7085232",
  "type": "PERMISSION_CHECK",
  "projectId": "11111111-1111-1111-1111-111111111111",
  "occurredAt": "2026-09-28T20:00Z",
  "description": "NEGADO GET /produtos para o cargo 'ADMIN' — invalid or inactive api key (3.5 ms)"
}
```

Cada evento gravado também é acrescentado como uma linha em `logs/audit-events.txt`, relativo à pasta de onde o serviço sobe.

**Erros:** `400 Bad Request` com os campos inválidos na mensagem (ex.: `"routePath: must start with /; granted: must not be null"`), ou `"Malformed request body"` para JSON quebrado ou UUID/data em formato inválido.

```bash
curl -X POST http://localhost:8081/audit-events/permission-checks \
  -H "Content-Type: application/json" \
  -d '{"projectId":"11111111-1111-1111-1111-111111111111","occurredAt":"2026-09-28T20:00:00Z","routePath":"/produtos","httpMethod":"GET","roleName":"ADMIN","granted":false,"reason":"invalid or inactive api key","durationMs":3.5}'
```

---

## Mensageria — fila `audit.events` (RabbitMQ)

Não é endpoint HTTP: é o contrato da mensagem que a aplicação principal publica a cada `POST /validate-permission` e que o `audit-service` consome para gravar a trilha (ADR-013, no [log de ADRs](https://github.com/Permission-SaaS/permission_saas/blob/main/docs/ARCHITECTURE.md)). A publicação usa a exchange padrão, com a fila `audit.events` como *routing key*.

**Mensagem** (`AuditMessage`, JSON): um envelope genérico, com os dados próprios do tipo de evento em `payload`.

```json
{
  "source": "permission-service",
  "type": "PERMISSION_CHECK",
  "occurredAt": "2026-10-03T03:27:58.123Z",
  "payload": {
    "projectId": "11111111-1111-1111-1111-111111111111",
    "routePath": "/users",
    "httpMethod": "GET",
    "roleName": "ADMIN",
    "granted": false,
    "reason": "invalid or inactive api key",
    "durationMs": 3.5
  }
}
```

| Campo | Regra |
|---|---|
| `source`, `type` | obrigatórios. Hoje só `type: PERMISSION_CHECK` é aceito |
| `occurredAt` | obrigatório, ISO-8601 em UTC |
| `payload` | obrigatório. Para `PERMISSION_CHECK`, as mesmas regras do corpo do [`POST /audit-events/permission-checks`](#post-audit-eventspermission-checks), sem o `occurredAt`, que vem do envelope |

**O que acontece com cada mensagem:**

- **Válida:** gravada pelo mesmo caso de uso do `POST` e confirmada (*ack*); sai da fila.
- **Com o `audit-service` fora do ar:** espera na fila, que é durável, até ele voltar.
- **Inválida** (tipo desconhecido, payload fora das regras, JSON quebrado), **ou que falhou ao gravar 3 vezes seguidas:** vai para a fila `audit.events.dlq`, onde fica para conferência.

**Como ver:** painel do RabbitMQ em `http://localhost:15672` (usuário `saas`, senha `saas123`), aba *Queues*. Pela API do painel, sem tirar as mensagens da fila:

```bash
curl -u saas:saas123 -H "Content-Type: application/json" \
  -X POST http://localhost:15672/api/queues/%2F/audit.events/get \
  -d '{"count":10,"ackmode":"ack_requeue_true","encoding":"auto"}'
```

No Postman do guarda-chuva, a pasta `5. RabbitMQ` tem três subpastas: `Caminho feliz` (a validação chega à trilha e a fila esvazia), `Mensagem invalida vai para a fila de mortas` (publica um `type` desconhecido e confere a `audit.events.dlq`) e `Consumidor fora do ar (manual)`, que faz o roteiro com o consumidor parado e depois de religá-lo.
