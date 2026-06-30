# Documento de Requisitos — ResolveJá (Marketplace de Serviços Locais)

## 1. Visão Geral

**ResolveJá** é uma plataforma web onde clientes encontram profissionais autônomos de serviços (eletricista, diarista, pedreiro, manicure, professor particular, etc.) na cidade de Marília-SP, filtrando por categoria e bairro/região. O profissional monta um perfil com portfólio de trabalhos já realizados e recebe avaliações apenas de clientes que efetivamente confirmaram um serviço com ele.

## 2. Atores

| Ator | Descrição |
|---|---|
| Cliente | Usuário que busca e contrata profissionais |
| Profissional | Usuário que oferece serviços, monta portfólio e responde solicitações |
| Administrador | Usuário interno que modera cadastros e denúncias (escopo mínimo no MVP) |

## 3. Requisitos Funcionais (RF)

### Autenticação e Perfil
- RF01 — O sistema deve permitir cadastro e login de Cliente e Profissional (e-mail + senha, papéis distintos)
- RF02 — O Profissional deve poder criar/editar um perfil com: categoria(s) de serviço, descrição, bairro/região de atuação em Marília, fotos de trabalhos realizados (portfólio)
- RF03 — O Cliente deve poder criar/editar um perfil básico (nome, contato)

### Busca
- RF04 — O sistema deve permitir buscar profissionais filtrando por categoria de serviço E bairro/região
- RF05 — O sistema deve listar profissionais com nota média de avaliação e exibir o perfil completo ao clicar

### Solicitação de Serviço (núcleo do fluxo de negócio)
- RF06 — O Cliente deve poder abrir uma "Solicitação de Serviço" para um Profissional, descrevendo brevemente a necessidade
- RF07 — A Solicitação possui status: `PENDENTE` → `ACEITA` ou `RECUSADA` (pelo profissional) → `CONCLUÍDA` ou `CANCELADA`
- RF08 — Ao aceitar, o Profissional escolhe a forma de contato disponibilizada ao cliente: WhatsApp/telefone, chat interno, ou ambos
- RF09 — Se "chat interno" for escolhido, o sistema deve permitir troca de mensagens simples entre Cliente e Profissional vinculada àquela Solicitação
- RF10 — Apenas o Cliente da Solicitação pode marcá-la como `CONCLUÍDA`

### Avaliações
- RF11 — O Cliente só pode avaliar (nota 1-5 + comentário) um Profissional se existir Solicitação com status `CONCLUÍDA` entre os dois
- RF12 — O Profissional pode responder publicamente a uma avaliação recebida
- RF13 — A nota média do Profissional é recalculada automaticamente a cada nova avaliação

### Administração (mínimo)
- RF14 — O Administrador pode desativar perfis denunciados

## 4. Requisitos Não Funcionais (RNF)

- RNF01 — Backend em Java com Spring Boot (REST API)
- RNF02 — Frontend em React + TypeScript
- RNF03 — Banco de dados relacional (PostgreSQL)
- RNF04 — Autenticação via JWT (stateless)
- RNF05 — Senhas armazenadas com hash (BCrypt)
- RNF06 — API documentada (Swagger/OpenAPI)
- RNF07 — Aplicação containerizada com Docker (deploy facilitado)
- RNF08 — Tempo de resposta de busca abaixo de 1s para até 1000 profissionais cadastrados (meta de portfólio, não de produção)

## 5. Regras de Negócio Importantes

- RN01 — Avaliação é bloqueada por regra de negócio no backend, não só na UI (validação de Solicitação concluída)
- RN02 — Um Profissional não pode avaliar a si mesmo nem outro Profissional
- RN03 — Uma Solicitação só pode ser cancelada pelo Cliente enquanto estiver `PENDENTE` ou `ACEITA` (não após `CONCLUÍDA`)
- RN04 — Forma de contato (WhatsApp/chat) é decidida pelo Profissional no momento de aceitar, não pelo Cliente na abertura

## 6. Fora do Escopo do MVP (trabalhos futuros)

- Pagamento integrado dentro da plataforma
- Geolocalização real (mapa) — bairro será um campo de texto/seleção, não coordenadas
- Notificações push/e-mail em tempo real
- App mobile nativo

---
*Próxima etapa: Casos de Uso (diagrama UML + descrição detalhada de cada um)*
