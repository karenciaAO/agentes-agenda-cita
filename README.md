# Proyecto Chatbot Multicanal Web
**Autora:** Karen Araque  
**Fecha:** 2025-08-24  
**URL público (S3):** http://aws-chatkaren.s3-website-us-east-1.amazonaws.com/

---

## 1) Introducción / Contexto
Construí un **chatbot web** con **Amazon Lex V2** que agenda citas (caso de uso: reservas de asesoría) y responde dinámicamente a través de **AWS Lambda**, almacenando las reservas en **Amazon DynamoDB**. El bot también **convierte respuestas a voz** con **Amazon Polly** para ofrecer una experiencia multimodal (texto + audio). El objetivo fue publicar un asistente real y accesible mediante un **sitio estático en S3**.

---

## 2) Arquitectura (alto nivel)
**Front-end:** Sitio estático en **Amazon S3** (URL anterior).  
**Back-end:** **API Gateway → Lambda** que invoca **Lex V2** (RecognizeText) y **Polly** (SynthesizeSpeech).  
**Datos:** **DynamoDB** para persistir citas (tabla: `WorkshopAppointments`/`MechanicAppointments`, vía variable `TABLE_NAME`).  
**Identidad/Seguridad:** **IAM** con permisos mínimos para Lex/Polly/DynamoDB; (para la variante client-side se usa Cognito Identity Pool).

**Flujo:** Usuario (web) → POST `/chat` (API GW) → **Lambda** → **Lex (NLP)** → (respuesta) → **Polly (opcional audio)** → Web.  
**Persistencia:** Lambda escribe la reserva en DynamoDB.
  
---

## 3) Intents, Utterances y Slots (≥ 3)
**(A)** `OfficeHours` (antes `OpeningHours`)  
- *Utterances:* “¿Cuál es tu horario?”, “¿A qué horas atiendes?”, “¿Puedo escribirte hoy?”  
- *Slots:* —  
- *Respuesta típica:* “Atiendo de **09:00–18:00 (GMT-5)**, lun–vie.”

**(B)** `CheckCallSlots` (antes `CheckAvailability`)  
- *Utterances:* “¿Tienes cupo el **[2025-08-26]**?”, “Disponibilidad para **mañana**”  
- *Slots:* `Date` (obligatorio)  
- *Respuesta típica:* “Disponibilidad para {fecha}: 10:00, 10:30, 11:00, 15:00…”

**(C)** `BookConsultation` (antes `MakeBooking`)  
- *Utterances:* “Reservar asesoría”, “Agendar una llamada”, “Quiero una demo de chatbot”  
- *Slots:*  
  - `Service`/`ServiceType` (valores: *Asesoría inicial, Auditoría de datos, Chatbot AWS/Lex, Clase 1:1*)  
  - `Date` (YYYY-MM-DD)  
  - `Time` (HH:MM 24h)  
  - `Name` (Nombre)  
  - `Phone` (WhatsApp/email opcional)  
- *Respuesta:* “¡Listo {name}! Reservé **{service}** el **{date}** a las **{time}**. ID **{id}**.” (se guarda en DynamoDB).

**(D)** `CancelConsultation` (antes `CancelBooking`)  
- *Utterances:* “Cancelar cita **A-XXXX**”, “Cancelar reserva del **{fecha}**”  
- *Slots:* `AppointmentId` o (`Phone` + `Date`)  
- *Acción:* Elimina ambas llaves (por cliente y por tienda) en DynamoDB, si existen.

**(E)** `Help` (antes `FallbackIntent`)  
- *Mensaje:* “Puedo **ver horarios**, **reservar** o **cancelar**. Ejemplos: ‘ver disponibilidad mañana’; ‘reservar asesoría el 2025-08-27 10:30’; ‘cancelar cita A-AB12CD34’.”

---

## 4) Descripción de la Lambda (resumen técnico)
**Repositorio lógico:** Handler **Python** con helpers `get_slot`, `elicit_slot`, `close`.  
**Validación de slots:** Si falta alguno (service/date/time/name), se usa **ElicitSlot** hasta completar.  
**Lógica de negocio:**  
- Genera `appointment_id` (UUID corto).  
- Guarda el ítem en **DynamoDB** (`AppointmentId`, `ServiceType`, `Date`, `Time`, `CustomerName`, `PhoneNumber`, `CreatedAt`).  
- Mensaje de confirmación con ID.
- Para `CheckAvailability`, consulta por fecha y arma lista de horas libres con el horario `{open, close, slotMinutes}` (defaults seguros si falta item `INFO#HOURS`).  
- `CancelConsultation` busca por `AppointmentId` o por (`Phone`, `Date`) y borra las entradas.

**Variables de entorno claves:**  
`TABLE_NAME`, `REGION`, `BOT_ID`, `BOT_ALIAS_ID`, `LOCALE_ID`, `ALLOW_ORIGIN`, `VOICE_ID` (opcional).

**Errores & resiliencia:** Manejo de `AccessDeniedException` (IAM), validación de rango horario y colisiones de agenda (query por `pk/sk` o por `date/time`).

---

## 5) Voz (Polly)
**VoiceId:** `Mia` (es-ES).  
**Motivación:** dicción clara y natural para español; volumen y ritmo apropiados para instrucciones rápidas.  
**Mejoras futuras:** usar **SSML** (pausas `<break>`, énfasis `<emphasis>`) y evaluar otras voces (es-MX) para ajuste regional.

---

## 6) Despliegue y permisos (resumen)
- **S3 Website:** `index.html` + `error.html`; **Bucket Policy** pública de solo lectura; *Block Public Access* desactivado a nivel bucket.  
- **API Gateway (HTTP API):** ruta **POST `/chat`** con **AWS_PROXY** a Lambda y **CORS** hacia `karenaraque.space`.  
- **Lambda Role:** `AWSLambdaBasicExecutionRole` + política **mínima**:  
  - `lex:RecognizeText` al ARN del **bot-alias**.  
  - `polly:SynthesizeSpeech` (opcional).  
  - `dynamodb:GetItem/PutItem/DeleteItem/Query/UpdateItem` a la tabla.  
- **Lex Alias:** versión con **locale** habilitado (`es_419` o `es_ES`) y, si aplica, **code hook** a Lambda.

---

## 7) Pruebas realizadas
- **Escenarios:** horario, disponibilidad por fecha, reserva completa, cancelación por ID y por teléfono+fecha.  
- **Front:** envío de texto → respuesta de Lex; reproducción de audio (si `VOICE_ID` configurada).  
- **Datos:** verificación en **DynamoDB** de ítems creados y eliminados.  
- **Logs:** CloudWatch (invocación y errores).

---

## 8) Lecciones aprendidas
- Diferencia entre **política de bucket** y **configuración de website** en S3 (RoutingRules JSON).  
- **Access Analyzer** puede bloquear guardados en consola si el usuario IAM no tiene permisos de validación; usar **CloudShell/CLI** evita ese chequeo.  
- En Lex, los **idiomas se habilitan por alias**; si no, la prueba muestra “No hay idiomas habilitados…”.  
- Manejo de **IAM mínimo** para evitar `AccessDenied` en DynamoDB/Lex/Polly.  
- En macOS, para pruebas locales no es imprescindible XAMPP; pero si se usa, la **raíz** es `…/htdocs`.

---

## 9) Anexos (referencias rápidas)
- **Endpoint S3:** http://aws-chatkaren.s3-website-us-east-1.amazonaws.com/  
- **Slots principales:** `Service/ServiceType`, `Date`, `Time`, `Name`, `Phone`.  
- **Locales válidos:** `es_419` / `es_ES`.  
- **Timezone:** America/Bogota (GMT-5).

---
**Autora:** Karen Araque – karenaraque.space
