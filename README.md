# Orquestación de flujos con n8n — IA en Slack e IVR de voz

Implementación de un **agente virtual conversacional** sobre **AWS + n8n**, integrando IA (vía OpenRouter) con **Slack** y con un **IVR de voz** (ElevenLabs).

Asignatura **CUY5132** · Labs **3.1.2 / 3.2.2 / 3.3.2** · Duoc UC

---

## 📋 Tecnologías y cuentas necesarias

- **AWS EC2** (Ubuntu Server) con Docker
- **n8n** (autohospedado en contenedor Docker)
- **DuckDNS** (DNS dinámico) → dominio: `abrahamn8n.duckdns.org`
- **Nginx** (proxy inverso) + **Certbot/Let's Encrypt** (HTTPS)
- **Slack** (app + bot)
- **OpenRouter** (modelo de IA gratuito)
- **ElevenLabs** (agente de voz TTS/STT)

---

## ⭐ Nota clave: el modelo de IA (leer antes de empezar)

El modelo gratuito fue lo más problemático. Resumen de lo que **NO** funcionó y lo que **SÍ**:

- ❌ El nodo **"OpenRouter Chat Model"** directo daba error `Payment required` y luego `Rate limit reached`.
- ✅ Lo que funcionó: nodo **"OpenAI Chat Model"** usando la **credencial de OpenRouter** (API key `sk-or-v1-...`).
  - **Modelo:** `google/gemini-2.5-flash`
  - **Add Option → Maximum Number of Tokens → `1000`** *(imprescindible: por defecto pide 65.535 tokens y la cuenta gratis no alcanza).*

> **Receta del modelo (vale para los dos labs):**
> OpenAI Chat Model → Credencial = cuenta OpenRouter → Modelo `google/gemini-2.5-flash` → Max Tokens `1000`.

---

## ⚠️ Importante: IP pública dinámica

La IP pública de la EC2 **cambia** cada vez que se apaga/prende la instancia. Tras cada reinicio:

1. En AWS, copiar la **nueva IP pública**.
2. Entrar a <https://www.duckdns.org>, poner la IP nueva en **current ip** del dominio y clic en **update ip**.
3. Verificar `https://abrahamn8n.duckdns.org` (no hace falta renovar SSL: el certificado es del dominio).

---

# Parte A — Infraestructura (AWS + n8n)

> Comandos de Docker/Nginx/Certbot universales. Los `install` dependen del SO: `apt` (Ubuntu) o `yum/dnf` (Amazon Linux).

### A1. Crear la instancia EC2
- AMI: **Ubuntu Server** (o Amazon Linux 2024).
- Tipo: `t3.micro` / `t3.small`.
- Par de llaves SSH (formato `.ppk` para PuTTY en Windows).

### A2. Security Group (puertos)
| Tipo | Protocolo | Puerto | Origen | Descripción |
|---|---|---|---|---|
| SSH | TCP | 22 | Mi IP | Administración |
| HTTP | TCP | 80 | 0.0.0.0/0 | Acceso web |
| HTTPS | TCP | 443 | 0.0.0.0/0 | Acceso web seguro |
| Custom TCP | TCP | 5678 | 0.0.0.0/0 | n8n *(opcional: con nginx no es estrictamente necesario abrirlo)* |

### A3. Conexión SSH
Con PuTTY (Windows) usando la llave `.ppk` y el usuario según el SO:
- Ubuntu → usuario `ubuntu`
- Amazon Linux → usuario `ec2-user`

### A4. Instalar Docker y Docker Compose
```bash
# Ubuntu
sudo apt update -y && sudo apt install -y docker.io
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker $USER
# (Amazon Linux: sudo yum install -y docker)
```

### A5. Desplegar n8n (contenedor) — usa TU dominio
```bash
sudo docker run -d --restart unless-stopped -it \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="abrahamn8n.duckdns.org" \
  -e WEBHOOK_TUNNEL_URL="https://abrahamn8n.duckdns.org/" \
  -e WEBHOOK_URL="https://abrahamn8n.duckdns.org/" \
  -v ~/.n8n:/root/.n8n \
  n8nio/n8n
```
Verificar con `docker ps`.

### A6. Crear el dominio en DuckDNS
- Crear el subdominio `abrahamn8n` en <https://www.duckdns.org> y asociarlo a la IP pública de la EC2.

### A7. Nginx como proxy inverso
```bash
sudo apt install -y nginx      # (Amazon Linux: sudo dnf install -y nginx)
sudo systemctl enable nginx && sudo systemctl start nginx
sudo nano /etc/nginx/conf.d/n8n.conf
```
Contenido del archivo (`n8n.conf`):
```nginx
server {
    listen 80;
    server_name abrahamn8n.duckdns.org;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Connection 'Upgrade';
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Probar y recargar:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

### A8. HTTPS con Certbot (soluciona la cookie segura de n8n)
```bash
sudo apt install -y certbot python3-certbot-nginx   # (Amazon Linux: sudo dnf install -y ...)
sudo certbot --nginx -d abrahamn8n.duckdns.org
sudo systemctl restart nginx
```
Ya se accede por `https://abrahamn8n.duckdns.org`.

### A9. Configuración inicial de n8n
- Crear el usuario administrador y el espacio de trabajo (ej: *"Laboratorio IA"*).
- (Opcional) Activar la licencia gratuita de la Community Edition desde **Settings → Usage and plan → Enter activation key** (la key llega por correo).

---

# Parte B — Lab 3.2.2: IA en Slack

**Flujo:** `Slack Trigger → AI Agent → Send a message (Slack)`

### B1. Crear la app de Slack
1. <https://api.slack.com/apps> → **Create an App → From scratch**.
2. Nombre `n8n-Abraham`, elegir el workspace.
3. En **Basic Information** anotar **Client ID** y **Client Secret**.

### B2. Credencial OAuth2 del nodo "Send a message"
1. En n8n, doble clic al nodo Slack **Send a message** → **Create new credential** (Slack OAuth2).
2. Pegar **Client ID** y **Client Secret**, copiar el **OAuth Redirect URL**.
3. En Slack → **OAuth & Permissions → Redirect URLs** → pegar la URL → **Save URLs**.
4. En **Scopes (Bot Token Scopes)** agregar: `app_mentions:read`, `channels:history`, `channels:read`, `chat:write`, `groups:history`, `groups:read`.
5. Volver a n8n y **Connect** (autorizar en el workspace).

### B3. Slack Trigger (Bot Token)
1. Agregar nodo **Slack Trigger** → nueva credencial (Slack API).
2. En Slack → **OAuth & Permissions → Reinstall to workspace** → permitir → copiar el **Bot User OAuth Token** (`xoxb-...`) y pegarlo en la credencial del trigger.
3. **Trigger On:** `app_mention` (o Any Event) y elegir un **canal** en *Channel to Watch*.
4. Copiar la **webhook URL** del trigger.

### B4. Event Subscriptions (verificación)
1. En Slack → **Event Subscriptions → Enable Events** → pegar la URL del webhook.
2. Si da error de verificación, en n8n elegir un canal en el trigger y dejar el nodo escuchando hasta que Slack muestre **Verified**.
3. Suscribir el evento `app_mention`.

### B5. Configurar el AI Agent
- **Source for Prompt (User Message):** `Define below`
- **Prompt (User Message):** `{{ $json.text }}`
  > ⚠️ En Slack el mensaje llega en `text`, **no** en `chatInput`. Si se deja `chatInput` aparece "undefined".

### B6. Conectar el AI Agent al nodo Slack (¡cuidado con los puertos!)
El AI Agent tiene dos tipos de conexión:
- **Salida principal** = punto a la **derecha** (línea **sólida**) → por ahí sigue el flujo.
- **Puertos de abajo** (Chat Model / Memory / Tool) = rombos con línea **punteada** → solo sub-nodos.

➡️ Conectar la **salida principal** del AI Agent al nodo **Send a message** (línea sólida). **No** usar el puerto *Tool*.

### B7. Modelo de IA (la receta)
- Conectar al puerto **Chat Model** un **OpenAI Chat Model**.
- Credencial = **OpenRouter** (`sk-or-v1-...`).
- Modelo `google/gemini-2.5-flash`.
- **Add Option → Maximum Number of Tokens → 1000**.

### B8. Nodo "Send a message" (Slack)
- **Resource:** Message · **Operation:** Send
- **Send Message To:** Channel
- **Channel:** `By ID` → ID del canal *(o `{{ $('Slack Trigger').item.json.channel }}` para responder en cualquier canal)*.
- **Message Type:** Simple Text Message
- **Message Text:** `{{ $json.output }}`

### B9. Probar
1. **Execute workflow** (o activar el flujo).
2. En Slack: `@n8n-Abraham hola, ¿cuánto mide Chile?`
3. El bot responde en el canal ✅. *(Si dice que no está en el canal: `/invite @n8n-Abraham`.)*

---

# Parte C — Lab 3.3.2: IVR que entiende comandos de voz

**Flujo:** `ElevenLabs (voz) → Webhook entrada → AI Agent (modelo) → Respond to Webhook (salida)`

### C1. Nuevo workflow + Webhook de entrada
1. n8n → **Create workflow** (nuevo). Nombre `IVR voz`.
2. Add first step → **Webhook**.
3. **HTTP Method:** `POST` · **Path:** `voz` · **Authentication:** None · **Respond:** Immediately *(se cambia al final)*.
4. URL de prueba: `https://abrahamn8n.duckdns.org/webhook-test/voz`

### C2. Agente de voz en ElevenLabs
1. <https://elevenlabs.io> → registrarse con Google.
2. Ir a la plataforma de agentes: <https://elevenlabs.io/app/agents/agents> *(si aparece la "nueva vista", clic en **Desactivar nueva vista**).*
3. **+ Nuevo agente → Agente en blanco** → nombre `agentedevoz` → Crear *(sin "Solo chat")*.
4. Pestaña **Agente**:
   - **Mensaje del sistema** (forzar el uso de la herramienta):
     > Eres un asistente experto en redes y ciberseguridad, claro y en español. Para responder CUALQUIER consulta del usuario, SIEMPRE usa la herramienta `n8nvoz` enviando el texto en `User_Request`; no respondas por tu cuenta.
   - **Primer mensaje:** *¡Hola! ¿En qué puedo ayudarte hoy?*
   - **Idioma:** Español · **Voz:** cualquiera · **LLM:** el que viene por defecto.

### C3. Herramienta webhook (conecta con n8n)
Pestaña **Herramientas → Añadir herramienta → Añadir herramienta webhook**:
- **Nombre:** `n8nvoz`
- **Descripción:** `Para enlazar a n8n y enviar las consultas`
- **Método:** `POST`
- **URL:** `https://abrahamn8n.duckdns.org/webhook-test/voz`
- **Parámetros del cuerpo (body):**
  - Descripción: `Por favor resume el request principal del usuario`
  - Propiedad → Tipo `String` · Identificador `User_Request` · **Requerido** ✅ · Tipo de valor `LLM Prompt`
- **Añadir herramienta** y guardar.

### C4. Probar que la request llega al webhook
1. En n8n, nodo **Webhook** → **Listen for test event**.
2. En ElevenLabs → **Vista previa** → hablar/escribir al agente (ej: "¿cuánto mide Chile?").
3. El nodo **Webhook** se pone **verde (1 item)** ✅. El dato llega en `body.User_Request`.

### C5. AI Agent + modelo
1. `+` a la derecha del Webhook → **AI Agent**.
2. **Source for Prompt:** `Define below` · **Prompt:** `{{ $json.body.User_Request }}` *(debe mostrar el texto, no "undefined")*.
3. Puerto **Chat Model** → **OpenAI Chat Model** (receta): credencial OpenRouter · `google/gemini-2.5-flash` · **Max Tokens 1000**.
4. **Execute step** → debe responder.

### C6. Respond to Webhook + prueba final de voz
1. `+` a la derecha del AI Agent → **Respond to Webhook** → **Respond With:** `First Incoming Item`.
2. Volver al **Webhook** de entrada → **Respond** → cambiar a **"Using 'Respond to Webhook' Node"**.
3. Guardar y **Activar** el workflow (o Execute workflow para escuchar).
   - Si se activa (Production URL), en ElevenLabs la URL de la herramienta debe ser **sin** `-test`: `https://abrahamn8n.duckdns.org/webhook/voz`
4. En ElevenLabs → **Vista previa** → **llamar al agente** (modo voz) → preguntar hablando.
5. El agente **lee la respuesta en voz alta** y los 4 nodos quedan en verde 🎙️✅ *(verificar en la pestaña **Executions** de n8n).*

---

## 🛑 Al terminar
- **Apagar (Stop) la instancia EC2** para no consumir créditos.
- Al prenderla de nuevo, la IP cambia → **actualizar DuckDNS**.

## Próximo lab
- **3.4.2 — Contenerización con Docker.**

---

## 📑 Datos del proyecto (referencia rápida)
| Cosa | Valor |
|---|---|
| Dominio DuckDNS | `abrahamn8n.duckdns.org` |
| Webhook voz (test) | `https://abrahamn8n.duckdns.org/webhook-test/voz` |
| Webhook voz (prod) | `https://abrahamn8n.duckdns.org/webhook/voz` |
| Modelo IA que funcionó | `google/gemini-2.5-flash` (nodo OpenAI Chat Model + credencial OpenRouter) |
| Max Tokens | `1000` |
| Bot de Slack | `@n8n-Abraham` |


<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/b0e7f71f-5aa6-4168-b48f-853ae4301fd1" />

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/419a996e-6ba3-4df9-8333-9fd45c64cc51" />
