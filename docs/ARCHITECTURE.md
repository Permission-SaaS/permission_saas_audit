# Arquitetura — `audit-service`

## Ideia central

O `audit-service` é a trilha de auditoria do Permission SaaS, extraída do monolito modular como o primeiro serviço independente, no padrão *Strangler Fig*. Este documento descreve o interior do serviço. Os fluxos entre serviços (a gravação pela fila, a consulta por OpenFeign, o Config Server) e o log de decisões (ADR-001 … ADR-015) ficam no [`docs/ARCHITECTURE.md` do repositório guarda-chuva](https://github.com/Permission-SaaS/permission_saas/blob/main/docs/ARCHITECTURE.md). Os números de ADR citados aqui apontam para lá.

---

## Estrutura

Aplicação Spring Boot própria (porta 8081), com as mesmas quatro camadas de um módulo do monolito — sem subpacote de módulo, porque o serviço inteiro **é** o módulo:

```
com.saas.audit/
  domain/          AuditEvent e subclasses, AuditEventRepository e AuditEventJournal (portas), exception/
  application/     SearchAuditEventsUseCase (query/), RegisterPermissionCheckUseCase (command/)
  infrastructure/  entidades JPA SINGLE_TABLE, consultas JPQL do ADR-009, AuditEventFileWriter,
                   messaging/ (declaração das filas e conversor JSON)
  api/             AuditEventController, DTOs, mappers, GlobalExceptionHandler,
                   messaging/AuditMessageListener (consumidor da fila audit.events)
```

- **Dono dos próprios dados:** banco `audit_db` num container Postgres próprio (`audit-postgres`, porta 5433), com usuário próprio — a aplicação principal nem tem credencial para entrar nele. O schema é da migration `V1` do serviço, cópia da `V8` do monolito.
- **Contrato pela rede, não por código:** `GET /audit-events`, `POST /audit-events/permission-checks` e, desde a etapa 4, a mensagem da fila `audit.events` no RabbitMQ (ver [`API.md`](API.md)). O que os dois lados têm em comum — `Mapper`, `ErrorResponse`, as exceções base — foi **copiado**, não compartilhado por biblioteca (ADR-008).
- **O que ficou de fora:** Spring Security (o serviço é chamado pela rede interna, não por clientes), Spring Modulith (é um módulo só), `AuditDemoRunner` e `AuditLogListener` — no serviço, o evento chega pela fila ou pelo `POST`, não por evento em memória.

---

## Direção futura

**Direção futura: um serviço de auditoria genérico (registrada em 01/10/2026, fora do escopo atual).** A
intenção do autor é tornar o `audit-service` um sistema de auditoria próprio e independente, em
repositório separado, do qual o `permission-service` seria só um dos clientes, ao lado de outros
projetos. **O repositório próprio já existe desde 05/10/2026** (ADR-015), extraído com o histórico.
Torná-lo genérico ainda é redesenho, porque o modelo de hoje fala a língua deste produto:

| Hoje | Genérico |
|---|---|
| Tipos fixos `PERMISSION_CHECK`/`PROJECT_LIFECYCLE`, com `CHECK` no banco | `type` livre, definido por quem envia |
| Colunas de permissão (`route_path`, `role_name`, `granted`...) | Um envelope comum (`source`, `type`, `occurredAt`, `actor`, `description`) e o resto num `payload` JSON (`jsonb`) |
| `POST /audit-events/permission-checks` | `POST /events`, um endpoint para qualquer tipo |
| Sem autenticação: só a rede interna chama | Chave de API por sistema cliente, com isolamento dos dados de cada cliente |
| Filtro `onlyDenied` | Filtros por `source`, `type` e período, e filtro dentro do `payload` |

Do lado do `permission-service`, a troca fica contida: o resto do módulo só conhece as portas
`AuditEventPublisher` e `AuditTrail`, então mudam apenas os adapters e os DTOs de `messaging/dto/` e
`client/dto/`. **Feito na etapa 4:** a mensagem do RabbitMQ já usa o envelope genérico (`source`,
`type`, `occurredAt`, `payload`, ADR-013). O custo foi o mesmo de um formato específico, e o formato
fica pronto para a evolução.
