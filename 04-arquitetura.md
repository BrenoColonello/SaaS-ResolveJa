# Documento de Arquitetura — ResolveJá

**Versão:** 1.0  
**Data:** 2026-07-02  
**Etapa:** 4 de 7 — Arquitetura do Sistema

---

## Sumário

1. [Estilo Arquitetural](#1-estilo-arquitetural)
2. [Diagrama de Camadas — Backend](#2-diagrama-de-camadas--backend)
3. [Estrutura de Pacotes Java](#3-estrutura-de-pacotes-java)
4. [Diagrama de Componentes](#4-diagrama-de-componentes)
5. [Diagrama de Deployment](#5-diagrama-de-deployment)
6. [Architecture Decision Records (ADRs)](#6-architecture-decision-records-adrs)

---

## 1. Estilo Arquitetural

### Decisão: Monolito Modular

O ResolveJá adota um **Monolito Modular** — uma única aplicação deployável, mas com fronteiras internas bem definidas entre módulos de domínio.

Essa é a mesma decisão tomada por plataformas como **Airbnb** e **Uber** em seus estágios iniciais. Ambas começaram como monolitos (Airbnb em Ruby on Rails até ~2017, Uber em Python + PostgreSQL), e migraram para microservices apenas quando o crescimento de escala e times tornou isso necessário. Para um MVP com time reduzido e base de usuários ainda não validada, os custos operacionais de microservices (latência de rede, deploys independentes, observabilidade distribuída) não têm retorno.

O Monolito Modular oferece:

- **Simplicidade operacional:** um único processo para subir, monitorar e debugar.
- **Coesão de deploy:** sem risco de versões incompatíveis entre serviços.
- **Migração futura segura:** módulos com fronteiras bem definidas podem ser extraídos como serviços independentes sem refatoração traumática.

---

## 2. Diagrama de Camadas — Backend

O backend segue a **Layered Architecture** com quatro camadas, onde a dependência sempre aponta para dentro — camadas externas conhecem as internas, nunca o contrário.

```
┌─────────────────────────────────────────────────┐
│              PRESENTATION LAYER                  │
│   Controllers · Request DTOs · Response DTOs     │
│   Validação de entrada · Mapeamento HTTP         │
└──────────────────────┬──────────────────────────┘
                       │ chama
┌──────────────────────▼──────────────────────────┐
│              APPLICATION LAYER                   │
│   Services · Casos de uso · Orquestração         │
│   Regras de negócio · Transações                 │
└──────────────────────┬──────────────────────────┘
                       │ opera sobre
┌──────────────────────▼──────────────────────────┐
│                DOMAIN LAYER                      │
│   Entities (JPA) · Enums · Interfaces de repo    │
│   Núcleo do negócio — sem dependência externa    │
└──────────────────────┬──────────────────────────┘
                       │ implementado por
┌──────────────────────▼──────────────────────────┐
│             INFRASTRUCTURE LAYER                 │
│   Repositories (Spring Data) · Segurança (JWT)   │
│   Configurações · Filtros · Persistência         │
└─────────────────────────────────────────────────┘
```

### Correspondência com ASP.NET Core

| Spring Boot (Java)        | ASP.NET Core (C#)              |
|---------------------------|--------------------------------|
| `@RestController`         | `[ApiController]`              |
| `@Service`                | Classe de serviço injetada     |
| `JpaRepository`           | `IRepository` / EF DbContext   |
| `@Entity`                 | Classe de modelo / EF Entity   |
| `@Component` / `@Bean`    | Registro no DI container       |
| `application.properties`  | `appsettings.json`             |
| Spring Security Filter    | ASP.NET Middleware              |

### Regra de ouro das camadas

> A **Domain Layer** não importa nada de Spring, JPA, ou qualquer framework. Ela é POJO puro. Isso garante que o núcleo do negócio possa ser testado sem subir o contexto do Spring — equivalente a testar uma classe C# sem instanciar o host.

---

## 3. Estrutura de Pacotes Java

O projeto é organizado **por feature (domínio)**, não por tipo técnico. Isso é uma decisão deliberada: evita que funcionalidades relacionadas fiquem espalhadas pelo projeto e facilita extrair um módulo como microservice no futuro.

```
com.resolveja/
├── config/                         # Configurações globais (CORS, Swagger, Security)
│   ├── SecurityConfig.java
│   ├── SwaggerConfig.java
│   └── JwtConfig.java
│
├── security/                       # Infraestrutura de autenticação JWT
│   ├── JwtFilter.java              # Intercepta requests e valida token
│   ├── JwtService.java             # Gera e valida tokens JWT
│   └── UserDetailsServiceImpl.java
│
├── usuario/                        # Módulo: gestão de usuários e autenticação
│   ├── controller/
│   │   └── AuthController.java
│   ├── service/
│   │   └── UsuarioService.java
│   ├── repository/
│   │   └── UsuarioRepository.java
│   ├── model/
│   │   ├── Usuario.java            # Entidade base (joined-table inheritance)
│   │   ├── Cliente.java
│   │   └── Profissional.java
│   └── dto/
│       ├── LoginRequest.java
│       ├── CadastroRequest.java
│       └── TokenResponse.java
│
├── servico/                        # Módulo: categorias e perfis de serviço
│   ├── controller/
│   │   └── ServicoController.java
│   ├── service/
│   │   └── ServicoService.java
│   ├── repository/
│   │   └── CategoriaRepository.java
│   ├── model/
│   │   ├── Categoria.java
│   │   └── ServicoOferecido.java
│   └── dto/
│       └── ServicoResponse.java
│
├── busca/                          # Módulo: busca de profissionais
│   ├── controller/
│   │   └── BuscaController.java
│   ├── service/
│   │   └── BuscaService.java
│   └── dto/
│       ├── BuscaRequest.java
│       └── ProfissionalResumoResponse.java
│
├── solicitacao/                    # Módulo: núcleo do fluxo de negócio
│   ├── controller/
│   │   └── SolicitacaoController.java
│   ├── service/
│   │   └── SolicitacaoService.java
│   ├── repository/
│   │   └── SolicitacaoRepository.java
│   ├── model/
│   │   ├── Solicitacao.java
│   │   └── StatusSolicitacao.java  # Enum: PENDENTE, ACEITA, RECUSADA, CONCLUIDA, CANCELADA
│   └── dto/
│       ├── AbrirSolicitacaoRequest.java
│       ├── ResponderSolicitacaoRequest.java
│       └── SolicitacaoResponse.java
│
├── mensagem/                       # Módulo: chat interno
│   ├── controller/
│   │   └── MensagemController.java
│   ├── service/
│   │   └── MensagemService.java
│   ├── repository/
│   │   └── MensagemRepository.java
│   ├── model/
│   │   └── Mensagem.java
│   └── dto/
│       ├── EnviarMensagemRequest.java
│       └── MensagemResponse.java
│
├── avaliacao/                      # Módulo: avaliações de profissionais
│   ├── controller/
│   │   └── AvaliacaoController.java
│   ├── service/
│   │   └── AvaliacaoService.java
│   ├── repository/
│   │   └── AvaliacaoRepository.java
│   ├── model/
│   │   └── Avaliacao.java
│   └── dto/
│       ├── CriarAvaliacaoRequest.java
│       └── AvaliacaoResponse.java
│
└── admin/                          # Módulo: administração (escopo mínimo MVP)
    ├── controller/
    │   └── AdminController.java
    └── service/
        └── AdminService.java
```

---

## 4. Diagrama de Componentes

Visão de alto nível dos blocos do sistema e como se comunicam:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTE (Browser)                         │
│                   React + TypeScript + CSS                        │
│          Componentes · React Router · Fetch/Axios                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS / REST (JSON)
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                     BACKEND (Spring Boot)                         │
│                                                                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐ │
│  │   Security   │   │  Controllers │   │      Services        │ │
│  │  JWT Filter  │──▶│  @RestCtrl   │──▶│  Regras de Negócio   │ │
│  │  BCrypt      │   │  DTOs        │   │  Orquestração        │ │
│  └──────────────┘   └──────────────┘   └──────────┬───────────┘ │
│                                                    │             │
│  ┌─────────────────────────────────────────────────▼───────────┐ │
│  │                   Repositories (Spring Data JPA)             │ │
│  │              Interfaces → implementação automática           │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Swagger / OpenAPI (Springdoc)                   │ │
│  │              Documentação automática via anotações           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │ JDBC / JPA
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                       BANCO DE DADOS                              │
│                        PostgreSQL 16                              │
│                   Esquema: resolvejadb                            │
└─────────────────────────────────────────────────────────────────┘
```

### Cortes transversais (Cross-Cutting Concerns)

- **Autenticação/Autorização:** Spring Security com filtro JWT aplicado globalmente, antes de qualquer controller.
- **Tratamento de erros:** `@ControllerAdvice` global — mapeia exceções de negócio para respostas HTTP padronizadas.
- **Validação:** Bean Validation (`@Valid`, `@NotNull`, etc.) na camada de DTOs — nunca nas entidades de domínio.
- **Logs:** SLF4J + Logback (incluso no Spring Boot) — sem configuração extra no MVP.

---

## 5. Diagrama de Deployment

O ambiente é inteiramente containerizado via **Docker Compose**, com três containers:

```
┌─────────────────────────────────────────────────────────────────┐
│                      docker-compose.yml                          │
│                                                                   │
│  ┌───────────────────┐   ┌────────────────────────────────────┐ │
│  │    frontend        │   │            backend                  │ │
│  │  container         │   │          container                  │ │
│  │                   │   │                                    │ │
│  │  React (Nginx)    │   │  Spring Boot (JAR)                 │ │
│  │  porta: 3000      │──▶│  porta: 8080                       │ │
│  │                   │   │                                    │ │
│  └───────────────────┘   └──────────────┬─────────────────────┘ │
│                                         │                        │
│                          ┌──────────────▼─────────────────────┐ │
│                          │              db                      │ │
│                          │          container                   │ │
│                          │                                    │ │
│                          │  PostgreSQL 16                     │ │
│                          │  porta: 5432                       │ │
│                          │  volume: postgres_data             │ │
│                          └────────────────────────────────────┘ │
│                                                                   │
│  Variáveis de ambiente: .env (não commitado — .gitignore)        │
│  Network: resolvejanet (bridge)                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Fluxo de inicialização

1. `db` sobe primeiro (healthcheck no PostgreSQL).
2. `backend` aguarda `db` estar pronto, então sobe e roda migrations (Flyway).
3. `frontend` sobe e aponta para `backend` via variável de ambiente `VITE_API_URL`.

> **Nota:** Flyway é a escolha para migrations de banco — equivalente ao `dotnet ef migrations` do Entity Framework, mas SQL puro. Cada migration é um arquivo versionado (`V1__create_schema.sql`, `V2__...`), commitable no git.

---

## 6. Architecture Decision Records (ADRs)

Cada ADR documenta uma decisão técnica relevante: o contexto, o que foi decidido, as alternativas consideradas e as consequências.

---

### ADR-001 — Monolito Modular vs. Microservices

**Status:** Aceito  
**Data:** 2026-07-02

**Contexto**  
O projeto é um MVP acadêmico com um único desenvolvedor e base de usuários não validada. A decisão de arquitetura de deployment impacta complexidade operacional, velocidade de desenvolvimento e capacidade de evolução futura.

**Decisão**  
Adotar Monolito Modular — uma única aplicação Spring Boot com módulos internos bem delimitados por pacote.

**Alternativas consideradas**  
- *Microservices desde o início:* descartado. Overhead de rede entre serviços, necessidade de service discovery, deploy coordenado e observabilidade distribuída são injustificáveis sem escala real. Airbnb e Uber iniciaram como monolitos e só migraram após crescimento significativo.
- *Monolito sem separação modular:* descartado. Dificulta leitura, testes e eventual extração de serviços.

**Consequências**  
Desenvolvimento mais rápido e operação mais simples. No futuro, cada pacote de domínio (ex: `solicitacao/`, `avaliacao/`) pode ser extraído como microservice com refatoração mínima, pois as fronteiras já estão definidas.

---

### ADR-002 — Autenticação: JWT Stateless vs. Sessão no Servidor

**Status:** Aceito  
**Data:** 2026-07-02

**Contexto**  
A API REST precisa de um mecanismo de autenticação. A escolha afeta escalabilidade, complexidade de implementação e necessidade de estado no servidor.

**Decisão**  
Adotar JWT (JSON Web Token) stateless. O token é gerado no login, enviado pelo cliente em cada request no header `Authorization: Bearer <token>`, e validado pelo filtro do Spring Security sem consulta ao banco.

**Alternativas consideradas**  
- *Sessions HTTP (stateful):* descartado. Requer armazenamento de sessão no servidor (memória ou Redis), o que adiciona infraestrutura e complica o cenário com múltiplas instâncias.
- *OAuth2 / OIDC (ex: Keycloak):* descartado para MVP. Adiciona um servidor de identidade externo desnecessário neste momento.

**Consequências**  
Senhas armazenadas com BCrypt (hash unidirecional). Tokens têm expiração configurável. Logout do lado do servidor requer blacklist de tokens (não implementado no MVP — token simplesmente expira). O frontend descarta o token localmente no logout.

---

### ADR-003 — Banco de Dados: PostgreSQL vs. NoSQL

**Status:** Aceito  
**Data:** 2026-07-02

**Contexto**  
Os dados do ResolveJá são inerentemente relacionais: usuários têm solicitações, solicitações têm avaliações, profissionais têm categorias e bairros. A escolha do banco afeta integridade referencial, facilidade de query e familiaridade da equipe.

**Decisão**  
Adotar PostgreSQL 16 como banco relacional principal.

**Alternativas consideradas**  
- *MongoDB (NoSQL):* descartado. O modelo de dados é fortemente relacional com constraints de integridade importantes (ex: avaliação só existe se Solicitação CONCLUÍDA). Relações many-to-many e FKs são naturais em SQL; emulá-las em documento seria complexo e frágil.
- *MySQL:* descartado por preferência técnica — PostgreSQL tem suporte a tipos avançados, melhor conformidade com o padrão SQL e ecossistema mais robusto com Spring Data JPA.

**Consequências**  
Spring Data JPA + Hibernate como ORM. Flyway para versionamento de schema. Acesso direto via JDBC desaconselhado — toda interação passa pelo repositório JPA.

---

### ADR-004 — Hierarquia de Usuários: Joined-Table Inheritance

**Status:** Aceito  
**Data:** 2026-07-02

**Contexto**  
O sistema possui três tipos de usuário (Cliente, Profissional, Administrador) com atributos comuns (email, senha, nome) e atributos específicos por tipo. O modelo de herança no banco impacta a estrutura de queries e a complexidade do schema.

**Decisão**  
Adotar **Joined-Table Inheritance** (`@Inheritance(strategy = InheritanceType.JOINED)` no JPA). Tabela `usuario` com colunas comuns; tabelas `cliente`, `profissional` e `administrador` com colunas específicas, ligadas por FK à `usuario`.

**Alternativas consideradas**  
- *Single Table Inheritance:* todas as colunas numa única tabela `usuario`, com `NULL` nas colunas não aplicáveis ao tipo. Mais simples e performático para leitura, mas gera schema poluído e dificulta constraints `NOT NULL` onde necessário.
- *Table Per Class:* uma tabela completa por tipo, sem tabela base. Descartado — duplica colunas comuns e complica queries polimórficas (ex: buscar qualquer usuário por email).

**Consequências**  
Queries que precisam de dados específicos de `Profissional` fazem JOIN com `usuario`. Custo de performance aceitável no porte do MVP. A regra de negócio de avaliação (só permitida entre Cliente e Profissional com Solicitação CONCLUÍDA) é validada na camada de Service, não no banco.

---

### ADR-005 — Organização de Pacotes: Por Feature vs. Por Camada

**Status:** Aceito  
**Data:** 2026-07-02

**Contexto**  
Projetos Spring Boot podem ser organizados de duas formas: por camada técnica (`controller/`, `service/`, `repository/` na raiz) ou por módulo de domínio (`solicitacao/`, `avaliacao/`, cada um com suas subcamadas).

**Decisão**  
Organizar por **feature (domínio)**. Cada módulo (`solicitacao/`, `avaliacao/`, etc.) contém seu próprio `controller/`, `service/`, `repository/`, `model/` e `dto/`.

**Alternativas consideradas**  
- *Organização por camada:* familiar para iniciantes, mas produz alta acoplamento implícito — ao trabalhar numa funcionalidade, o desenvolvedor navega por múltiplas pastas distantes no projeto. Em projetos maiores, torna-se difícil de manter.

**Consequências**  
Toda a lógica de uma funcionalidade fica co-localizada. Facilita onboarding, code review e eventual extração de módulo. O custo é que não existe uma pasta `service/` ou `controller/` única — cada módulo tem a sua. Desenvolvedor novo precisa entender a convenção antes de navegar.

---

*Documento gerado na Etapa 4 do projeto ResolveJá. Próximas etapas: Diagramas de Sequência (05) → Contratos de API (06) → Implementação (07).*
