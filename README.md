# Checkpoint 4 – Sincronización del cerebro agéntico con el ecosistema de negocio

Pre-entrega 4 (Módulo 4) del curso **IA Automation Avanzado – Coderhouse**.
Autora: Camila Barrionuevo.

> El contexto de negocio es ficticio y sirve para el ejercicio. Evoluciona el proyecto de M1–M3 (consultora de mejora de procesos y automatización que atiende leads).

## Qué hace

Cuando llega un correo a la casilla comercial:

1. **Descarta los correos automáticos** (auto-reply, out of office, undeliverable, `no-reply@`) para evitar bucles de auto-respuestas.
2. **Busca el contacto en HubSpot** por email y **solo lo crea si no existe** (evita el Error 409 por duplicados).
3. Un modelo redacta una **respuesta como BORRADOR** en Gmail. Nunca se envía sola: un humano la revisa.
4. **Limpia el payload** y avisa al equipo de operaciones por Slack.

## Arquitectura

```
Gmail Trigger ─► ① IF anti auto-reply ─(true)─► Descartar
                        │(false)
                        ▼
              Normalizar datos del correo
                        ▼
        ② Look up: buscar contacto en HubSpot
                        ▼
              ¿Contacto ya existe?
        (sí) ─► CRM existente ──────────────┐
        (no) ─► Create contacto ─► CRM nuevo ┤
                                             ▼
                          Redactor de borrador (IA, gpt-4o)
                                             ▼
                          ③ Gmail – Create Draft (HITL)
                                             ▼
                          ④ Set – limpiar payload
                                             ▼
                          Slack #operaciones
```

| Nodo clave de la rúbrica | Qué resuelve |
|---|---|
| ① IF anti auto-reply | Corta el bucle infinito de auto-respuestas (asunto, remitente y header `Auto-Submitted`). |
| ② Look up antes del Create | Evita duplicados en el CRM (Error 409). |
| ③ Gmail Create Draft | Human-in-the-loop: no existe ningún nodo de envío. |
| ④ Set de limpieza | Deja solo 5 campos cortos y sin binarios para Slack (evita Error 400 / payload pesado). |

## Mínimo privilegio

| Conector | Lectura (pasado) | Escritura (futuro) |
|---|---|---|
| Gmail OAuth2 | Bandeja INBOX no leída, sin descargar adjuntos | Solo `Create Draft` |
| HubSpot OAuth2 | `Search` de contactos; propiedades `email`, `firstname`, `lastname` | Solo `Upsert` de contacto (`email`, `firstname`, `lastname`) |
| Slack OAuth2 | Ninguna | Solo `chat:write` en `#operaciones` |

Scopes HubSpot: `crm.objects.contacts.read` y `crm.objects.contacts.write`. Sin acceso a deals ni empresas.



## Pruebas (test de regresión manual)

| Caso | Resultado esperado |
|---|---|
| Correo con asunto "Out of office" | Termina en `Descartar - correo automatico`; no toca CRM ni Gmail |
| Correo desde `no-reply@…` | Igual que el anterior |
| Consulta de un contacto nuevo | Se crea en HubSpot, queda un borrador en Gmail y llega el aviso a Slack |
| Segundo correo del mismo remitente | No crea duplicado (`EXISTENTE`), nuevo borrador y aviso |
| Consulta de precios o con intento de manipulación | Borrador neutral de derivación a un asesor |

## Evidencia de ejecución

Carpeta `evidencia/` (pruebas con correos ficticios):

1. Ejecución con contacto **nuevo**: el contacto se crea en HubSpot.
2. Ejecución con contacto **existente**: el Look up lo encuentra y no se crea duplicado.
3. Borrador generado en Gmail (no se envía).
4. Avisos en Slack para ambos casos, con el mismo Trace ID de la ejecución.

## Seguridad

Este repositorio no contiene claves ni credenciales. El JSON exportado solo referencia credenciales por nombre.
