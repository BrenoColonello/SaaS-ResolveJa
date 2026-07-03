# Contratos de API — ResolveJá

**Versão:** 1.0  
**Data:** 2026-07-02  
**Etapa:** 6 de 7 — Contratos REST

> Durante a implementação, estes contratos serão refletidos automaticamente no Swagger UI via **Springdoc OpenAPI** (`/swagger-ui.html`). Este documento serve como especificação prévia — fonte de verdade antes do código existir.

---

## Convenções Gerais

### Base URL
```
http://localhost:8080/api/v1
```

### Autenticação
Todas as rotas marcadas com [Auth] exigem o header:
```
Authorization: Bearer <jwt_token>
```
Rotas marcadas com [Público] são públicas.

### Formato de resposta de erro (padrão)
```json
{
  "status": 400,
  "erro": "Descrição do erro",
  "timestamp": "2026-07-02T10:00:00Z"
}
```

### Papéis (roles)
| Role | Descrição |
|---|---|
| `CLIENTE` | Usuário que busca e contrata |
| `PROFISSIONAL` | Usuário que oferece serviços |
| `ADMIN` | Administrador interno |

---

## Sumário de Endpoints

| Grupo | Método | Rota | Auth | Descrição |
|---|---|---|---|---|
| Auth | POST | `/auth/cadastro` | [Público] | Cadastro de novo usuário |
| Auth | POST | `/auth/login` | [Público] | Login e geração de JWT |
| Perfil | GET | `/perfil` | [Auth] | Retorna perfil do usuário autenticado |
| Perfil | PUT | `/perfil` | [Auth] | Atualiza perfil do usuário autenticado |
| Busca | GET | `/busca` | [Público] | Busca profissionais por categoria e bairro |
| Busca | GET | `/profissionais/{id}` | [Público] | Perfil completo de um profissional |
| Categorias | GET | `/categorias` | [Público] | Lista todas as categorias disponíveis |
| Solicitações | POST | `/solicitacoes` | [Auth: CLIENTE] | Abre nova solicitação |
| Solicitações | GET | `/solicitacoes/enviadas` | [Auth: CLIENTE] | Lista solicitações enviadas pelo cliente |
| Solicitações | GET | `/solicitacoes/recebidas` | [Auth: PROFISSIONAL] | Lista solicitações recebidas |
| Solicitações | GET | `/solicitacoes/{id}` | [Auth] | Detalhe de uma solicitação |
| Solicitações | PATCH | `/solicitacoes/{id}/aceitar` | [Auth: PROFISSIONAL] | Aceita uma solicitação pendente |
| Solicitações | PATCH | `/solicitacoes/{id}/recusar` | [Auth: PROFISSIONAL] | Recusa uma solicitação pendente |
| Solicitações | PATCH | `/solicitacoes/{id}/concluir` | [Auth: CLIENTE] | Marca solicitação como concluída |
| Solicitações | PATCH | `/solicitacoes/{id}/cancelar` | [Auth] | Cancela uma solicitação |
| Mensagens | GET | `/solicitacoes/{id}/mensagens` | [Auth] | Lista mensagens de uma solicitação |
| Mensagens | POST | `/solicitacoes/{id}/mensagens` | [Auth] | Envia mensagem numa solicitação |
| Avaliações | POST | `/avaliacoes` | [Auth: CLIENTE] | Cria avaliação de profissional |
| Avaliações | GET | `/profissionais/{id}/avaliacoes` | [Público] | Lista avaliações de um profissional |
| Avaliações | POST | `/avaliacoes/{id}/resposta` | [Auth: PROFISSIONAL] | Profissional responde a uma avaliação |
| Admin | PATCH | `/admin/usuarios/{id}/desativar` | [Auth: ADMIN] | Desativa um perfil |

---

## 1. Autenticação

### POST `/auth/cadastro` [Público]

Registra novo usuário. O campo `papel` define se será Cliente ou Profissional.

**Request body:**
```json
{
  "nome": "João Silva",
  "email": "joao@email.com",
  "senha": "minhasenha123",
  "papel": "CLIENTE",
  "telefone": "14999990000",
  "bairro": "Centro"
}
```

> Para `papel: "PROFISSIONAL"`, campos adicionais:
```json
{
  "nome": "Maria Eletricista",
  "email": "maria@email.com",
  "senha": "minhasenha123",
  "papel": "PROFISSIONAL",
  "telefone": "14988880000",
  "descricao": "Eletricista com 10 anos de experiência",
  "bairroAtuacao": "Vila Mariana",
  "categoriaIds": [1, 3]
}
```

**Responses:**

| Status | Descrição | Body |
|---|---|---|
| `201 Created` | Cadastro realizado | `{ "id": 1, "nome": "João Silva", "email": "...", "papel": "CLIENTE" }` |
| `409 Conflict` | Email já cadastrado | `{ "erro": "Email já cadastrado" }` |
| `400 Bad Request` | Dados inválidos | `{ "erro": "...", "campos": { "email": "formato inválido" } }` |

---

### POST `/auth/login` [Público]

Autentica o usuário e retorna o JWT.

**Request body:**
```json
{
  "email": "joao@email.com",
  "senha": "minhasenha123"
}
```

**Responses:**

| Status | Descrição | Body |
|---|---|---|
| `200 OK` | Login bem-sucedido | `{ "token": "eyJ...", "tipo": "Bearer", "expiracao": "2026-07-03T10:00:00Z", "papel": "CLIENTE" }` |
| `401 Unauthorized` | Credenciais inválidas | `{ "erro": "Email ou senha incorretos" }` |

---

## 2. Perfil

### GET `/perfil` [Auth]

Retorna os dados do usuário autenticado (Cliente ou Profissional).

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Dados do perfil |
| `401 Unauthorized` | Token inválido ou ausente |

**Response body (CLIENTE):**
```json
{
  "id": 1,
  "nome": "João Silva",
  "email": "joao@email.com",
  "telefone": "14999990000",
  "bairro": "Centro",
  "papel": "CLIENTE"
}
```

**Response body (PROFISSIONAL):**
```json
{
  "id": 2,
  "nome": "Maria Eletricista",
  "email": "maria@email.com",
  "telefone": "14988880000",
  "descricao": "Eletricista com 10 anos de experiência",
  "bairroAtuacao": "Vila Mariana",
  "notaMedia": 4.8,
  "totalAvaliacoes": 23,
  "categorias": [
    { "id": 1, "nome": "Eletricista" }
  ],
  "papel": "PROFISSIONAL"
}
```

---

### PUT `/perfil` [Auth]

Atualiza dados do perfil do usuário autenticado. Enviar apenas os campos a atualizar.

**Request body (CLIENTE):**
```json
{
  "nome": "João S.",
  "telefone": "14999991111",
  "bairro": "Jardim Europa"
}
```

**Request body (PROFISSIONAL):**
```json
{
  "descricao": "Eletricista e instalador de ar-condicionado",
  "bairroAtuacao": "Fragata",
  "categoriaIds": [1, 5]
}
```

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Perfil atualizado |
| `400 Bad Request` | Dados inválidos |
| `401 Unauthorized` | Token inválido |

---

## 3. Busca

### GET `/busca` [Público]

Busca profissionais filtrando por categoria e bairro. Ambos os parâmetros são opcionais — sem filtros retorna todos.

**Query params:**
| Param | Tipo | Obrigatório | Exemplo |
|---|---|---|---|
| `categoria` | string | não | `Eletricista` |
| `bairro` | string | não | `Centro` |
| `page` | int | não (default: 0) | `0` |
| `size` | int | não (default: 20) | `20` |

**Exemplo:** `GET /busca?categoria=Eletricista&bairro=Centro&page=0&size=10`

**Response `200 OK`:**
```json
{
  "profissionais": [
    {
      "id": 2,
      "nome": "Maria Eletricista",
      "descricao": "Eletricista com 10 anos de experiência",
      "bairroAtuacao": "Centro",
      "notaMedia": 4.8,
      "totalAvaliacoes": 23,
      "categorias": ["Eletricista"]
    }
  ],
  "totalItens": 1,
  "totalPaginas": 1,
  "paginaAtual": 0
}
```

---

### GET `/profissionais/{id}` [Público]

Retorna o perfil completo de um profissional, incluindo avaliações recentes.

**Path param:** `id` — ID do profissional

**Response `200 OK`:**
```json
{
  "id": 2,
  "nome": "Maria Eletricista",
  "descricao": "Eletricista com 10 anos de experiência",
  "telefone": "14988880000",
  "bairroAtuacao": "Centro",
  "notaMedia": 4.8,
  "totalAvaliacoes": 23,
  "categorias": [
    { "id": 1, "nome": "Eletricista" }
  ],
  "avaliacoesRecentes": [
    {
      "id": 10,
      "nota": 5,
      "comentario": "Excelente trabalho!",
      "nomeCliente": "João Silva",
      "criadoEm": "2026-06-15T14:00:00Z",
      "respostaProfissional": null
    }
  ]
}
```

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Perfil encontrado |
| `404 Not Found` | Profissional não existe |

---

## 4. Categorias

### GET `/categorias` [Público]

Lista todas as categorias de serviço disponíveis.

**Response `200 OK`:**
```json
[
  { "id": 1, "nome": "Eletricista" },
  { "id": 2, "nome": "Encanador" },
  { "id": 3, "nome": "Diarista" },
  { "id": 4, "nome": "Pedreiro" },
  { "id": 5, "nome": "Manicure" }
]
```

---

## 5. Solicitações de Serviço

### POST `/solicitacoes` [Auth: CLIENTE]

Abre uma nova solicitação de serviço para um profissional.

**Request body:**
```json
{
  "profissionalId": 2,
  "descricao": "Preciso trocar uma tomada queimada na sala"
}
```

**Responses:**

| Status | Descrição | Body |
|---|---|---|
| `201 Created` | Solicitação criada | SolicitacaoResponse |
| `404 Not Found` | Profissional não encontrado | — |
| `403 Forbidden` | Usuário não é CLIENTE | — |

**SolicitacaoResponse:**
```json
{
  "id": 15,
  "status": "PENDENTE",
  "descricao": "Preciso trocar uma tomada queimada na sala",
  "cliente": { "id": 1, "nome": "João Silva" },
  "profissional": { "id": 2, "nome": "Maria Eletricista" },
  "formaContato": null,
  "telefoneContato": null,
  "criadoEm": "2026-07-02T09:00:00Z",
  "atualizadoEm": "2026-07-02T09:00:00Z"
}
```

---

### GET `/solicitacoes/enviadas` [Auth: CLIENTE]

Lista todas as solicitações enviadas pelo cliente autenticado.

**Query params:** `status` (opcional) — filtra por status (`PENDENTE`, `ACEITA`, `CONCLUIDA`, etc.)

**Response `200 OK`:** lista de SolicitacaoResponse

---

### GET `/solicitacoes/recebidas` [Auth: PROFISSIONAL]

Lista todas as solicitações recebidas pelo profissional autenticado.

**Query params:** `status` (opcional)

**Response `200 OK`:** lista de SolicitacaoResponse

---

### GET `/solicitacoes/{id}` [Auth]

Retorna detalhes de uma solicitação. Apenas o Cliente ou Profissional envolvidos podem visualizar.

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | SolicitacaoResponse |
| `403 Forbidden` | Usuário não é parte da solicitação |
| `404 Not Found` | Solicitação não existe |

---

### PATCH `/solicitacoes/{id}/aceitar` [Auth: PROFISSIONAL]

Aceita uma solicitação `PENDENTE` e define a forma de contato oferecida ao cliente.

**Request body:**
```json
{
  "formaContato": "WHATSAPP",
  "telefoneContato": "14988880000"
}
```

> `formaContato` aceita: `"WHATSAPP"`, `"CHAT"`, `"AMBOS"`  
> `telefoneContato` é obrigatório quando `formaContato` é `"WHATSAPP"` ou `"AMBOS"`

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | SolicitacaoResponse com status `ACEITA` |
| `403 Forbidden` | Usuário não é o profissional da solicitação |
| `422 Unprocessable Entity` | Status atual não é `PENDENTE` |

---

### PATCH `/solicitacoes/{id}/recusar` [Auth: PROFISSIONAL]

Recusa uma solicitação `PENDENTE`.

**Request body:** *(vazio)*

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | SolicitacaoResponse com status `RECUSADA` |
| `403 Forbidden` | Usuário não é o profissional da solicitação |
| `422 Unprocessable Entity` | Status atual não é `PENDENTE` |

---

### PATCH `/solicitacoes/{id}/concluir` [Auth: CLIENTE]

Marca uma solicitação `ACEITA` como `CONCLUÍDA`. Habilita avaliação.

**Request body:** *(vazio)*

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | SolicitacaoResponse com status `CONCLUIDA` |
| `403 Forbidden` | Usuário não é o cliente da solicitação |
| `422 Unprocessable Entity` | Status atual não é `ACEITA` |

---

### PATCH `/solicitacoes/{id}/cancelar` [Auth]

Cancela uma solicitação. Cliente pode cancelar se `PENDENTE`; Profissional pode cancelar se `ACEITA`.

**Request body:** *(vazio)*

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | SolicitacaoResponse com status `CANCELADA` |
| `403 Forbidden` | Usuário não tem permissão no estado atual |
| `422 Unprocessable Entity` | Transição de status inválida |

---

## 6. Mensagens (Chat Interno)

> Disponível apenas em solicitações onde `formaContato` é `"CHAT"` ou `"AMBOS"`.

### GET `/solicitacoes/{id}/mensagens` [Auth]

Lista todas as mensagens de uma solicitação em ordem cronológica.

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Lista de mensagens |
| `403 Forbidden` | Usuário não é parte da solicitação |

**Response `200 OK`:**
```json
[
  {
    "id": 1,
    "remetente": { "id": 1, "nome": "João Silva" },
    "conteudo": "Olá, qual o prazo para o serviço?",
    "enviadoEm": "2026-07-02T10:00:00Z"
  },
  {
    "id": 2,
    "remetente": { "id": 2, "nome": "Maria Eletricista" },
    "conteudo": "Posso ir amanhã à tarde.",
    "enviadoEm": "2026-07-02T10:05:00Z"
  }
]
```

---

### POST `/solicitacoes/{id}/mensagens` [Auth]

Envia uma nova mensagem na solicitação.

**Request body:**
```json
{
  "conteudo": "Olá, qual o prazo para o serviço?"
}
```

**Responses:**

| Status | Descrição |
|---|---|
| `201 Created` | Mensagem enviada |
| `403 Forbidden` | Usuário não é parte da solicitação |
| `422 Unprocessable Entity` | Chat não habilitado nesta solicitação |

---

## 7. Avaliações

### POST `/avaliacoes` [Auth: CLIENTE]

Cria uma avaliação para um profissional. **Regra de negócio validada no backend:** deve existir Solicitação `CONCLUÍDA` entre o cliente autenticado e o profissional informado.

**Request body:**
```json
{
  "profissionalId": 2,
  "solicitacaoId": 15,
  "nota": 5,
  "comentario": "Serviço impecável, rápido e bem feito!"
}
```

**Responses:**

| Status | Descrição |
|---|---|
| `201 Created` | Avaliação criada |
| `403 Forbidden` | Não existe Solicitação CONCLUÍDA entre os dois |
| `409 Conflict` | Avaliação já realizada para esta solicitação |

**Response `201 Created`:**
```json
{
  "id": 30,
  "nota": 5,
  "comentario": "Serviço impecável, rápido e bem feito!",
  "profissional": { "id": 2, "nome": "Maria Eletricista", "notaMedia": 4.9 },
  "criadoEm": "2026-07-02T11:00:00Z"
}
```

---

### GET `/profissionais/{id}/avaliacoes` [Público]

Lista avaliações de um profissional com paginação.

**Query params:** `page` (default: 0), `size` (default: 10)

**Response `200 OK`:**
```json
{
  "avaliacoes": [
    {
      "id": 30,
      "nota": 5,
      "comentario": "Serviço impecável!",
      "nomeCliente": "João Silva",
      "criadoEm": "2026-07-02T11:00:00Z",
      "respostaProfissional": "Obrigado pela confiança!"
    }
  ],
  "notaMedia": 4.9,
  "totalAvaliacoes": 24,
  "totalPaginas": 3,
  "paginaAtual": 0
}
```

---

### POST `/avaliacoes/{id}/resposta` [Auth: PROFISSIONAL]

Permite ao profissional responder publicamente a uma avaliação recebida.

**Request body:**
```json
{
  "resposta": "Obrigado pela confiança, foi um prazer atender!"
}
```

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Resposta registrada |
| `403 Forbidden` | Avaliação não é do profissional autenticado |
| `409 Conflict` | Avaliação já possui resposta |

---

## 8. Administração

### PATCH `/admin/usuarios/{id}/desativar` [Auth: ADMIN]

Desativa o perfil de um usuário (Cliente ou Profissional).

**Request body:** *(vazio)*

**Responses:**

| Status | Descrição |
|---|---|
| `200 OK` | Usuário desativado |
| `403 Forbidden` | Usuário autenticado não é ADMIN |
| `404 Not Found` | Usuário não existe |

---

## Máquina de Estados da Solicitação

```
          ┌─────────────┐
          │   PENDENTE  │
          └──────┬──────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
   ┌─────────┐       ┌──────────┐
   │  ACEITA │       │ RECUSADA │
   └────┬────┘       └──────────┘
        │
   ┌────┴────────┐
   ▼             ▼
┌──────────┐  ┌───────────┐
│ CONCLUÍDA│  │ CANCELADA │
└──────────┘  └───────────┘
```

| Transição | Quem pode | Condição |
|---|---|---|
| PENDENTE → ACEITA | Profissional | status atual = PENDENTE |
| PENDENTE → RECUSADA | Profissional | status atual = PENDENTE |
| PENDENTE → CANCELADA | Cliente | status atual = PENDENTE |
| ACEITA → CONCLUÍDA | Cliente | status atual = ACEITA |
| ACEITA → CANCELADA | Profissional | status atual = ACEITA |

---

*Documento gerado na Etapa 6 do projeto ResolveJá. Próximas etapas: Implementação (07) → Relatório Final.*
