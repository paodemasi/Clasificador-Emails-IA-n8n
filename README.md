# 📧 Clasificador de Emails con IA Local — n8n + Ollama

Sistema de automatización que clasifica correos entrantes en tiempo real usando un modelo de lenguaje local (LLM), sin enviar datos sensibles a servicios externos. Diseñado bajo un paradigma de **privacidad, resiliencia y trazabilidad**.

---

## 🧭 Visión General

El workflow monitorea la bandeja de entrada de Gmail, detecta nuevos correos y los procesa automáticamente:

1. Detecta si el correo tiene adjuntos → rama separada con alerta inmediata
2. Normaliza y prepara el contenido del email
3. Lo envía a un LLM local (Ollama + Llama 3.2) para clasificación
4. Parsea y valida la respuesta estructurada
5. Ejecuta acciones multicanal: etiqueta en Gmail, registra en Google Sheets y notifica por Telegram

En caso de error en cualquier paso, el sistema redirige el email a revisión manual sin perder el ticket.

---

## 🏗️ Arquitectura del Flujo

```
Gmail Trigger
    │
    ▼
¿Tiene adjuntos?
    ├── SÍ → Label: Con-Adjuntos → Google Sheets → Telegram (alerta adjuntos)
    │
    └── NO → Preparar Email
                    │
                    ▼
            Ollama - Clasificar
                    ├── ERROR de conexión → Label: Revisión Manual → Sheets - Error → Telegram (alerta error)
                    │
                    └── OK → Parsear Respuesta Ollama
                                    │
                              ¿Error de Parseo?
                                    ├── SÍ → Label: Revisión Manual → Sheets - Error → Telegram (alerta error)
                                    │
                                    └── NO → Switch por Categoría
                                                ├── Consulta → Label Gmail
                                                ├── Soporte  → Label Gmail
                                                └── Urgente  → Label Gmail
                                                        │
                                                    Marcar como Leído
                                                        │
                                                    Google Sheets
                                                        │
                                                    Telegram (notificación)
```

---

## ⚙️ Componentes Clave

| Componente | Tecnología | Rol |
|---|---|---|
| Ingesta | Gmail Trigger (polling) | Captura correos nuevos en tiempo real |
| Cerebro cognitivo | Ollama + Llama 3.2 (local) | Clasificación, prioridad y resumen |
| Normalización | JavaScript (nodo Code) | Limpieza y estructuración del email |
| Resiliencia | Nodo IF + rama de error | Fallback a revisión manual ante fallas |
| Registro | Google Sheets | Trazabilidad histórica de todos los emails |
| Notificaciones | Telegram Bot | Alertas en tiempo real por categoría |
| Organización | Gmail Labels | Etiquetado automático de la bandeja |

---

## 🧠 Justificación de Decisiones Técnicas

**¿Por qué HTTP Request en lugar de nodo nativo de Ollama?**
Permite control granular sobre los parámetros del modelo (temperatura, formato de respuesta, timeout) y facilita el parseo manual de la respuesta, resultando en un pipeline más robusto ante cambios de versión del modelo.

**¿Por qué JSON estricto en el prompt?**
Se diseñó un prompt que instruye al modelo a responder *únicamente* con JSON válido, eliminando texto adicional o markdown que rompe el parseo automático. Esto reduce los errores de ejecución en los nodos subsiguientes y mejora la confiabilidad del flujo.

**¿Por qué LLM local (Ollama) en lugar de API externa?**
Los emails de clientes contienen información sensible del negocio. Usar un modelo local garantiza que los datos no salgan de la infraestructura, cumpliendo con criterios básicos de privacidad sin costo por token.

**¿Por qué rama de contingencia para adjuntos?**
Los archivos binarios requieren procesamiento diferente al texto. Separarlos desde el inicio evita errores de contexto en el LLM y permite una gestión manual más eficiente de documentos que requieren revisión humana.

---

## 🚀 Instrucciones de Despliegue

### Prerrequisitos

- [n8n](https://n8n.io/) instalado (self-hosted o Docker)
- [Ollama](https://ollama.ai/) instalado localmente con el modelo `llama3.2` descargado
- Cuenta de Gmail con OAuth2 configurado
- Bot de Telegram creado via [@BotFather](https://t.me/BotFather)
- Google Sheets API habilitada en Google Cloud Console

### Paso 1 — Instalar y levantar Ollama

```bash
# Instalar Ollama (Linux/Mac)
curl -fsSL https://ollama.ai/install.sh | sh

# Descargar el modelo
ollama pull llama3.2

# Verificar que Ollama está corriendo
curl http://localhost:11434/api/tags
```

> Si usás n8n en Docker, Ollama debe ser accesible en `host.docker.internal:11434` desde el contenedor.

### Paso 2 — Configurar credenciales en n8n

Antes de importar el workflow, creá las siguientes credenciales en n8n:

| Credencial | Tipo | Dónde obtenerla |
|---|---|---|
| `Gmail account` | Gmail OAuth2 | [Google Cloud Console](https://console.cloud.google.com/) → APIs → Gmail API |
| `Google Sheets account` | Google Sheets OAuth2 | Misma consola → Sheets API |
| `Telegram account` | Telegram Bot API | [@BotFather](https://t.me/BotFather) → `/newbot` |

### Paso 3 — Importar el workflow

1. En n8n, ir a **Workflows → Import from file**
2. Seleccionar el archivo `clasificador-mails.json`
3. Guardar

### Paso 4 — Configurar variables del workflow

Una vez importado, actualizar los siguientes valores en los nodos indicados:

#### Nodo `Label: Consulta` / `Label: Soporte` / `Label: Urgente`
Reemplazar los IDs de labels de Gmail:
```
Label_CONSULTA_ID  →  ID real del label "AutoClasificado/Consulta"
Label_SOPORTE_ID   →  ID real del label "AutoClasificado/Soporte"
Label_URGENTE_ID   →  ID real del label "AutoClasificado/Urgente"
```

Para obtener los IDs de los labels, usá la [Gmail API Explorer](https://developers.google.com/gmail/api/reference/rest/v1/users.labels/list) o ejecutá:
```bash
curl -H "Authorization: Bearer TU_TOKEN" \
  "https://gmail.googleapis.com/gmail/v1/users/me/labels"
```

#### Nodo `Google Sheets - Agregar Fila - Mail clasificado` y `Sheets - Error`
Reemplazar el ID de la planilla:
```
1X_JVN9Sthl1QmIroq8vGzN3jLLFAvRs7M0wT8Gjfs_U  →  ID de tu propia Google Sheet
```

El ID está en la URL de tu planilla:
```
https://docs.google.com/spreadsheets/d/[ESTE-ES-EL-ID]/edit
```

#### Nodo `Telegram - Notificar` / `Telegram - Alerta Adjuntos` / `Telegram - Alerta Error`
Reemplazar el Chat ID:
```
TU_CHAT_ID  →  Tu Chat ID personal
```

Para obtener tu Chat ID, enviá un mensaje a tu bot y consultá:
```
https://api.telegram.org/botTU_TOKEN/getUpdates
```

#### Nodo `Ollama - Clasificar`
Si Ollama corre en un puerto diferente o en otra dirección:
```
http://host.docker.internal:11434/api/generate  →  tu dirección de Ollama
```

### Paso 5 — Estructura de Google Sheets

Crear una hoja llamada `Resumen` con las siguientes columnas en orden:

| FECHA | REMITENTE | ASUNTO | CATEGORÍA | PRIORIDAD | RESUMEN | DOCUMENTOS ADJUNTOS | RESPONDIDO | FECHA RESPONDIDO | RESUELTO |

### Paso 6 — Activar el workflow

1. En n8n, hacer clic en el toggle **Publish** del workflow
2. El trigger de Gmail comenzará a hacer polling cada minuto
3. Verificar en Telegram que llegan las notificaciones de prueba

---

## 📊 Métricas de Evaluación

| Métrica | Valor objetivo | Umbral de alerta |
|---|---|---|
| Tasa de clasificación correcta | ≥ 85% | < 70% → revisión del prompt |
| Tiempo de procesamiento por email | < 30 segundos | > 60 seg → revisar timeout de Ollama |
| Tasa de errores de parseo | < 5% | > 15% → revisar formato del prompt |
| Emails derivados a revisión manual | < 10% | > 25% → revisión del flujo completo |
| Disponibilidad del flujo | ≥ 99% en horario laboral | Caída → alerta automática en Telegram |

Para medir la tasa de clasificación correcta, revisar periódicamente la columna CATEGORÍA en Google Sheets contra los emails reales y registrar discrepancias en una hoja auxiliar `Errores_Clasificacion`.

---

## 🔧 Plan de Mantenimiento

### Revisiones periódicas

| Frecuencia | Tarea | Responsable |
|---|---|---|
| Semanal | Revisar hoja `Revisión Manual` — analizar causas de desvío | Operador del flujo |
| Mensual | Auditar muestra de 50 emails clasificados vs. categoría real | Operador del flujo |
| Mensual | Revisar métricas de error en Google Sheets | Operador del flujo |
| Trimestral | Evaluar si hay una versión más nueva de Llama disponible en Ollama | Responsable técnico |
| Trimestral | Revisar y ajustar el prompt de clasificación según errores acumulados | Responsable técnico |
| Ante cambios de negocio | Actualizar categorías y reglas del prompt si cambia el tipo de emails recibidos | Responsable técnico |

### Criterios de actualización del modelo LLM

Actualizar el modelo (`llama3.2` → versión más nueva) cuando se cumpla alguna de estas condiciones:

- La tasa de errores de parseo supera el 15% por dos semanas consecutivas
- Ollama publica una versión con mejoras documentadas en seguimiento de instrucciones JSON
- El tiempo de procesamiento supera los 60 segundos de forma consistente y hay modelos más eficientes disponibles

Para actualizar:
```bash
ollama pull llama3.2:latest   # o el modelo nuevo elegido
```
Luego actualizar el nombre del modelo en el nodo `Ollama - Clasificar` y ejecutar 10 emails de prueba para validar antes de activar en producción.

### Procedimiento ante caída del flujo

1. Verificar en n8n que el workflow está activo
2. Verificar que Ollama responde: `curl http://localhost:11434/api/tags`
3. Revisar logs de n8n para identificar el nodo con error
4. Los emails recibidos durante la caída quedarán sin procesar — revisar manualmente la bandeja de entrada del período afectado

---

## 🗂️ Estructura del Repositorio

```
clasificador-emails-n8n/
├── README.md                          # Este archivo
├── clasificador-mails.json            # Workflow exportado de n8n
└── sheets-template/
    └── estructura-resumen.md          # Columnas requeridas en Google Sheets
```

---

## 🛠️ Stack Tecnológico

- **n8n** — orquestación del flujo de automatización
- **Ollama + Llama 3.2** — modelo de lenguaje local para clasificación con IA
- **Gmail API** — ingesta de correos y etiquetado automático
- **Google Sheets API** — registro histórico y trazabilidad
- **Telegram Bot API** — notificaciones en tiempo real
  

---

## 👤 Autora

**Paola Demasi** — [github.com/paodemasi](https://github.com/paodemasi)

Proyecto desarrollado como trabajo final del curso AI Automation de Coderhouse, en el marco de un proceso de formación en automatización de datos e inteligencia artificial aplicada.
