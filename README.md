# 📡 GroupRadar

Monitore grupos específicos do Telegram e receba um alerta sempre que aparecer uma mensagem que te interessa.

O GroupRadar fica de olho nos grupos e canais que você escolher, compara cada nova mensagem com uma lista de regras (palavras-chave, frases ou expressões regulares) e, quando encontra algo, te avisa num chat privado com o trecho da mensagem e o link direto para ela.

> **Exemplos de uso:** promoções em grupos de ofertas, vagas de emprego com determinada stack, menções à sua empresa ou produto, anúncios de ingressos, avisos em grupos de condomínio.

---

## ✨ Funcionalidades

- **Vários grupos ao mesmo tempo:** monitora quantos grupos e canais você quiser, inclusive os privados de que você já participa.
- **Regras flexíveis:** palavras-chave, frases exatas, regex e termos de exclusão.
- **Regras por grupo:** cada grupo pode ter seu próprio conjunto de regras.
- **Alertas com contexto:** autor, grupo, horário, trecho destacado e link para a mensagem original.
- **Anti-spam:** deduplicação e cooldown configurável para não repetir alertas iguais.
- **Configuração em arquivo:** tudo em um `config.yaml` versionável.
- **Pronto para Docker:** roda 24/7 num servidor pequeno.

---

## 🧠 Como funciona

```
┌────────────────────┐     novas mensagens     ┌──────────────┐   match?   ┌──────────────────┐
│ Grupos monitorados │ ──────────────────────▶ │  GroupRadar  │ ─────────▶ │ Bot de alertas   │ ──▶ você 📱
│ (sua conta)        │      (MTProto)          │  (matcher)   │            │ (Bot API)        │
└────────────────────┘                         └──────────────┘            └──────────────────┘
```

O GroupRadar usa **dois componentes**:

1. **Userbot (sua conta do Telegram):** lê as mensagens dos grupos via [Telethon](https://github.com/LonamiWebs/Telethon). Isso é necessário porque bots comuns só leem grupos em que foram adicionados e, mesmo assim, normalmente não veem todas as mensagens por causa do *privacy mode*. Com a sua conta, dá para monitorar qualquer grupo de que você já participa, sem precisar ser admin.
2. **Bot de alertas:** um bot criado no [@BotFather](https://t.me/BotFather) que envia as notificações para você (ou para um grupo de alertas).

> ⚠️ A conta usada como userbot **apenas lê** as mensagens. O GroupRadar não envia nada nos grupos monitorados.

---

## 🚀 Começando

### Pré-requisitos

- Python 3.11+
- Uma conta do Telegram que participe dos grupos que você quer monitorar
- `API_ID` e `API_HASH`, obtidos em [my.telegram.org](https://my.telegram.org) → *API development tools*
- Um token de bot, obtido no [@BotFather](https://t.me/BotFather) com `/newbot`

### Instalação

```bash
git clone https://github.com/<seu-usuario>/telegram-groupradar.git
cd telegram-groupradar

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
cp config.example.yaml config.yaml
```

### Variáveis de ambiente (`.env`)

```dotenv
# Credenciais da sua conta (my.telegram.org)
TG_API_ID=1234567
TG_API_HASH=0123456789abcdef0123456789abcdef

# Bot que envia os alertas (@BotFather)
ALERT_BOT_TOKEN=123456789:AAH...

# Chat que recebe os alertas (seu user ID ou o ID de um grupo)
ALERT_CHAT_ID=123456789

# Nome do arquivo de sessão do Telethon
SESSION_NAME=groupradar
```

> 💡 Para descobrir seu `ALERT_CHAT_ID`, mande uma mensagem para o seu bot e rode `python -m groupradar whoami`, ou use o [@userinfobot](https://t.me/userinfobot). Lembre-se de dar `/start` no bot antes, senão ele não consegue te mandar mensagens.

### Primeira execução (login)

```bash
python -m groupradar login
```

O Telegram vai mandar um código para a sua conta (e pedir a senha de 2FA, se você usar). Depois disso, o arquivo `groupradar.session` fica salvo localmente e o login não é pedido de novo.

### Descobrindo os IDs dos grupos

```bash
python -m groupradar list-chats
```

```
-1001234567890  Promoções Tech BR        (supergroup)
-1009876543210  Vagas Python Remoto      (channel)
-4001122334     Condomínio Bloco B       (group)
```

### Rodando

```bash
python -m groupradar run
```

---

## ⚙️ Configuração (`config.yaml`)

```yaml
# Regras aplicadas a todos os grupos (opcional)
global_rules:
  - name: "Meu nome"
    keywords: ["fulano da silva", "@meuusuario"]

groups:
  - id: -1001234567890
    alias: "Promoções Tech"
    rules:
      - name: "Monitor bom e barato"
        keywords: ["monitor", "ultrawide"]
        all_of: ["cupom"]              # precisa conter TODOS estes termos
        exclude: ["usado", "defeito"]  # ignora se contiver algum destes
      - name: "SSD com preço"
        regex: "ssd.*(1\\s?tb).*r\\$\\s?\\d+"

  - id: -1009876543210
    alias: "Vagas Python"
    rules:
      - name: "Vaga remota sênior"
        keywords: ["python"]
        all_of: ["remoto"]
        any_of: ["sênior", "senior", "sr"]

options:
  case_sensitive: false     # ignora maiúsculas/minúsculas
  normalize_accents: true   # "promocao" casa com "promoção"
  include_edits: true       # também verifica mensagens editadas
  cooldown_seconds: 300     # não repete alerta da mesma regra+grupo nesse intervalo
  snippet_length: 280       # tamanho do trecho enviado no alerta
```

### Referência das regras

| Campo      | Tipo         | Descrição                                                    |
|------------|--------------|--------------------------------------------------------------|
| `name`     | string       | Nome da regra, exibido no alerta                             |
| `keywords` | lista        | Dispara se **qualquer** termo aparecer                       |
| `any_of`   | lista        | Exige ao menos **um** destes termos (além de `keywords`)     |
| `all_of`   | lista        | Exige **todos** estes termos                                 |
| `exclude`  | lista        | Descarta a mensagem se **qualquer** termo aparecer           |
| `regex`    | string       | Expressão regular (Python `re`) aplicada ao texto            |
| `from`     | lista        | Só considera mensagens desses usuários (`@username` ou ID)   |

Uma regra dispara quando **todas** as condições definidas nela são satisfeitas. Se uma mensagem casar com mais de uma regra, você recebe um único alerta listando todas elas.

---

## 🔔 Exemplo de alerta

```
📡 GroupRadar — Monitor bom e barato

👥 Promoções Tech
👤 @joaozinho · 15/10/2026 14:32

"...Monitor LG 29" ULTRAWIDE por R$ 899 com o CUPOM TECH10, só hoje..."

🔗 Abrir mensagem
```

---

## 🐳 Docker

```bash
# Faça o login uma vez, de forma interativa, para gerar a sessão
docker compose run --rm groupradar login

# Depois rode em segundo plano
docker compose up -d
docker compose logs -f
```

O `docker-compose.yml` monta `./data` como volume para persistir o arquivo de sessão e o estado de deduplicação.

---

## 🛠️ Comandos

| Comando                              | Descrição                                          |
|--------------------------------------|----------------------------------------------------|
| `python -m groupradar login`         | Faz o login da conta e salva a sessão              |
| `python -m groupradar list-chats`    | Lista grupos e canais com seus IDs                 |
| `python -m groupradar whoami`        | Mostra os IDs úteis para configurar o `.env`       |
| `python -m groupradar check-config`  | Valida o `config.yaml` sem conectar                |
| `python -m groupradar test "texto"`  | Testa quais regras casariam com um texto           |
| `python -m groupradar run`           | Inicia o monitoramento                             |

---

## 📁 Estrutura do projeto

```
telegram-groupradar/
├── groupradar/
│   ├── __main__.py      # CLI
│   ├── client.py        # Conexão Telethon (userbot)
│   ├── matcher.py       # Avaliação das regras
│   ├── notifier.py      # Envio de alertas via Bot API
│   ├── config.py        # Carregamento e validação do config.yaml
│   └── dedup.py         # Cooldown e deduplicação
├── tests/
├── config.example.yaml
├── .env.example
├── docker-compose.yml
├── Dockerfile
└── requirements.txt
```

---

## 🔒 Segurança e privacidade

- O arquivo `*.session` **dá acesso total à sua conta do Telegram.** Nunca faça commit dele; ele já está no `.gitignore`. Trate-o como uma senha.
- Não compartilhe seu `API_HASH` nem o token do bot.
- Todo o processamento é local: nenhuma mensagem sai da sua máquina, exceto os alertas enviados para o seu próprio chat.
- Respeite a privacidade dos membros dos grupos e os [Termos de Serviço do Telegram](https://telegram.org/tos). Use o GroupRadar só em grupos dos quais você participa legitimamente.
- Uma conta secundária reduz o risco caso algo dê errado.

---

## 🗺️ Roadmap

- [ ] Comandos no bot para adicionar e remover regras pelo próprio Telegram (`/add`, `/rules`, `/pause`)
- [ ] OCR em imagens
- [ ] Envio dos alertas também por webhook, Discord e e-mail
- [ ] Resumo diário dos matches
- [ ] Recarregar o `config.yaml` sem reiniciar o processo

---

## 🤝 Contribuindo

1. Faça um fork do projeto
2. Crie uma branch: `git checkout -b feat/minha-feature`
3. Rode os testes: `pytest`
4. Abra um Pull Request

---

## 📄 Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE) para mais detalhes.
