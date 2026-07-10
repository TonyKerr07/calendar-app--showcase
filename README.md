# Sala App 📅 — Sistema Multi-perfil de Agendamento de Salas (Showcase)

> **Aviso:** Este é um repositório de demonstração (*showcase*). O código foi desenvolvido para resolver um problema real de um profissional de saúde, otimizando o fluxo de locação de salas em espaços compartilhados.

## 🎯 O Problema

Muitos profissionais independentes (como quiropraxistas, psicólogos e fisioterapeutas) não possuem consultório próprio e alugam salas por hora em clínicas ou coworkings. 

O fluxo antigo do meu cliente era exaustivo:
1. Ligar para a clínica para verificar quais horários estavam livres na sala.
2. Mandar mensagem para o paciente oferecendo as opções.
3. Esperar o paciente responder.
4. Ligar novamente para a clínica para confirmar a reserva (torcendo para ninguém ter alugado a sala nesse meio tempo).

## 💡 A Solução

Desenvolvi uma plataforma web (SPA) com **3 perfis de acesso** (Administrador do Espaço, Profissional e Cliente Final). A aplicação cruza, em tempo real, a disponibilidade física das salas com a agenda de trabalho do profissional, permitindo que o paciente faça o autoagendamento sem nenhum atrito.

*(O vídeo demonstrando o fluxo completo será adicionado aqui em breve)*

---

## 🛠️ Stack Tecnológica

- **Backend:** Node.js 18+ com Express
- **Banco de Dados:** MySQL (com modelagem relacional para salas, bloqueios e usuários)
- **Frontend:** SPA (Single Page Application) responsiva em Vanilla JavaScript, HTML5 e CSS3 puro.
- **Autenticação e Segurança:** JWT (Access e Refresh Tokens), Google OAuth 2.0, bcryptjs, Helmet e Rate Limiting.
- **Notificações:** Integração com Gmail API (Service Accounts) para envio de e-mails transacionais.

---

## 🚀 Destaques da Arquitetura e Lógica de Negócio

### 1. Motor de Resolução de Conflitos (Disponibilidade)
A maior complexidade do backend foi o algoritmo de busca de horários. Para exibir um horário livre para o paciente, o sistema calcula dinamicamente:
- A grade de funcionamento padrão da clínica (ex: Seg a Sex, 08h às 18h).
- A grade de atendimento específica do profissional.
- Reservas já confirmadas na sala desejada.
- Reservas do profissional em *outras* salas no mesmo horário.
- Bloqueios manuais de manutenção feitos pelo admin.

### 2. Automação Multi-Perfil
A arquitetura foi pensada para atender a todos os envolvidos no ecossistema de uma clínica:
- **Espaço (Admin):** Cadastra salas, gerencia limites de capacidade e bloqueia dias para manutenção ou horários que possam ter sido agendados por outros meios.
- **Profissional:** Pode agendar salas diretamente pelo painel para seus pacientes ou gerar um link público personalizado (`/agendar/nome-do-profissional`) para o paciente praticar o autoagendamento.
- **Cliente:** Acessa o link, faz login (email ou Google) e escolhe um horário livre.

### 3. Sistema de Notificações Transacionais
Para garantir que ninguém perca o horário e que a clínica tenha controle de quem está no prédio, o sistema dispara e-mails automatizados usando a **Gmail API** com templates em HTML a cada evento. Quando uma reserva é feita, o Paciente, o Profissional e a Recepção da Clínica recebem confirmações instantâneas.

### 4. Segurança e Sessão
Implementado fluxo moderno de autenticação com **Access Tokens** de vida curta e **Refresh Tokens** persistidos em cookies `httpOnly` para manter os usuários logados com segurança sem expor credenciais no LocalStorage. O sistema também permite login simplificado via Google (OAuth 2.0).

---

## 👨‍💻 Autor

**Antonio Kerr**
- [LinkedIn](https://www.linkedin.com/in/antoniokerr/)
- [GitHub](https://github.com/tonykerr07)
