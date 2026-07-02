# Diagramas de Sequência — ResolveJá

**Versão:** 1.0  
**Data:** 2026-07-02  
**Etapa:** 5 de 7 — Fluxos Críticos do Sistema

---

## Sumário

1. [Autenticação — Cadastro e Login com JWT](#1-autenticação--cadastro-e-login-com-jwt)
2. [Busca de Profissionais](#2-busca-de-profissionais)
3. [Solicitação de Serviço — Abertura e Resposta do Profissional](#3-solicitação-de-serviço--abertura-e-resposta-do-profissional)
4. [Conclusão de Solicitação](#4-conclusão-de-solicitação)
5. [Avaliação de Profissional](#5-avaliação-de-profissional)

---

## 1. Autenticação — Cadastro e Login com JWT

Este fluxo cobre o registro de um novo usuário e o login retornando um token JWT. O token é usado em todos os demais fluxos no header `Authorization: Bearer <token>`.

```mermaid
sequenceDiagram
    actor Usuario as Usuário (Browser)
    participant FE as Frontend (React)
    participant SEC as Spring Security
    participant AC as AuthController
    participant US as UsuarioService
    participant DB as PostgreSQL

    Note over Usuario, DB: ── CADASTRO ──

    Usuario->>FE: preenche formulário de cadastro
    FE->>AC: POST /auth/cadastro { nome, email, senha, papel }
    AC->>US: cadastrar(request)
    US->>DB: SELECT * FROM usuario WHERE email = ?
    DB-->>US: resultado

    alt email já cadastrado
        US-->>AC: throw EmailJaCadastradoException
        AC-->>FE: 409 Conflict { "erro": "Email já cadastrado" }
        FE-->>Usuario: exibe mensagem de erro
    else email disponível
        US->>US: BCrypt.hash(senha)
        US->>DB: INSERT INTO usuario (...) + INSERT INTO cliente/profissional (...)
        DB-->>US: entidade salva
        US-->>AC: UsuarioResponse
        AC-->>FE: 201 Created { id, nome, email, papel }
        FE-->>Usuario: redireciona para login
    end

    Note over Usuario, DB: ── LOGIN ──

    Usuario->>FE: informa email e senha
    FE->>SEC: POST /auth/login { email, senha }
    Note right of SEC: rota pública — JWT Filter<br/>não exige token aqui
    SEC->>AC: passa requisição
    AC->>US: autenticar(email, senha)
    US->>DB: SELECT * FROM usuario WHERE email = ?
    DB-->>US: usuario com senha hash

    alt credenciais inválidas
        US-->>AC: throw CredenciaisInvalidasException
        AC-->>FE: 401 Unauthorized
        FE-->>Usuario: exibe "email ou senha incorretos"
    else credenciais válidas
        US->>US: BCrypt.verify(senha, hash)
        US->>US: JwtService.gerarToken(usuario)
        US-->>AC: TokenResponse { token, expiracao }
        AC-->>FE: 200 OK { token, expiracao }
        FE->>FE: armazena token (memória/estado)
        FE-->>Usuario: redireciona para home autenticada
    end
```

---

## 2. Busca de Profissionais

Fluxo de busca por profissionais filtrando por categoria e bairro. Pode ser acessado por usuários não autenticados (busca pública) ou autenticados.

```mermaid
sequenceDiagram
    actor Cliente as Cliente (Browser)
    participant FE as Frontend (React)
    participant SEC as Spring Security / JwtFilter
    participant BC as BuscaController
    participant BS as BuscaService
    participant DB as PostgreSQL

    Cliente->>FE: seleciona categoria + bairro e clica Buscar
    FE->>SEC: GET /busca?categoria=Eletricista&bairro=Centro

    Note right of SEC: rota pública — token opcional<br/>JwtFilter não bloqueia

    SEC->>BC: repassa requisição
    BC->>BS: buscar(categoria, bairro)
    BS->>DB: SELECT p.*, u.nome, p.nota_media<br/>FROM profissional p<br/>JOIN usuario u ON u.id = p.usuario_id<br/>JOIN servico_oferecido so ON so.profissional_id = p.id<br/>JOIN categoria c ON c.id = so.categoria_id<br/>WHERE c.nome ILIKE ? AND p.bairro_atuacao ILIKE ?<br/>ORDER BY p.nota_media DESC
    DB-->>BS: lista de profissionais

    alt nenhum resultado encontrado
        BS-->>BC: lista vazia
        BC-->>FE: 200 OK { profissionais: [] }
        FE-->>Cliente: exibe "nenhum profissional encontrado"
    else resultados encontrados
        BS-->>BC: List<ProfissionalResumoResponse>
        BC-->>FE: 200 OK { profissionais: [...] }
        FE-->>Cliente: exibe cards com nome, categoria, bairro, nota média
    end

    Cliente->>FE: clica no card de um profissional
    FE->>BC: GET /profissionais/{id}
    BC->>BS: buscarPerfil(id)
    BS->>DB: SELECT completo do profissional (+ portfólio, avaliações)
    DB-->>BS: ProfissionalDetalheResponse
    BS-->>BC: ProfissionalDetalheResponse
    BC-->>FE: 200 OK { perfil completo }
    FE-->>Cliente: exibe página de perfil do profissional
```

---

## 3. Solicitação de Serviço — Abertura e Resposta do Profissional

Este é o fluxo central do ResolveJá. O Cliente abre uma solicitação, que inicia como `PENDENTE`. O Profissional pode `ACEITAR` (escolhendo a forma de contato) ou `RECUSAR`.

```mermaid
sequenceDiagram
    actor Cliente as Cliente
    actor Prof as Profissional
    participant FE as Frontend (React)
    participant SEC as JwtFilter
    participant SC as SolicitacaoController
    participant SS as SolicitacaoService
    participant DB as PostgreSQL

    Note over Cliente, DB: ── CLIENTE ABRE SOLICITAÇÃO ──

    Cliente->>FE: preenche descrição e clica "Solicitar Serviço"
    FE->>SEC: POST /solicitacoes<br/>Authorization: Bearer <token><br/>{ profissionalId, descricao }
    SEC->>SEC: valida JWT → extrai clienteId
    SEC->>SC: repassa com usuário autenticado

    SC->>SS: abrir(clienteId, profissionalId, descricao)
    SS->>DB: SELECT * FROM usuario WHERE id = profissionalId AND tipo = 'PROFISSIONAL'
    DB-->>SS: profissional existe

    SS->>DB: INSERT INTO solicitacao<br/>{ clienteId, profissionalId, descricao, status: PENDENTE }
    DB-->>SS: Solicitacao { id, status: PENDENTE }
    SS-->>SC: SolicitacaoResponse
    SC-->>FE: 201 Created { id, status: "PENDENTE" }
    FE-->>Cliente: exibe "Solicitação enviada! Aguardando resposta."

    Note over Cliente, DB: ── PROFISSIONAL RESPONDE ──

    Prof->>FE: acessa lista de solicitações pendentes
    FE->>SEC: GET /solicitacoes/recebidas<br/>Authorization: Bearer <token>
    SEC->>SEC: valida JWT → extrai profissionalId
    SEC->>SC: repassa
    SC->>SS: listarRecebidas(profissionalId)
    SS->>DB: SELECT * FROM solicitacao WHERE profissional_id = ? AND status = 'PENDENTE'
    DB-->>SS: lista de solicitações
    SS-->>SC: List<SolicitacaoResponse>
    SC-->>FE: 200 OK [{ id, cliente, descricao, status }]
    FE-->>Prof: exibe solicitações pendentes

    alt Profissional ACEITA
        Prof->>FE: clica Aceitar e escolhe forma de contato (WhatsApp / Chat)
        FE->>SEC: PATCH /solicitacoes/{id}/aceitar<br/>{ formaContato: "WHATSAPP", telefone: "..." }
        SEC->>SC: repassa com profissionalId autenticado
        SC->>SS: aceitar(solicitacaoId, profissionalId, formaContato)
        SS->>DB: SELECT * FROM solicitacao WHERE id = ? AND profissional_id = ? AND status = 'PENDENTE'
        DB-->>SS: solicitacao válida
        SS->>DB: UPDATE solicitacao SET status = 'ACEITA', forma_contato = ?, telefone = ?
        DB-->>SS: ok
        SS-->>SC: SolicitacaoResponse { status: ACEITA, formaContato, telefone }
        SC-->>FE: 200 OK
        FE-->>Prof: "Solicitação aceita"
        FE-->>Cliente: (no próximo acesso) exibe status ACEITA + dados de contato

    else Profissional RECUSA
        Prof->>FE: clica Recusar
        FE->>SEC: PATCH /solicitacoes/{id}/recusar
        SEC->>SC: repassa com profissionalId autenticado
        SC->>SS: recusar(solicitacaoId, profissionalId)
        SS->>DB: SELECT * FROM solicitacao WHERE id = ? AND profissional_id = ? AND status = 'PENDENTE'
        DB-->>SS: solicitacao válida
        SS->>DB: UPDATE solicitacao SET status = 'RECUSADA'
        DB-->>SS: ok
        SS-->>SC: SolicitacaoResponse { status: RECUSADA }
        SC-->>FE: 200 OK
        FE-->>Prof: "Solicitação recusada"
    end
```

---

## 4. Conclusão de Solicitação

Somente o Cliente pode marcar uma solicitação como `CONCLUÍDA`. A solicitação precisa estar com status `ACEITA`. Esse status é pré-requisito para avaliação (Fluxo 5).

```mermaid
sequenceDiagram
    actor Cliente as Cliente
    participant FE as Frontend (React)
    participant SEC as JwtFilter
    participant SC as SolicitacaoController
    participant SS as SolicitacaoService
    participant DB as PostgreSQL

    Cliente->>FE: acessa solicitação aceita e clica "Marcar como Concluído"
    FE->>SEC: PATCH /solicitacoes/{id}/concluir<br/>Authorization: Bearer <token>
    SEC->>SEC: valida JWT → extrai clienteId
    SEC->>SC: repassa requisição

    SC->>SS: concluir(solicitacaoId, clienteId)

    SS->>DB: SELECT * FROM solicitacao WHERE id = ?
    DB-->>SS: solicitacao

    alt solicitacao não pertence ao cliente autenticado
        SS-->>SC: throw AcessoNegadoException
        SC-->>FE: 403 Forbidden
        FE-->>Cliente: erro de permissão
    else status não é ACEITA
        SS-->>SC: throw TransicaoInvalidaException
        SC-->>FE: 422 Unprocessable Entity { "erro": "Só é possível concluir solicitações ACEITAS" }
        FE-->>Cliente: exibe mensagem de erro
    else tudo válido
        SS->>DB: UPDATE solicitacao SET status = 'CONCLUIDA', concluida_em = NOW()
        DB-->>SS: ok
        SS-->>SC: SolicitacaoResponse { status: CONCLUIDA }
        SC-->>FE: 200 OK
        FE-->>Cliente: exibe "Serviço concluído! Deseja avaliar o profissional?"
        Note right of FE: botão de avaliação só<br/>aparece agora
    end
```

---

## 5. Avaliação de Profissional

Regra de negócio central: o Cliente só pode avaliar se existir uma Solicitação com status `CONCLUÍDA` entre ele e o Profissional. Essa validação é feita no **backend**, não apenas na UI.

```mermaid
sequenceDiagram
    actor Cliente as Cliente
    participant FE as Frontend (React)
    participant SEC as JwtFilter
    participant AC as AvaliacaoController
    participant AS as AvaliacaoService
    participant DB as PostgreSQL

    Cliente->>FE: preenche nota (1-5) e comentário, clica Enviar
    FE->>SEC: POST /avaliacoes<br/>Authorization: Bearer <token><br/>{ profissionalId, nota, comentario }
    SEC->>SEC: valida JWT → extrai clienteId
    SEC->>AC: repassa requisição

    AC->>AS: avaliar(clienteId, profissionalId, nota, comentario)

    Note right of AS: VALIDAÇÃO 1 — existe solicitação concluída?
    AS->>DB: SELECT * FROM solicitacao<br/>WHERE cliente_id = ? AND profissional_id = ?<br/>AND status = 'CONCLUIDA'
    DB-->>AS: resultado

    alt nenhuma solicitação CONCLUÍDA encontrada
        AS-->>AC: throw AvaliacaoNaoPermitidaException
        AC-->>FE: 403 Forbidden { "erro": "Avalie apenas profissionais com serviço concluído" }
        FE-->>Cliente: exibe mensagem de erro
    else solicitação CONCLUÍDA existe
        Note right of AS: VALIDAÇÃO 2 — já avaliou esse profissional nessa solicitação?
        AS->>DB: SELECT * FROM avaliacao WHERE solicitacao_id = ?
        DB-->>AS: resultado

        alt avaliação já existe
            AS-->>AC: throw AvaliacaoDuplicadaException
            AC-->>FE: 409 Conflict { "erro": "Você já avaliou este profissional" }
            FE-->>Cliente: exibe mensagem de erro
        else ainda não avaliou
            AS->>DB: INSERT INTO avaliacao<br/>{ clienteId, profissionalId, solicitacaoId, nota, comentario }
            DB-->>AS: avaliacao salva

            Note right of AS: atualiza nota média do profissional
            AS->>DB: SELECT AVG(nota) FROM avaliacao WHERE profissional_id = ?
            DB-->>AS: nova média
            AS->>DB: UPDATE profissional SET nota_media = ? WHERE id = ?
            DB-->>AS: ok

            AS-->>AC: AvaliacaoResponse { id, nota, comentario }
            AC-->>FE: 201 Created
            FE-->>Cliente: exibe "Avaliação enviada! Obrigado."
        end
    end
```

---

## Resumo das Regras de Negócio Validadas no Backend

| Fluxo | Regra | Onde é validada |
|---|---|---|
| Login | Senha confere com hash BCrypt | `UsuarioService` |
| Busca | Filtro por categoria + bairro | `BuscaService` (query JPA) |
| Abrir solicitação | Profissional destino existe | `SolicitacaoService` |
| Aceitar/Recusar | Apenas o Profissional da solicitação pode responder | `SolicitacaoService` |
| Aceitar/Recusar | Status deve ser `PENDENTE` | `SolicitacaoService` |
| Concluir | Apenas o Cliente da solicitação pode concluir | `SolicitacaoService` |
| Concluir | Status deve ser `ACEITA` | `SolicitacaoService` |
| Avaliar | Deve existir solicitação `CONCLUÍDA` entre os dois | `AvaliacaoService` |
| Avaliar | Não pode avaliar o mesmo profissional duas vezes (por solicitação) | `AvaliacaoService` |

---

*Documento gerado na Etapa 5 do projeto ResolveJá. Próximas etapas: Contratos de API (06) → Implementação (07).*
