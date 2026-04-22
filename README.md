# 📡 Monitoramento de Dispositivos Unifi — n8n Workflows

Sistema de monitoramento automatizado para dispositivos **Unifi** (UniFi Network Application), com detecção de desconexões, controle de estado via **Baserow** e notificação por **e-mail**.

O projeto é composto por **dois workflows** que trabalham em conjunto.

---

## 🔁 Arquitetura Geral

```
[1/2] Monitoramento
  ├── Trigger (a cada minuto)
  │     └── API Unifi → Separa dispositivos → Filtra desconectados
  │           ├── Loop 1: Verifica estado no Baserow → Envia alerta (chama [2/2])
  │           └── Loop 2: Atualiza registro no Baserow
  │
  └── Trigger de Recriação (diário, 01h)
        └── Limpa tabela Baserow → Recria todos os registros da API

[2/2] Tratar Alerta
  └── Acionado pelo workflow [1/2]
        └── Monta e-mail HTML → Envia alerta de dispositivo offline
```

---

## 📂 Arquivos

```
.
├── 1-2-monitoramento-dispositivos.json   # Workflow principal de monitoramento
├── 2-2-tratar-alerta-dispositivo.json    # Sub-workflow de envio de alerta
└── README.md
```

---

## ⚙️ Como funciona

### Workflow 1/2 — Monitoramento

| Trigger | Frequência | Descrição |
|---|---|---|
| Trigger de atualização | A cada minuto | Consulta a API Unifi e processa os dispositivos |
| Trigger de recriação | Diário (01h) | Limpa e recria toda a tabela de dispositivos no Baserow |

**Fluxo de atualização:**
1. Chama a API Unifi e obtém todos os dispositivos
2. Divide a resposta em itens individuais por device
3. Filtra os que estão com estado `disconnected`
4. Para cada dispositivo desconectado:
   - Verifica no Baserow se já existe registro (evita alertas duplicados)
   - Atualiza o status no Baserow para `disconnected`
   - Chama o workflow [2/2] para enviar o alerta

**Fluxo de recriação:**
1. Busca todos os registros da tabela no Baserow
2. Deleta cada um
3. Consulta a API Unifi novamente
4. Recria os registros com o status atual de cada dispositivo

### Workflow 2/2 — Tratar Alerta

1. Recebido o acionamento do workflow [1/2] com `device_id`, `device_name` e `device_state`
2. Monta as variáveis do e-mail (título e HTML responsivo com suporte a dark mode)
3. Envia o e-mail de alerta para a equipe responsável

---

## 🔐 Configuração — Placeholders

Antes de importar, substitua os placeholders pelos valores reais do seu ambiente:

### API Unifi

| Placeholder | Descrição |
|---|---|
| `{{UNIFI_API_KEY}}` | API Key da UniFi Network Application |

### Baserow

| Placeholder | Descrição |
|---|---|
| `{{BASEROW_CREDENTIAL_ID}}` | ID da credencial Baserow no n8n |
| `{{BASEROW_CREDENTIAL_NAME}}` | Nome da credencial Baserow no n8n |
| `{{BASEROW_DATABASE_ID}}` | ID do banco de dados no Baserow |
| `{{BASEROW_TABLE_ID}}` | ID da tabela de dispositivos no Baserow |
| `{{FIELD_ID_DEVICE_ID}}` | ID do campo que armazena o identificador do device |
| `{{FIELD_ID_STATUS}}` | ID do campo de status (`connected` / `disconnected`) |
| `{{FIELD_ID_HOSTNAME}}` | ID do campo de hostname do device |
| `{{FIELD_ID_LAST_SEEN}}` | ID do campo de data/hora da última atualização |

### E-mail

| Placeholder | Descrição |
|---|---|
| `{{SENDER_EMAIL}}` | E-mail remetente (ex: `noreply@seudominio.com`) |
| `{{RECIPIENT_EMAIL}}` | E-mail que receberá os alertas de dispositivo offline |
| `{{SMTP_CREDENTIAL_ID}}` | ID da credencial SMTP no n8n |
| `{{SMTP_CREDENTIAL_NAME}}` | Nome da credencial SMTP no n8n |

### Linking entre workflows

| Placeholder | Descrição |
|---|---|
| `{{SUBWORKFLOW_ID}}` | ID do workflow [2/2] no n8n (obtido após importar o segundo workflow) |

### n8n

| Placeholder | Descrição |
|---|---|
| `{{N8N_INSTANCE_ID}}` | ID da instância do n8n (opcional) |

---

## 🗄️ Estrutura da Tabela no Baserow

Crie uma tabela com as seguintes colunas:

| Campo | Tipo sugerido | Descrição |
|---|---|---|
| `device_id` | Texto | Identificador único do dispositivo na API Unifi |
| `hostname` | Texto | Nome do host do dispositivo |
| `status` | Texto | Estado atual: `connected` ou `disconnected` |
| `last_seen` | Data/Hora | Última atualização de status |

---

## 🔑 Como obter a API Key da Unifi

1. Acesse o painel do **UniFi Network Application** ou **UniFi Site Manager**
2. Vá em **Settings** → **Control Plane** → **API Keys**
3. Gere uma nova chave e copie o valor

---

## 📦 Como importar no n8n

1. Importe primeiro o arquivo `2-2-tratar-alerta-dispositivo.json`
2. Anote o **ID** gerado para esse workflow
3. Importe o arquivo `1-2-monitoramento-dispositivos.json`
4. No nó **"Call [2/2] Tratar Alerta de Dispositivo"**, atualize o `workflowId` com o ID anotado
5. Configure todas as credenciais (Baserow e SMTP)
6. Substitua os demais placeholders
7. Ative ambos os workflows

---

## 🛠️ Requisitos

- n8n v1.0+
- Conta Baserow (self-hosted ou cloud)
- API Key da Unifi Network Application
- Servidor SMTP ou serviço de e-mail transacional (ex: Resend, SendGrid, Brevo)
