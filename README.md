# SalaApp 📅 — Plataforma White Label de Agendamento de Salas (Showcase)

> **Aviso:** Este é um repositório de demonstração (*showcase*). O código-fonte original é mantido em repositório privado por acordo de confidencialidade com o cliente.

---

## 🎯 O Problema

Profissionais autônomos que alugam salas por hora — psicólogos, terapeutas, nutricionistas, massagistas — enfrentam o mesmo gargalo operacional: a agenda vive espalhada entre WhatsApp, papel e planilha, e o dono do espaço não tem visibilidade real de quem está usando qual sala, em qual horário, nem consegue evitar que dois profissionais reservem a mesma sala ao mesmo tempo.

Ao conversar com o dono de um espaço de salas compartilhadas, mapeei três dores que se repetiam com clientes diferentes do mesmo nicho:

1. Cada profissional tinha uma forma diferente de organizar sua agenda pessoal, e nenhuma delas conversava com a disponibilidade real das salas.
2. Clientes finais (pacientes, alunos) precisavam ligar ou mandar mensagem para confirmar um horário, sem saber se ele já estava ocupado.
3. O espaço físico não tinha como bloquear uma sala inteira (manutenção, feriado) sem cancelar manualmente reserva por reserva.

## 💡 A Solução

O **SalaApp** é uma plataforma web *white label* de agendamento que separa três perfis de acesso — Cliente, Profissional e Administrador do espaço — e resolve o problema de concorrência de agenda em duas dimensões simultâneas: disponibilidade da **sala** (definida pelo espaço) e disponibilidade do **profissional** (definida por ele mesmo, em faixas de horário livres). Cada profissional ganha um link público exclusivo (`/agendar/slug-do-profissional`) que os próprios clientes usam para se autoagendar, sem depender de mensagens.

[▶️ **Em breve link do vídeo**]()

---

## 🛠️ Stack Tecnológica

| Camada | Tecnologia | Decisão |
|--------|-----------|---------|
| Backend | Node.js + Express 4 | I/O assíncrono nativo, ecossistema maduro para uma API de agendamento com muitas consultas de disponibilidade em paralelo |
| Banco de dados | MySQL com pool de conexões | Modelo relacional se encaixa bem em reservas com múltiplas FKs (sala, profissional, cliente) e permite índices compostos para checagem de conflito |
| Autenticação | JWT (Access + Refresh Token) + Google OAuth 2.0 | Sessão stateless, renovação silenciosa via cookie httpOnly, sem exigir senha de clientes finais que entram só para agendar |
| Frontend | HTML5 + CSS3 + JavaScript puro (SPA via Fetch API, client-side router) | Zero build step — o cliente do espaço físico não precisa de pipeline de deploy para customizar a instância |
| Upload | Multer com storage local + otimização de imagem | Fotos das salas para o carrossel de seleção, com fallback para ícone quando não há imagem |
| E-mail | Serviço de e-mail transacional desacoplado (interface própria) | Notificações de confirmação, cancelamento e lembrete não bloqueiam a resposta da API — disparadas via `Promise.all` fire-and-forget |
| Hospedagem | Railway | Deploy contínuo via Git, variáveis de ambiente por instância |

---

## 🏗️ Arquitetura do Sistema

```
┌──────────────────────────────────────────────────────────────┐
│                          RAILWAY                              │
│                                                                │
│  ┌──────────────────────┐     ┌───────────────────────────┐  │
│  │   Express (API)       │     │   Static Files             │  │
│  │   /api/reservas/*     │◄───►│   index.html (SPA router)  │  │
│  │   /api/salas/*        │     │   config.json (white label) │  │
│  │   /api/usuarios/*     │     └───────────────────────────┘  │
│  └──────────┬────────────┘                                    │
│             │                                                  │
└─────────────┼──────────────────────────────────────────────────┘
              │
              ├──► MySQL — reservas, salas, usuários, bloqueios
              ├──► /uploads — fotos das salas
              └──► Serviço de e-mail — confirmações e cancelamentos
```

### Três perfis, uma única base de código

O sistema não separa cliente/profissional/admin em aplicações distintas — todos compartilham o mesmo SPA e a mesma tabela `usuarios`, diferenciados por uma coluna `perfil`. Essa escolha trocou isolamento absoluto por velocidade de desenvolvimento e manutenção, já que os três perfis compartilham componentes de UI (calendário, cards de sala, badges de status) que seriam duplicados em bases separadas.

```
usuarios.perfil = 'cliente'        → agenda para si mesmo
usuarios.perfil = 'profissional'   → tem link público, define faixas de horário
usuarios.perfil = 'admin'          → gerencia salas, bloqueios e vê tudo
```

Cada rota de API valida o perfil no middleware antes do controller (`autenticarPerfil('admin')`, `autenticarPerfil('cliente','profissional')`), então a superfície de ataque de "profissional acessando rota de admin" é fechada na camada de roteamento, não no controller.

---

## 🔐 Destaques Técnicos

### 1. Disponibilidade em duas camadas: Sala × Profissional

O ponto mais delicado do domínio: um horário só está disponível se **a sala** estiver livre **e** o **profissional** também estiver, e essas duas disponibilidades são definidas por atores diferentes (o dono do espaço define a sala; o profissional define sua própria agenda).

A primeira versão usava um único par `horario_inicio`/`horario_fim` por profissional — simples, mas incapaz de representar uma agenda real com pausas (ex.: atende de manhã, para às 12h, volta às 14h). A modelagem evoluiu para faixas de horário livres por dia da semana, armazenadas como JSON:

```json
{
  "1": [["07:00","08:00"], ["08:00","09:00"], ["12:00","13:00"]],
  "3": [["14:00","15:00"], ["15:00","16:00"]]
}
```

A geração de slots disponíveis é a interseção de três conjuntos: disponibilidade da sala, faixas do profissional naquele dia da semana, e o que já está ocupado (reservas confirmadas + bloqueios manuais + reservas do mesmo profissional em **outras** salas no mesmo horário). Esse último ponto evita um problema sutil: sem essa checagem cruzada por `profissional_id` (independente da sala), seria possível o mesmo profissional aparecer em duas salas simultaneamente.

```javascript
async function verificarConflitoHorarioProfissional(profissionalId, dataInicio, dataFim, excludeReservaId = null) {
  const [conflitos] = await pool.query(
    `SELECT id FROM reservas WHERE profissional_id = ? AND status NOT IN ('cancelada')
     AND data_inicio < ? AND data_fim > ? AND (? IS NULL OR id != ?)`,
    [profissionalId, dataFim, dataInicio, excludeReservaId, excludeReservaId]
  );
  return conflitos.length > 0;
}
```

### 2. Calendário inteligente sem N+1

Desabilitar no calendário os dias sem nenhum horário livre poderia ser resolvido no frontend, consultando `/horarios-disponiveis` para cada um dos ~30 dias do mês visível — 30 requisições por troca de mês, por sala, por profissional. Em vez disso, existe um endpoint dedicado (`/reservas/dias-disponiveis`) que resolve a disponibilidade do mês inteiro em uma única chamada, reaproveitando as mesmas queries de conflito internamente e retornando apenas a lista de dias com pelo menos um slot livre. O frontend troca de mês e dispara uma única requisição, nunca uma por dia.

### 3. Padrão "sacola" para agendamento múltiplo com falha parcial

Profissionais recorrentes (ex.: pacote de 4 sessões semanais) precisavam agendar vários horários — possivelmente em dias e salas diferentes — em uma única ação, sem que a falha de um horário específico (já ocupado entre a seleção e a confirmação) derrubasse os demais.

A solução foi tratar o carrinho de horários como uma lista de operações independentes, processadas sequencialmente no backend, cada uma com seu próprio resultado:

```javascript
for (const r of listaReservas) {
  const conflito = await verificarConflito(r.sala_id, r.data_inicio, r.data_fim);
  if (conflito) { resultados.push({ status: 'erro', mensagem: 'Horário ocupado' }); continue; }
  // ...persiste e segue para o próximo item da sacola
}
```

A resposta final agrega sucessos e falhas (`{ sucessos: 3, falhas: 1, resultados: [...] }`), e o frontend usa esse retorno para dar feedback específico de qual item não pôde ser confirmado — em vez de um `200` ou `409` genérico para o lote inteiro.

### 4. Cancelamento sem login via token opaco

Clientes recebem por e-mail um link de cancelamento que não exige autenticação: `/reservas/cancelar/:token`. O token é um UUID v4 gerado no momento da criação da reserva e armazenado na própria linha (`token_cancelamento`), nunca reaproveitado entre reservas. Isso elimina a fricção de "esqueci minha senha" para uma ação simples, sem abrir uma rota de escrita não autenticada de fato — o token funciona como uma capability de posse único, e a query de cancelamento já filtra `status != 'cancelada'` para ser idempotente.

### 5. White label via config.json

Assim como em outros projetos da mesma linha, a identidade visual de cada instância é externalizada em runtime:

```json
{
  "appName": "Espaço Aurora",
  "corPrimaria": "#4F46E5",
  "corSecundaria": "#7C3AED",
  "logo": "/assets/logo.png",
  "googleClientId": "..."
}
```

O `loadConfig()` aplica as variáveis CSS e troca textos/logo antes da primeira renderização da SPA, permitindo reutilizar o mesmo build para múltiplos espaços físicos sem qualquer alteração de código.

### 6. Renovação de sessão silenciosa com retry transparente

A camada `api()` do frontend encapsula toda chamada HTTP: se a API responde `401`, ela tenta renovar o token via `/auth/refresh` (cookie httpOnly) e refaz a requisição original **uma única vez** — o parâmetro `retry` evita loop infinito caso o refresh também falhe. O restante da aplicação nunca lida com expiração de token diretamente:

```javascript
async function api(method, path, body = null, retry = true) {
  const res = await fetch(`${CONFIG.apiUrl}${path}`, opts);
  if (res.status === 401 && retry && !isAuthRoute) {
    const ok = await refreshToken();
    if (ok) return api(method, path, body, false);
    logout(); return null;
  }
  return res;
}
```

### 7. Carrossel de seleção como parte do formulário, não como galeria

No link público do profissional, o carrossel de fotos de cada sala não é decorativo — o clique na sala inteira (foto, nome ou descrição) seleciona aquela opção para o agendamento, com o carrossel interno navegável de forma independente (`event.stopPropagation()` nos botões de seta) para não disparar a seleção sala ao trocar de foto. A ordem "escolha a sala → escolha a data → escolha o horário" também recarrega a disponibilidade de dias assim que a sala muda, já que cada sala tem sua própria janela de disponibilidade.

---

## 📁 Estrutura do Projeto

```
salaflow/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   └── database.js         # Pool MySQL
│   │   ├── controllers/
│   │   │   ├── reservasController.js   # Conflitos, sacola, dias-disponíveis
│   │   │   ├── salasController.js      # CRUD de salas + bloqueios
│   │   │   └── usuariosController.js   # Perfil, faixas de horário, bloqueios do profissional
│   │   ├── middlewares/
│   │   │   └── auth.js             # autenticar / autenticarPerfil / renovarToken
│   │   ├── services/
│   │   │   └── emailService.js     # Confirmação, notificação, cancelamento
│   │   ├── routes/
│   │   │   └── index.js
│   │   └── app.js
│   └── uploads/                    # Fotos das salas (gitignored)
├── frontend/
│   ├── assets/
│   │   ├── js/
│   │   │   ├── app.js              # Router SPA, calendário, sessão
│   │   │   ├── salas.js            # Fluxo de agendamento (cliente/profissional/link público)
│   │   │   └── dashboards.js       # Dashboards dos três perfis
│   │   └── css/
│   │       └── main.css
│   ├── config.json                 # Identidade visual da instância
│   └── index.html
└── database/
    └── schema.sql
```

---

## 🗄️ Modelo de Dados (simplificado)

```
usuarios
  id, nome, email, telefone, perfil, nome_fantasia, slug_profissional
  horario_inicio, horario_fim, dias_trabalho, faixas_horario (JSON)
  profissional_vinculado_id, ativo, google_id

salas
  id, nome, descricao, capacidade, fotos (JSON), ativo

disponibilidade_salas
  id, sala_id → salas, dia_semana, hora_inicio, hora_fim, ativo

reservas
  id, sala_id → salas, profissional_id → usuarios, cliente_id → usuarios
  data_inicio, data_fim, status, origem, observacoes, token_cancelamento

bloqueios                    -- bloqueio manual de uma sala (admin)
  id, sala_id → salas, data_inicio, data_fim, motivo

bloqueios_profissional       -- bloqueio manual da agenda do profissional
  id, profissional_id → usuarios, data_inicio, data_fim, motivo
```

---

## 🌊 Fluxos Principais

### Fluxo do Cliente (via link do profissional)

```
Link do profissional → /agendar/slug
    │
    ├─ Já tem conta ──────────────────────────► login inline
    └─ Conta nova → cadastro rápido (ou Google OAuth)
                         │
                    Escolhe sala (carrossel)
                         │
                    Calendário mostra só dias com horário livre
                    naquela sala + para aquele profissional
                         │
                    Escolhe um ou mais horários (sacola)
                         │
                    [Confirmar todos]
                         │
                    Reservas criadas individualmente
                    E-mail de confirmação para cliente e profissional
```

### Fluxo do Profissional

```
Dashboard → define faixas de horário por dia da semana
         → gera link público automaticamente (slug)
         → pode bloquear períodos pessoais (férias, compromissos)
         → agenda diretamente para um cliente (com ou sem conta)
         → vê reservas futuras e histórico, filtrado por cliente
```

### Fluxo do Administrador do Espaço

```
Dashboard → cria/edita salas (fotos, capacidade, disponibilidade semanal)
         → bloqueia uma sala específica ou todas de uma vez (feriado, manutenção)
         → vê todas as reservas do espaço, filtráveis por sala e data
         → gerencia usuários (ativa/desativa profissionais e clientes)
```

---

## 🔒 Camadas de Segurança

```
Requisição HTTP
      │
      ▼
[Rate Limiter]     → limite de tentativas em /auth/*
      │
      ▼
[CORS + credentials] → cookies httpOnly restritos à origem da instância
      │
      ▼
[autenticar]        → valida JWT, dispara refresh transparente no frontend
      │
      ▼
[autenticarPerfil]  → cliente/profissional/admin isolados por rota
      │
      ▼
Controller          → checagem de conflito antes de qualquer escrita em `reservas`
      │
      ▼
[Multer]            → whitelist de MIME type para fotos de sala, limite de 5MB
```

---

## 👨‍💻 Autor

**Antonio Kerr**

- [LinkedIn](https://www.linkedin.com/in/antoniokerr/)
- [GitHub](https://github.com/tonykerr07)
