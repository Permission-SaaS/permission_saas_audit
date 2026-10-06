# permission_saas_audit

Serviço de trilha de auditoria do **Permission SaaS** (`audit-service`, porta 8081), extraído do
monolito modular como serviço independente. Tem banco próprio (`audit_db`), grava os eventos que
chegam pela fila `audit.events` do RabbitMQ (ou por `POST`) e responde às consultas em
`GET /audit-events`.

Faz parte da organização [Permission-SaaS](https://github.com/Permission-SaaS). O repositório
[`permission_saas`](https://github.com/Permission-SaaS/permission_saas) é o guarda-chuva: reúne todos
os repositórios como submódulos, sobe o sistema inteiro pelo Docker Compose e guarda a visão de
arquitetura e o log de ADRs.

## Rodar só este serviço

Precisa de JDK 21, de um PostgreSQL 16 em `localhost:5433` (banco `audit_db`, usuário `audit`, senha
`audit123`) e do RabbitMQ em `localhost:5672` (usuário `saas`, senha `saas123`). O profile `dev`, o
padrão, já aponta para eles e não usa o Config Server.

```bash
docker compose up -d audit-postgres rabbitmq   # na raiz do guarda-chuva
./mvnw spring-boot:run                         # porta 8081
curl http://localhost:8081/actuator/health
```

Swagger UI: http://localhost:8081/swagger-ui/index.html. Cada evento gravado também vira uma linha em
`logs/audit-events.txt`, relativo à pasta de onde o serviço sobe.

## Build

```bash
./mvnw clean package
```

O serviço ainda não tem testes automatizados.

## Documentação

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md): camadas, dono dos dados, contrato e a direção futura (auditoria genérica)
- [`docs/DOMAIN.md`](docs/DOMAIN.md): a hierarquia `AuditEvent`
- [`docs/API.md`](docs/API.md): os endpoints e o contrato da fila `audit.events`
