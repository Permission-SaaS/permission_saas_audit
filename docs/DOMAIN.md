# Domínio — `audit-service`

Glossário da trilha de auditoria: as entidades e invariantes que o serviço grava no banco próprio (`audit_db`). A aplicação principal guarda uma cópia de `AuditEvent` e `PermissionCheckEvent` (o evento que ela monta e publica) e o modelo de leitura `AuditTrailEntry`. Ver a [documentação do `permission-service`](https://github.com/Permission-SaaS/permission_saas_api/blob/main/docs/DOMAIN.md#audit), onde também está o próximo campo planejado, o `userId` em `PermissionCheckEvent`.

Na persistência, a herança vira `SINGLE_TABLE`: uma tabela `audit_events`, com a coluna discriminadora `event_type` e `CHECK`s que exigem, para cada tipo, os campos dele (migration `V1`; ADR-004, no [log de ADRs](https://github.com/Permission-SaaS/permission_saas/blob/main/docs/ARCHITECTURE.md)).

---

### AuditEvent (abstrata)

Raiz da trilha de auditoria. É abstrata porque a trilha guarda tipos heterogêneos de evento, cada um com campos próprios — a herança descreve o domínio, não existe só para atender a requisito.

| Campo | Tipo | Observação |
|---|---|---|
| `id` | UUID | PK |
| `projectId` | UUID | FK → `Project`, guardado por id: o projeto mora em outro serviço, com outro banco |
| `occurredAt` | timestamp | default `now()` |

Contrato abstrato: `describe()` (resumo legível, usado no console e no arquivo de auditoria) e `type()` (discriminador — vira a `@DiscriminatorColumn` do mapeamento `SINGLE_TABLE`).

### PermissionCheckEvent

Resultado de uma validação executada pela chain do módulo `permission`. Evento de maior volume da trilha.

| Campo | Tipo | Observação |
|---|---|---|
| `routePath`, `httpMethod`, `roleName` | String | dados da tentativa de acesso |
| `granted` | boolean | resultado |
| `reason` | String | motivo da recusa; `null` quando concedida |
| `durationMs` | double | tempo gasto pela chain |
| `ipAddress`, `country` | String | origem; `country` enriquecido via OpenFeign |

`type()` = `PERMISSION_CHECK`.

### ProjectLifecycleEvent

Mudança estrutural em um projeto.

| Campo | Tipo | Observação |
|---|---|---|
| `action` | `LifecycleAction` enum | `CREATED`, `UPDATED`, `DELETED` |
| `projectName` | String | nome no momento do evento |
| `performedBy` | UUID | FK → `Client` responsável |

`type()` = `PROJECT_LIFECYCLE`.
