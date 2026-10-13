# Chatbot de clima no Telegram (n8n + OpenWeather)

Bot do Telegram feito no n8n que responde a temperatura atual de uma cidade brasileira. Você manda o nome da cidade e ele consulta a OpenWeather e devolve algo como:

```
🌤️ A temperatura em Belo Horizonte é de 25°C.
```

Se não encontrar a cidade:

```
❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## Fluxo

```
Telegram Trigger → Normalizar entrada → Consultar OpenWeather → Resposta válida?
                                                                  ├─ sim → Formatar resposta → Enviar temperatura
                                                                  └─ não → Enviar erro
```

- **Normalizar entrada**: pega o texto da mensagem e salva em `queue`. Tira espaços sobrando, acentos e maiúsculas, troca `/` e ` - ` por vírgula e adiciona `,br` no final se não tiver. Assim, `São Paulo , SP` vira `sao paulo,sp,br`.
- **Consultar OpenWeather**: faz `GET /data/2.5/weather` com `q`, `units=metric`, `lang=pt_br` e `appid` vindo de `$env.OPENWEATHER_API_KEY`. O valor de `queue` vai no parâmetro `q`, que é o nome que a API usa. Deixei "Never Error" ligado pra que um 404 não quebre a execução e caia no IF.
- **Resposta válida?**: confere se o status é 200 e se a resposta tem `main.temp` e `name`.
- **Formatar resposta**: arredonda a temperatura e monta a mensagem.

## Variáveis

- `OPENWEATHER_API_KEY`: chave da OpenWeather, lida como variável de ambiente pelo n8n.
- `TELEGRAM_BOT_TOKEN`: token do bot gerado no @BotFather, cadastrado como credencial dentro do n8n.

Nenhuma chave está no repositório. Use o `.env.example` como modelo.

## Rodando o n8n com Docker

O Telegram só manda webhook pra HTTPS, então localmente eu uso o ngrok:

```bash
cp .env.example .env      # preencha com seus valores
ngrok http 5678           # copie a URL https para WEBHOOK_URL no .env
docker compose up -d
```

Depois é só acessar `http://localhost:5678`.

O `docker-compose.yml` já define `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Sem isso, as versões mais novas do n8n bloqueiam o `$env` e a API retorna 401.

Se for rodar sem Docker, exporte as mesmas variáveis antes do `n8n start`.

## Importando o workflow

1. No n8n, crie um workflow novo, vá em **⋯ → Import from File** e selecione `workflow-chatbot-telegram.json`.
2. No nó **Telegram Trigger**, crie uma credencial **Telegram API** colando o token do BotFather.
3. Selecione essa mesma credencial nos nós **Enviar temperatura** e **Enviar erro**.
4. Ative o workflow.

A chave da OpenWeather não precisa de credencial, porque vem do ambiente. Se trocar a chave, reinicie o n8n. Uma chave recém-criada pode demorar um pouco pra ativar e, enquanto isso, a API responde 401.

## Testando

Mande uma mensagem pro bot:

- `São Paulo,SP,BR` → 🌤️ A temperatura em São Paulo é de XX°C.
- `Belo Horizonte,MG,BR` → 🌤️ A temperatura em Belo Horizonte é de XX°C.
- `Presidente Prudente - SP` → 🌤️ A temperatura em Presidente Prudente é de XX°C.
- `Cidadeinexistente,XX,BR` → mensagem de erro

A OpenWeather só usa a UF pra cidades dos EUA, então pro Brasil a busca é por cidade + país. Em cidades com o mesmo nome em estados diferentes, ela pode trazer a maior.

## Arquivos

- `workflow-chatbot-telegram.json`: workflow exportado, sem credenciais
- `docker-compose.yml`: n8n local
- `.env.example`: modelo das variáveis
