# Checkpoint 4 — Verte Bien | Integraciones Avanzadas

Proyecto de automatización en **n8n** para recibir consultas mediante un Webhook, gestionar contactos y memoria de sesiones en Airtable, clasificar consultas con un agente de IA, derivar el procesamiento a Workers especializados, registrar trazabilidad en Slack y generar borradores en Gmail para revisión humana.

## Archivo principal

- `checkpoint4_javier_salafica.json`: exportación del workflow **Checkpoint 4 - Verte Bien - Integraciones Avanzadas**.

## Requisitos y dependencias

- Una instancia de n8n compatible con los nodos del archivo exportado.
- Credenciales configuradas en n8n para **Airtable** (Personal Access Token), **OpenAI**, **Slack** y **Gmail** (OAuth2).
- Base de Airtable **Verte Bien - Memoria Agéntica**, con las tablas `Contactos_CRM` y `Memoria_Sesiones` y los campos que utilizan los nodos del workflow.
- Los subworkflows **Verte Bien - Worker Precios** y **Verte Bien - Worker Información**, configurados y accesibles desde los nodos `Ejecutar Worker Precios` y `Ejecutar Worker Información`. **No están incluidos en este JSON**, por lo que deben existir en la instancia de n8n para ejecutar esas ramas.
- Canal de Slack autorizado para el registro de eventos.

> **Nota sobre autenticación:** Gmail utiliza OAuth2; Airtable se conecta mediante un Personal Access Token en esta implementación. El archivo exportado contiene referencias a credenciales, pero no las credenciales ni sus secretos.

## Importación y configuración

1. En n8n, importar `checkpoint4_javier_salafica.json` como workflow.
2. Revisar y volver a seleccionar las credenciales de Airtable, OpenAI, Gmail y Slack en sus respectivos nodos.
3. Verificar la base y tablas de Airtable. En `Contactos_CRM` se utilizan los campos `Email`, `Nombre` y `Session_ID`; los campos `Fecha_Creacion` y `Ultima_Actualizacion` son automáticos. En `Memoria_Sesiones` se utilizan los campos configurados en los nodos de memoria, entre ellos `Session_ID`, `Cantidad_Mensajes`, `Resumen Consolidado`, `Estado del Caso` y `Datos Clave`.
4. Revisar que los nodos `Ejecutar Worker Precios` y `Ejecutar Worker Información` apunten a los subworkflows correctos en la instancia de destino.
5. Comprobar que el nodo `Log Trazabilidad Multi-Agente` utilice un canal de Slack accesible y que `Gmail | Crear borrador para revisión humana` tenga una cuenta autorizada.
6. Para pruebas, abrir `Entrada Webhook - Verte Bien`, seleccionar **Listen for test event** y utilizar la **Test URL**. Para usar la URL de producción, activar el workflow y utilizar la **Production URL** que muestre n8n. El exportado se encuentra con `active: false`.

## Entrada del Webhook

- Método: **POST**
- Ruta: `vertebien-consultas`
- Cuerpo: JSON

Ejemplo de solicitud:

```json
{
  "session_id": "test-crm-001",
  "nombre": "Contacto Prueba",
  "email": "contacto.prueba@example.com",
  "mensaje": "Quiero información sobre Botox",
  "subject": "Consulta de tratamiento"
}
```

Ejemplo de respuesta HTTP configurada en `Responder Webhook - Verte Bien`:

```json
{
  "status": "ok",
  "message": "Consulta recibida correctamente",
  "session_id": "test-crm-001"
}
```

**Importante:** esta respuesta confirma la recepción de la consulta, no la finalización de todo el procesamiento posterior.

## Funcionamiento principal

1. `Entrada Webhook - Verte Bien` recibe la consulta y `Normalizar Entrada Webhook - Verte Bien` extrae los campos.
2. `Seguridad | Filtrar respuestas automáticas` identifica mensajes automáticos para evitar su procesamiento.
3. `CRM | Buscar contacto por email` realiza una búsqueda **Search** en `Contactos_CRM` utilizando `Email` y un límite de un resultado.
4. `CRM | ¿Contacto existente?` decide: **True** → `CRM | Actualizar contacto`; **False** → `CRM | Crear contacto`.
5. Ambas ramas continúan hacia `Memoria | Buscar sesión`, donde se recupera o crea el contexto conversacional según `Session_ID`.
6. `Manager Verte Bien` clasifica la consulta en `PRECIO`, `INFORMACION` o `ESCALAR` y el router selecciona la rama correspondiente.
7. Las ramas `PRECIO` e `INFORMACION` ejecutan los Workers externos. `ESCALAR` prepara un borrador de Gmail para revisión humana, **sin enviarlo automáticamente**.
8. `Seguridad | Limpiar payload para Slack` prepara los datos de trazabilidad y `Log Trazabilidad Multi-Agente` los registra en Slack.

## Pruebas de validación

### A. Contacto nuevo

Enviar una solicitud con un email que no esté en `Contactos_CRM` y verificar:

- `CRM | Buscar contacto por email` no encuentra resultados.
- `CRM | ¿Contacto existente?` sale por **False**.
- `CRM | Crear contacto` crea el registro con `Email`, `Nombre` y `Session_ID`.

### B. Contacto existente y prevención de duplicados

Repetir la solicitud con **el mismo email** y otro `session_id`. Verificar:

- La búsqueda devuelve el ID del registro ya creado.
- El IF sale por **True**.
- `CRM | Actualizar contacto` actualiza `Nombre` y `Session_ID` sobre ese ID.
- En Airtable sigue existiendo **un solo contacto** para el email probado.

Esta prueba valida el mecanismo de búsqueda previa en condiciones secuenciales. La tabla Airtable no incorpora, por sí sola, una restricción de unicidad que garantice ausencia de duplicados ante solicitudes simultáneas.

### C. Clasificación y trazabilidad

Probar consultas de precio, información y escalamiento; revisar la rama elegida y el registro en Slack. En un caso `ESCALAR`, verificar que Gmail cree un **borrador**, no un correo enviado.

### D. Filtro de respuestas automáticas

Probar un `subject` que indique una respuesta automática y comprobar que no avance hacia el CRM.

## Alcance de la entrega

El JSON incluido permite inspeccionar la arquitectura y configuración del workflow principal. Para reproducir la ejecución completa hacen falta las credenciales, la base Airtable, el canal Slack y los dos Workers externos mencionados. No se incluyen tokens, contraseñas ni secretos.
