# 🤖 Bot de Cobrança Automático via WhatsApp

**n8n + Evolution API + Google Sheets + Docker**

Este projeto implementa uma **Régua de Cobrança automática via WhatsApp**, ideal para mensalidades recorrentes como academias, artes marciais (ex: Jiu-Jitsu), escolas, serviços por assinatura ou qualquer cobrança mensal simples.

A automação:

* lê diariamente uma planilha do Google Sheets,
* identifica quem deve ser cobrado,
* calcula o tempo até o vencimento (ou atraso),
* e envia mensagens personalizadas via WhatsApp.

Tudo isso usando **stack 100% gratuita e open source**, rodando em **Docker**.

---

## 🎯 Objetivo do Projeto

Automatizar cobranças recorrentes de forma:

* simples,
* controlável por planilha,
* sem expor credenciais pessoais,
* evitando erros comuns (mensagem duplicada, cobrança fora do mês, bugs em iPhone).

O projeto foi pensado para **pessoas técnicas e não técnicas**:
o dono do negócio mexe na planilha, o robô faz o resto.

---

## 🧠 Visão Geral da Arquitetura

Fluxo lógico do sistema:

```
Agendamento diário
      ↓
Leitura Google Sheets
      ↓
Filtro de pendências
      ↓
Cálculo de dias até o vencimento
      ↓
Roteamento por cenário
      ↓
Envio via WhatsApp
```

Componentes:

* **n8n** → cérebro da automação (workflows)
* **Evolution API (v2)** → gateway WhatsApp Web
* **Google Sheets** → banco de dados operacional
* **PostgreSQL** → banco interno do n8n
* **Redis** → cache e fila da Evolution API
* **Docker** → empacotamento e portabilidade

Todos os serviços rodam em containers isolados, comunicando-se pela mesma rede Docker.

---

## 🛠️ Tech Stack

* n8n
* Evolution API (v2.1.1 estável)
* PostgreSQL
* Redis
* Google Sheets API
* Google Drive API
* Docker + Docker Compose

---

## 📋 Pré-requisitos

* Docker Desktop instalado

  * No Windows, recomenda-se uso com **WSL2**
* Conta Google (para Google Cloud e Sheets)
* Um número de WhatsApp (Pessoal ou Business)
* Celular para leitura do QR Code

---

## 📁 Estrutura de Pastas

```
bot-cobranca/
├── docker-compose.yml
├── n8n_data/
├── postgres_data/
├── redis_data/
└── evolution_store/
```

Essas pastas garantem persistência dos dados entre reinícios.

---

## 🚀 Instalação (Windows / PowerShell)

### 1️⃣ Preparando o Ambiente

```powershell
New-Item -ItemType Directory -Force -Path "C:\bot-cobranca"
cd C:\bot-cobranca

New-Item -ItemType Directory -Force -Path "n8n_data"
New-Item -ItemType Directory -Force -Path "postgres_data"
New-Item -ItemType Directory -Force -Path "redis_data"
New-Item -ItemType Directory -Force -Path "evolution_store"

New-Item -ItemType File -Name "docker-compose.yml"
```

---

### 2️⃣ docker-compose.yml

> ⚠️ Atenção: versões e variáveis foram escolhidas para **evitar bugs conhecidos no iOS**.

```yaml
version: '3.9'

networks:
  evolution_net:
    driver: bridge

services:
  postgres:
    image: postgres:15-alpine
    container_name: evolution-postgres
    restart: unless-stopped
    networks:
      - evolution_net
    environment:
      POSTGRES_DB: evolution
      POSTGRES_USER: evolution
      POSTGRES_PASSWORD: evolution
    volumes:
      - evolution_postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U evolution"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: evolution-redis
    restart: unless-stopped
    networks:
      - evolution_net
    command: ["redis-server", "--requirepass", "CRIE SUA SENHA SEGURA AQUI", "--appendonly", "no", "--bind", "0.0.0.0"]
    volumes:
      - evolution_redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "INSIRA SUA SENHA SEGUR AQUI", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  evolution-api:
    image: atendai/evolution-api:latest
    container_name: evolution-api
    restart: unless-stopped
    networks:
      - evolution_net
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "8080:8080"
    environment:
      # =====================
      # SERVIDOR
      # =====================
      SERVER_URL: http://localhost:8080
      AUTHENTICATION_TYPE: apikey
      AUTHENTICATION_API_KEY: sua_api_key_aqui
      
      # =====================
      # SESSÃO (CORREÇÃO AQUI)
      # =====================
      # Removemos a versão fixa para evitar o erro 401/device_removed
      CONFIG_SESSION_PHONE_CLIENT: "Insira O nome da sua instancia aqui"
      CONFIG_SESSION_PHONE_NAME: "Chrome"
      CONFIG_SESSION_PHONE_VERSION: 2.3000.1031221906
      
      # =====================
      # BANCO DE DADOS (CORREÇÃO DE PERFORMANCE)
      # =====================
      DATABASE_ENABLED: "true"
      DATABASE_PROVIDER: postgresql
      DATABASE_CONNECTION_URI: postgresql://evolution:evolution@postgres:5432/evolution
      # Desligamos o sync pesado para não derrubar a conexão inicial
      DATABASE_SAVE_DATA_INSTANCE: "true"
      DATABASE_SAVE_DATA_NEW_MESSAGE: "false"
      DATABASE_SAVE_DATA_CONTACTS: "false" 
      DATABASE_SAVE_DATA_CHATS: "false"

      # =====================
      # CACHE REDIS
      # =====================
      CACHE_REDIS_ENABLED: "true"
      CACHE_REDIS_URI: redis://:INSIRA SUA SENHA SEGURA AQUI@evolution-redis:6379/0?family=4
      CACHE_REDIS_PREFIX_KEY: "evolution"
      CACHE_REDIS_SAVE_INSTANCES: "false"
      CACHE_LOCAL_ENABLED: "false"

      # =====================
      # OUTROS
      # =====================
      DEL_INSTANCE: "false"
      WEBSOCKET_ENABLED: "false"

    volumes:
      - evolution_instances:/evolution/instances

volumes:
  evolution_postgres_data:
  evolution_instances:
  evolution_redis_data:
```

Suba tudo:

```powershell
docker compose up -d
```

Acessos:

* n8n → [http://localhost:5678](http://localhost:5678)
* Evolution API → [http://localhost:8080/manager](http://localhost:8080/manager)

---

## 📱 Conectando o WhatsApp

1. Acesse o painel da Evolution API
2. Informe a API Key
3. Crie uma instância
4. Gere o QR Code
5. Leia com o celular (WhatsApp Web)

---

## 🔐 Google Sheets (Service Account)

Para leitura segura da planilha, usamos **Service Account** um “crachá digital” do robô.

### APIs obrigatórias

* Google Sheets API
* Google Drive API

### Passos resumidos

1. Criar projeto no Google Cloud
2. Criar Service Account
3. Gerar chave **JSON**
4. Guardar o arquivo com segurança

---

## 📊 Estrutura da Planilha

### Aba 1: `Cadastro` (uso humano)

```
Nome | Telefone | Dia | Vencimento | Valor
```

> Essa aba **não é lida pelo robô**.

---

### Aba 2: `Financeiro` (uso do robô)

Colunas **obrigatoriamente nessa ordem**:

```
A Nome
B Telefone
C Dia
D Mes
E Ano
F Valor
G Status
```

Exemplo de Status:

* `Pendente`
* `Pago`

---

### Compartilhamento (CRUCIAL)

* Copie o `client_email` do JSON
* Compartilhe a planilha com esse e-mail
* Permissão: **Editor**

Sem isso, o n8n retorna erro `403 Forbidden`.

---

## 🧠 Workflow no n8n

### Fluxo Geral

```
Schedule Trigger
→ Google Sheets
→ IF (Status = Pendente)
→ Edit Fields (Diferença de dias)
→ Switch
→ HTTP Request (envio)
```

---

### Cálculo de Dias

No **Edit Fields**:

```js
{{ $json.Dia - $now.day }}
```

Variável: `DiferencaDias`

---

### Switch (Régua de Cobrança)

| Diferença | Cenário          |
| --------- | ---------------- |
| 5         | Aviso antecipado |
| 2         | Aviso próximo    |
| 0         | Vence hoje       |
| < 0       | Em atraso        |

---

## 💬 Templates de Mensagem

> ⚠️ Para iOS: `"linkPreview": false` é **obrigatório**.

```json
{
  "number": "{{ $('If').item.json.Telefone }}",
  "text": "Olá {{ $('If').item.json.Nome }}! Sua mensalidade de R$ {{ $('If').item.json.Valor }} vence em 5 dias.",
  "linkPreview": false
}
```

Usamos **referência absoluta** (`$('If')`) para evitar perda de contexto.

---

## 🌐 HTTP Request (Evolution API)

Configuração padrão:

* Method: `POST`
* URL:

```
http://evolution-api:8080/message/sendText/NomeDaInstancia
```

* Header:

```
apikey: SUA_API_KEY
```

* Content-Type: `JSON`

---

## 🧯 Troubleshooting

### QR Code não aparece

```powershell
docker compose logs -f evolution-api
```

### Mensagem vazia no iPhone

* Garanta:

```yaml
CONFIG_SESSION_PHONE_VERSION: 2.2413.51
```

Recrie o container:

```powershell
docker compose up -d --force-recreate evolution-api
```

### Google Sheets “Domain Error”

```powershell
docker compose restart
```

---

## 🚧 Limitações Conhecidas

* Não possui controle de duplicidade por ID
* Não bloqueia spam automaticamente
* Não valida pagamento (manual via Status)

---

## 🔮 Próximos Passos

* Marcar como `Cobrado`
* Integração com Pix / Stripe
* Logs de envio
* Painel de métricas
* Multi-instâncias WhatsApp

---

## ⚠️ Aviso Legal

Este projeto é educacional.
O uso de automação no WhatsApp deve respeitar:

* termos do WhatsApp,
* LGPD,
* consentimento do cliente.

---

## 🧩 Conclusão

Este repositório entrega:

* Infraestrutura reprodutível
* Integração segura com Google
* Lógica confiável de cobrança
* Compatibilidade com Android e iOS

Um sistema simples e extensível.
