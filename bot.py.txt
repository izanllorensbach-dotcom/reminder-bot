import os
import json
import logging
import base64
import pickle
from datetime import datetime, timedelta
import anthropic
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
from google.auth.transport.requests import Request
from googleapiclient.discovery import build
import re

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

TELEGRAM_TOKEN = os.environ["TELEGRAM_TOKEN"]
ALLOWED_USER_ID = int(os.environ["TELEGRAM_USER_ID"])
ANTHROPIC_API_KEY = os.environ["ANTHROPIC_API_KEY"]
CALENDAR_ID_PERSONAL = os.environ.get("CALENDAR_ID_PERSONAL", "primary")
CALENDAR_ID_TALLER = os.environ.get("CALENDAR_ID_TALLER", "primary")
GOOGLE_TOKEN_B64 = os.environ.get("GOOGLE_TOKEN_B64", "")

SCOPES = ['https://www.googleapis.com/auth/calendar']

def get_calendar_service():
    creds = None
    if GOOGLE_TOKEN_B64:
        creds = pickle.loads(base64.b64decode(GOOGLE_TOKEN_B64))
    elif os.path.exists('token.pickle'):
        with open('token.pickle', 'rb') as f:
            creds = pickle.load(f)
    if creds and creds.expired and creds.refresh_token:
        creds.refresh(Request())
    if not creds:
        raise Exception("Sin credenciales Google. Ejecuta setup_auth.py primero.")
    return build('calendar', 'v3', credentials=creds)

def parse_task_with_claude(user_message: str) -> dict:
    client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)
    now = datetime.now()
    weekdays_es = {"Monday":"lunes","Tuesday":"martes","Wednesday":"miércoles","Thursday":"jueves","Friday":"viernes","Saturday":"sábado","Sunday":"domingo"}
    
    prompt = f"""Convierte este mensaje en un evento de Google Calendar.
Fecha/hora actual: {now.strftime('%Y-%m-%d %H:%M')} ({weekdays_es.get(now.strftime('%A'))})
Mañana = {(now+timedelta(days=1)).strftime('%Y-%m-%d')}
Pasado mañana = {(now+timedelta(days=2)).strftime('%Y-%m-%d')}

Mensaje: "{user_message}"

Responde SOLO con JSON válido, sin texto extra ni backticks:
{{"titulo":"...","fecha":"YYYY-MM-DD","hora_inicio":"HH:MM","hora_fin":"HH:MM","descripcion":"","calendario":"personal","todo_el_dia":false,"recordatorio_minutos":30}}

Reglas calendario: "taller" si menciona taller/Jarvis/coches/reparaciones/chapas/piezas/pintura-taller. Todo lo demás: "personal".
Reglas recordatorio: urgente=10min, normal=30min, deadline=60min.
Sin hora → 09:00-09:30. Duración default 30min."""

    msg = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=400,
        messages=[{"role": "user", "content": prompt}]
    )
    text = re.sub(r'```json\n?|\n?```', '', msg.content[0].text).strip()
    return json.loads(text)

def create_calendar_event(parsed: dict) -> str:
    service = get_calendar_service()
    cal_id = CALENDAR_ID_TALLER if parsed.get("calendario") == "taller" else CALENDAR_ID_PERSONAL
    reminders = {'useDefault': False, 'overrides': [{'method': 'popup', 'minutes': parsed.get("recordatorio_minutos", 30)}]}
    
    if parsed.get("todo_el_dia"):
        event = {'summary': parsed["titulo"], 'description': parsed.get("descripcion",""),
                 'start': {'date': parsed["fecha"]}, 'end': {'date': parsed["fecha"]}, 'reminders': reminders}
    else:
        event = {'summary': parsed["titulo"], 'description': parsed.get("descripcion",""),
                 'start': {'dateTime': f"{parsed['fecha']}T{parsed['hora_inicio']}:00", 'timeZone': 'Europe/Madrid'},
                 'end': {'dateTime': f"{parsed['fecha']}T{parsed['hora_fin']}:00", 'timeZone': 'Europe/Madrid'},
                 'reminders': reminders}
    
    result = service.events().insert(calendarId=cal_id, body=event).execute()
    return result.get('htmlLink', '')

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ALLOWED_USER_ID: return
    await update.message.reply_text(
        "🗓️ *Bot de recordatorios activo*\n\n"
        "Escríbeme cualquier tarea en lenguaje natural.\n\n"
        "*Ejemplos:*\n"
        "• `Llamar al proveedor de pintura el jueves a las 10`\n"
        "• `Revisar presupuesto del taller mañana`\n"
        "• `Publicar en APlenaVista el viernes`\n"
        "• `Deadline entrega REMAKE el lunes`\n\n"
        "📅 /hoy — eventos de hoy\n"
        "📆 /semana — próximos 7 días", parse_mode='Markdown')

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ALLOWED_USER_ID: return
    msg = await update.message.reply_text("⏳ Creando evento...")
    try:
        parsed = parse_task_with_claude(update.message.text)
        link = create_calendar_event(parsed)
        emoji = "🔧" if parsed.get("calendario") == "taller" else "📅"
        hora = "Todo el día" if parsed.get("todo_el_dia") else f"{parsed['hora_inicio']} – {parsed['hora_fin']}"
        resp = (f"{emoji} *Evento creado*\n\n📌 {parsed['titulo']}\n📆 {parsed['fecha']}\n"
                f"⏰ {hora}\n🔔 {parsed.get('recordatorio_minutos',30)} min antes\n📁 {parsed.get('calendario','personal').capitalize()}")
        if link: resp += f"\n\n[Abrir en Calendar]({link})"
        await msg.edit_text(resp, parse_mode='Markdown')
    except json.JSONDecodeError:
        await msg.edit_text("❌ No entendí la fecha. Intenta: `tarea el lunes a las 10`", parse_mode='Markdown')
    except Exception as e:
        logger.error(f"Error: {e}")
        await msg.edit_text(f"❌ Error: {str(e)}")

async def ver_hoy(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ALLOWED_USER_ID: return
    try:
        service = get_calendar_service()
        now = datetime.now()
        s, e = now.strftime('%Y-%m-%dT00:00:00+01:00'), now.strftime('%Y-%m-%dT23:59:59+01:00')
        events = []
        for cid in [CALENDAR_ID_PERSONAL, CALENDAR_ID_TALLER]:
            events.extend(service.events().list(calendarId=cid, timeMin=s, timeMax=e, singleEvents=True, orderBy='startTime').execute().get('items',[]))
        if not events:
            await update.message.reply_text("📭 Sin eventos hoy."); return
        resp = f"📅 *Hoy — {now.strftime('%d/%m/%Y')}*\n\n"
        for ev in events:
            st = ev['start'].get('dateTime', ev['start'].get('date',''))
            resp += f"`{st[11:16] if 'T' in st else '──'}` {ev['summary']}\n"
        await update.message.reply_text(resp, parse_mode='Markdown')
    except Exception as e:
        await update.message.reply_text(f"❌ {str(e)}")

async def ver_semana(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ALLOWED_USER_ID: return
    try:
        service = get_calendar_service()
        now = datetime.now()
        s = now.strftime('%Y-%m-%dT00:00:00+01:00')
        e = (now+timedelta(days=7)).strftime('%Y-%m-%dT23:59:59+01:00')
        events = []
        for cid in [CALENDAR_ID_PERSONAL, CALENDAR_ID_TALLER]:
            events.extend(service.events().list(calendarId=cid, timeMin=s, timeMax=e, singleEvents=True, orderBy='startTime').execute().get('items',[]))
        if not events:
            await update.message.reply_text("📭 Sin eventos esta semana."); return
        dows = {"Monday":"Lun","Tuesday":"Mar","Wednesday":"Mié","Thursday":"Jue","Friday":"Vie","Saturday":"Sáb","Sunday":"Dom"}
        events.sort(key=lambda x: x['start'].get('dateTime',x['start'].get('date','')))
        resp, cur = "📆 *Próximos 7 días*\n", ""
        for ev in events:
            st = ev['start'].get('dateTime',ev['start'].get('date',''))
            day = st[:10]
            if day != cur:
                cur = day
                dt = datetime.strptime(day,'%Y-%m-%d')
                resp += f"\n*{dows.get(dt.strftime('%A'),'')} {dt.strftime('%d/%m')}*\n"
            resp += f"  `{st[11:16] if 'T' in st else '──'}` {ev['summary']}\n"
        await update.message.reply_text(resp, parse_mode='Markdown')
    except Exception as e:
        await update.message.reply_text(f"❌ {str(e)}")

def main():
    app = Application.builder().token(TELEGRAM_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("hoy", ver_hoy))
    app.add_handler(CommandHandler("semana", ver_semana))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    logger.info("🤖 Bot iniciado")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == '__main__':
    main()
