import asyncio, html, logging, os, re, time, datetime
import aiosqlite, aiohttp
from dotenv import load_dotenv
from aiogram import Bot, Dispatcher, Router, F, BaseMiddleware
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.storage.base import StorageKey
from aiogram.types import Message, CallbackQuery, InlineKeyboardMarkup, InlineKeyboardButton
TOKEN ="9017820023:AAFQDQg9PTJp--DF1D4XedwiODIG9m70tqk"
ADMINS = [6869945520]
DB_PATH = os.getenv("DB_PATH", "data/bot.sqlite3")
FF_API = os.getenv("FF_API_URL", "https://ff-info-api-tau.vercel.app/info")
MAX_PHOTOS = int(os.getenv("MAX_PHOTOS", "5"))
MAX_MB = int(os.getenv("MAX_FILE_MB", "5"))
RATE = int(os.getenv("RATE_LIMIT", "30"))

if not ADMINS:
    raise SystemExit("ADMIN_IDS فارغ")

SCHEMA = """
CREATE TABLE IF NOT EXISTS tickets(
  id INTEGER PRIMARY KEY AUTOINCREMENT, tg_id INTEGER NOT NULL, uid TEXT NOT NULL,
  nick TEXT, lvl TEXT, region TEXT, created_acc TEXT, first_bind TEXT,
  issue TEXT, still_login TEXT, verified INTEGER DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'new', created_at TEXT, updated_at TEXT);
CREATE TABLE IF NOT EXISTS proofs(
  id INTEGER PRIMARY KEY AUTOINCREMENT, ticket_id INTEGER NOT NULL, file_id TEXT NOT NULL);
CREATE TABLE IF NOT EXISTS events(
  id INTEGER PRIMARY KEY AUTOINCREMENT, ticket_id INTEGER, actor INTEGER, action TEXT, ts TEXT);
CREATE INDEX IF NOT EXISTS ix_open ON tickets(tg_id, status);
"""

def now():
    return datetime.datetime.utcnow().isoformat(timespec="seconds")

async def db_init():
    os.makedirs(os.path.dirname(DB_PATH) or ".", exist_ok=True)
    async with aiosqlite.connect(DB_PATH) as db:
        await db.executescript(SCHEMA)
        await db.commit()

async def open_ticket(tg_id):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("""SELECT * FROM tickets WHERE tg_id=? AND status!='closed'
                                  ORDER BY id DESC LIMIT 1""", (tg_id,))
        return await cur.fetchone()

async def get_ticket(tid):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("SELECT * FROM tickets WHERE id=?", (tid,))
        return await cur.fetchone()

async def create_ticket(tg_id, d):
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("""INSERT INTO tickets(tg_id,uid,nick,lvl,region,created_acc,
            first_bind,issue,still_login,verified,status,created_at,updated_at)
            VALUES(?,?,?,?,?,?,?,?,?,?,'new',?,?)""",
            (tg_id, d["uid"], d.get("nick"), d.get("lvl"), d.get("region"),
             d.get("created_acc"), d.get("first_bind"), d.get("issue"),
             d.get("still_login"), int(bool(d.get("verified"))), now(), now()))
        await db.commit()
        return cur.lastrowid

async def update_ticket(tid, **f):
    if not f:
        return
    f["updated_at"] = now()
    sets = ", ".join(f"{k}=?" for k in f)
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(f"UPDATE tickets SET {sets} WHERE id=?", (*f.values(), tid))
        await db.commit()

async def add_proof(tid, fid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("INSERT INTO proofs(ticket_id,file_id) VALUES(?,?)", (tid, fid))
        await db.commit()

async def get_proofs(tid):
    async with aiosqlite.connect(DB_PATH) as db:
        cur = await db.execute("SELECT file_id FROM proofs WHERE ticket_id=?", (tid,))
        return [r[0] for r in await cur.fetchall()]

async def log_event(tid, actor, action):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("INSERT INTO events(ticket_id,actor,action,ts) VALUES(?,?,?,?)",
                         (tid, actor, action, now()))
        await db.commit()

async def stats():
    async with aiosqlite.connect(DB_PATH) as db:
        out = {}
        for k, w in (("new", "status='new'"), ("sent", "status='sent'"),
                     ("answered", "status='answered'"), ("closed", "status='closed'")):
            cur = await db.execute(f"SELECT COUNT(*) FROM tickets WHERE {w}")
            out[k] = (await cur.fetchone())[0]
        return out

async def pending(limit=10):
    async with aiosqlite.connect(DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute("""SELECT id,uid,nick,issue FROM tickets
            WHERE status IN ('new','need_more') ORDER BY id LIMIT ?""", (limit,))
        return await cur.fetchall()

async def delete_user(tg_id):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM proofs WHERE ticket_id IN (SELECT id FROM tickets WHERE tg_id=?)", (tg_id,))
        await db.execute("DELETE FROM events WHERE ticket_id IN (SELECT id FROM tickets WHERE tg_id=?)", (tg_id,))
        await db.execute("DELETE FROM tickets WHERE tg_id=?", (tg_id,))
        await db.commit()

class Form(StatesGroup):
    uid = State(); confirm = State(); nick = State(); lvl = State()
    region = State(); created_acc = State(); first_bind = State()
    issue = State(); still_login = State(); proof = State()

REGIONS = ["MEA", "SA", "EG", "MA", "DZ", "TN", "EU", "ID", "BR", "PK", "IN", "TH", "VN"]
BINDS = ["Google", "Facebook", "Apple", "VK", "Huawei", "X"]
ISSUES = [("نسيت الإيميل", "forgot"), ("الإيميل تبدل بلا علمي", "changed"), ("الحساب مسروق", "stolen")]
ISSUE_EN = {"forgot": "Forgot recovery email",
            "changed": "Recovery email changed without notice",
            "stolen": "Account stolen"}

def start_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="بدا التيكيت", callback_data="begin")]])

def confirm_kb():
    return InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="نعم، هادا حسابي", callback_data="cfm:yes"),
        InlineKeyboardButton(text="لا", callback_data="cfm:no")]])

def grid(items, prefix):
    rows, row = [], []
    for it in items:
        label, val = (it, it) if isinstance(it, str) else it
        row.append(InlineKeyboardButton(text=label, callback_data=f"{prefix}:{val}"))
        if len(row) == 3:
            rows.append(row); row = []
    if row:
        rows.append(row)
    return InlineKeyboardMarkup(inline_keyboard=rows)

def yesno_kb():
    return InlineKeyboardMarkup(inline_keyboard=[[
        InlineKeyboardButton(text="نعم", callback_data="lg:yes"),
        InlineKeyboardButton(text="لا", callback_data="lg:no")]])

def done_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="سالي، صيفط", callback_data="proof:done")]])

def admin_kb(tid):
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="نص التيكيت", callback_data=f"a:copy:{tid}"),
         InlineKeyboardButton(text="طلب إثبات إضافي", callback_data=f"a:more:{tid}")],
        [InlineKeyboardButton(text="صيفتت", callback_data=f"a:sent:{tid}"),
         InlineKeyboardButton(text="سد", callback_data=f"a:close:{tid}")]])

def status_kb():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="حالة التيكيت", callback_data="status")]])

WELCOME = ("بوت تذاكر استرجاع حساب فري فاير.\n\n"
           "غادي نجمع منك المعلومات اللي كيطلبها الدعم ونصيفطها للأدمن ف تيكيت مرتب.\n"
           "خاصك تكون كتعرف الـ UID ديالك.")

def status_text(t):
    n = {"new": "تسجل، كيتسنى المراجعة", "need_more": "محتاج إثبات إضافي",
         "sent": "تصيفط للدعم", "answered": "جاوب الدعم", "closed": "مسدود"}
    return f"تيكيت #{t['id']}\nUID: {t['uid']}\nالحالة: {n.get(t['status'], t['status'])}"

def ticket_text(t, proofs):
    return (
        "Free Fire account recovery request\n\n"
        f"Player UID: {t['uid']}\nIn-game name: {t['nick']}\nLevel: {t['lvl']}\n"
        f"Region: {t['region']}\nAccount created (approx): {t['created_acc']}\n"
        f"First bound platform: {t['first_bind']}\n"
        f"Issue: {ISSUE_EN.get(t['issue'], t['issue'])}\n"
        f"Still able to log in via secondary binding: {t['still_login']}\n"
        f"Proof files attached: {proofs}\n\n"
        "Player still holds a valid secondary binding but cannot access the recovery "
        "email. Please assist with resetting the recovery email.")

async def ff_lookup(uid):
    if not FF_API:
        return None
    try:
        async with aiohttp.ClientSession() as s:
            async with s.get(f"{FF_API}?uid={uid}", timeout=aiohttp.ClientTimeout(total=8)) as r:
                if r.status != 200:
                    return None
                data = await r.json(content_type=None)
        info = data.get("data") or data
        nick = info.get("nickname") or info.get("name")
        if not nick:
            return None
        return {"nick": nick, "lvl": str(info.get("level") or ""),
                "region": (info.get("region") or "").upper()}
    except Exception:
        return None

class Throttle(BaseMiddleware):
    def __init__(self):
        self.hits = {}

    async def __call__(self, handler, event, data):
        u = data.get("event_from_user")
        if u is None or isinstance(event, CallbackQuery):
            return await handler(event, data)
        t = time.time()
        h = [x for x in self.hits.get(u.id, []) if t - x < 3600]
        if len(h) >= RATE:
            await event.answer("بزاف من الرسائل. تسنى شي دقيقة.")
            return
        h.append(t); self.hits[u.id] = h
        return await handler(event, data)

user = Router()
admin = Router()
admin.message.filter(F.from_user.id.in_(ADMINS))
admin.callback_query.filter(F.from_user.id.in_(ADMINS))

@user.message(Command("start"))
async def cmd_start(m: Message, state: FSMContext):
    await state.clear()
    t = await open_ticket(m.from_user.id)
    if t:
        await m.answer(status_text(t), reply_markup=status_kb()); return
    await m.answer(WELCOME, reply_markup=start_kb())

@user.message(Command("help"))
async def cmd_help(m: Message):
    await m.answer("/start تيكيت جديد\n/ticket حالة التيكيت\n/cancel إلغاء\n/delete حذف بياناتي")

@user.message(Command("cancel"))
async def cmd_cancel(m: Message, state: FSMContext):
    await state.clear()
    await m.answer("تلغى. /start باش تبدا من جديد.")

@user.callback_query(F.data == "status")
@user.message(Command("ticket"))
async def cmd_status(e, state: FSMContext):
    t = await open_ticket(e.from_user.id)
    txt = status_text(t) if t else "ما عندكش تيكيت مفتوح."
    if isinstance(e, CallbackQuery):
        await e.message.answer(txt); await e.answer()
    else:
        await e.answer(txt)

@user.message(Command("delete"))
async def cmd_delete(m: Message, state: FSMContext):
    await delete_user(m.from_user.id); await state.clear()
    await m.answer("تحيدت كل البيانات ديالك.")

@user.callback_query(F.data == "begin")
async def cb_begin(c: CallbackQuery, state: FSMContext):
    if await open_ticket(c.from_user.id):
        await c.answer("عندك تيكيت مفتوح.", show_alert=True); return
    await state.set_state(Form.uid)
    await c.message.answer("أرسل الـ UID ديال الحساب (6 إلى 10 أرقام):")
    await c.answer()

@user.message(Form.uid)
async def st_uid(m: Message, state: FSMContext):
    v = (m.text or "").strip()
    if not (v.isdigit() and 6 <= len(v) <= 10):
        await m.answer("UID غير صالح. أرقام فقط، من 6 لـ 10."); return
    await state.update_data(uid=v)
    info = await ff_lookup(v)
    if info:
        await state.update_data(nick=info["nick"], lvl=info["lvl"],
                                region=info["region"], verified=True)
        await state.set_state(Form.confirm)
        await m.answer(f"تحققنا من الـ UID.\nالاسم: {info['nick']}\nالمستوى: {info['lvl'] or '-'}"
                       f"\nالمنطقة: {info['region'] or '-'}\n\nواش هادا حسابك؟",
                       reply_markup=confirm_kb())
        return
    await state.set_state(Form.nick)
    await m.answer("ما قدرتش نتحقق من الـ UID. أرسل الاسم داخل اللعبة:")

@user.callback_query(Form.confirm, F.data == "cfm:yes")
async def st_yes(c: CallbackQuery, state: FSMContext):
    d = await state.get_data()
    if not d.get("region"):
        await state.set_state(Form.region)
        await c.message.answer("المنطقة / السيرفر:", reply_markup=grid(REGIONS, "rg"))
    else:
        await state.set_state(Form.created_acc)
        await c.message.answer("تاريخ تقريبي لإنشاء الحساب، بصيغة YYYY-MM:")
    await c.answer()

@user.callback_query(Form.confirm, F.data == "cfm:no")
async def st_no(c: CallbackQuery, state: FSMContext):
    await state.set_state(Form.nick)
    await c.message.answer("أرسل الاسم داخل اللعبة:"); await c.answer()

@user.message(Form.nick)
async def st_nick(m: Message, state: FSMContext):
    v = (m.text or "").strip()
    if not 3 <= len(v) <= 20:
        await m.answer("الاسم بين 3 و 20 حرف."); return
    await state.update_data(nick=v)
    await state.set_state(Form.lvl)
    await m.answer("المستوى (رقم من 1 لـ 100):")

@user.message(Form.lvl)
async def st_lvl(m: Message, state: FSMContext):
    v = (m.text or "").strip()
    if not (v.isdigit() and 1 <= int(v) <= 100):
        await m.answer("رقم من 1 لـ 100."); return
    await state.update_data(lvl=v)
    await state.set_state(Form.region)
    await m.answer("المنطقة / السيرفر:", reply_markup=grid(REGIONS, "rg"))

@user.callback_query(Form.region, F.data.startswith("rg:"))
async def st_region(c: CallbackQuery, state: FSMContext):
    await state.update_data(region=c.data.split(":", 1)[1])
    await state.set_state(Form.created_acc)
    await c.message.answer("تاريخ تقريبي لإنشاء الحساب، بصيغة YYYY-MM:"); await c.answer()

@user.message(Form.created_acc)
async def st_created(m: Message, state: FSMContext):
    v = (m.text or "").strip()
    if not re.fullmatch(r"\d{4}-(0[1-9]|1[0-2])", v):
        await m.answer("الصيغة YYYY-MM، مثال 2021-06."); return
    await state.update_data(created_acc=v)
    await state.set_state(Form.first_bind)
    await m.answer("أول منصة ربطتها بالحساب:", reply_markup=grid(BINDS, "bd"))

@user.callback_query(Form.first_bind, F.data.startswith("bd:"))
async def st_bind(c: CallbackQuery, state: FSMContext):
    await state.update_data(first_bind=c.data.split(":", 1)[1])
    await state.set_state(Form.issue)
    await c.message.answer("شنو المشكل بالضبط؟", reply_markup=grid(ISSUES, "is")); await c.answer()

@user.callback_query(Form.issue, F.data.startswith("is:"))
async def st_issue(c: CallbackQuery, state: FSMContext):
    await state.update_data(issue=c.data.split(":", 1)[1])
    await state.set_state(Form.still_login)
    await c.message.answer("واش مازال كتقدر تدخل للحساب بالربط الثانوي؟",
                           reply_markup=yesno_kb())
    await c.answer()

@user.callback_query(Form.still_login, F.data.startswith("lg:"))
async def st_login(c: CallbackQuery, state: FSMContext):
    await state.update_data(still_login="yes" if c.data.endswith("yes") else "no", proofs=[])
    await state.set_state(Form.proof)
    await c.message.answer(f"صيفط الإثباتات: سكرينشوتات قديمة، إيصالات شحن، أي دليل.\n"
                           f"من 1 لـ {MAX_PHOTOS} صور. ملي تسالي اضغط الزر.",
                           reply_markup=done_kb())
    await c.answer()

@user.message(Form.proof, F.photo)
async def st_photo(m: Message, state: FSMContext):
    d = await state.get_data()
    p = d.get("proofs", [])
    if len(p) >= MAX_PHOTOS:
        await m.answer(f"وصلتي للحد ({MAX_PHOTOS}). اضغط الزر باش تسالي."); return
    f = m.photo[-1]
    if f.file_size and f.file_size > MAX_MB * 1024 * 1024:
        await m.answer(f"الصورة كبيرة، الحد {MAX_MB}MB."); return
    p.append(f.file_id)
    await state.update_data(proofs=p)
    await m.answer(f"تزادت ({len(p)}/{MAX_PHOTOS}).", reply_markup=done_kb())

@user.callback_query(Form.proof, F.data == "proof:done")
async def st_done(c: CallbackQuery, state: FSMContext):
    d = await state.get_data()
    p = d.get("proofs", [])
    if not p:
        await c.answer("خاصك صورة وحدة على الأقل.", show_alert=True); return
    if d.get("tid"):
        tid = d["tid"]
        for fid in p:
            await add_proof(tid, fid)
        await update_ticket(tid, status="new")
        await log_event(tid, c.from_user.id, "proof_added")
        await state.clear()
        await c.message.answer(f"تزادت الإثباتات فالتيكيت #{tid}. عاودناه للمراجعة.")
    else:
        tid = await create_ticket(c.from_user.id, d)
        for fid in p:
            await add_proof(tid, fid)
        await log_event(tid, c.from_user.id, "created")
        await state.clear()
        await c.message.answer(f"تسجل التيكيت #{tid}. غادي نتواصلو معاك هنا.\n"
                               f"ما تعاودش تصيفط نفس الطلب.", reply_markup=status_kb())
        head = (f"تيكيت #{tid}\nUID: {d['uid']}\nNick: {d.get('nick')}\nLevel: {d.get('lvl')}\n"
                f"Region: {d.get('region')}\nCreated: {d.get('created_acc')}\n"
                f"First bind: {d.get('first_bind')}\nIssue: {d.get('issue')}\n"
                f"Still login: {d.get('still_login')}\n"
                f"Verified: {'yes' if d.get('verified') else 'no'}")
        for a in ADMINS:
            try:
                await c.bot.send_message(a, head, reply_markup=admin_kb(tid))
                for fid in p:
                    await c.bot.send_photo(a, fid)
            except Exception:
                pass
    await c.answer()

@admin.message(Command("stats"))
async def a_stats(m: Message):
    s = await stats()
    await m.answer(f"جديد: {s['new']}\nمصيفط: {s['sent']}\nمجاوب: {s['answered']}\nمسدود: {s['closed']}")

@admin.message(Command("pending"))
async def a_pending(m: Message):
    rows = await pending()
    if not rows:
        await m.answer("ما كاين حتى تيكيت كيتسنى."); return
    await m.answer("\n".join(f"#{r['id']} | {r['uid']} | {r['nick']} | {r['issue']}" for r in rows))

@admin.message(Command("reply"))
async def a_reply(m: Message):
    parts = (m.text or "").split(maxsplit=2)
    if len(parts) < 3:
        await m.answer("الصيغة: /reply TG_ID الرسالة"); return
    try:
        await m.bot.send_message(int(parts[1]), parts[2])
    except Exception as e:
        await m.answer(f"فشل: {e}"); return
    await m.answer("توصل.")

@admin.callback_query(F.data.startswith("a:"))
async def a_actions(c: CallbackQuery, dispatcher):
    _, act, tid_s = c.data.split(":")
    tid = int(tid_s)
    t = await get_ticket(tid)
    if not t:
        await c.answer("تيكيت غير موجود", show_alert=True); return

    if act == "copy":
        proofs = await get_proofs(tid)
        await c.message.answer(f"<code>{html.escape(ticket_text(t, len(proofs)))}</code>")
    elif act == "sent":
        await update_ticket(tid, status="sent"); await log_event(tid, c.from_user.id, "sent")
        await c.bot.send_message(t["tg_id"], f"تيكيتك #{tid} تصيفط للدعم.")
        await c.answer("تسجل كمصيفط")
    elif act == "close":
        await update_ticket(tid, status="closed"); await log_event(tid, c.from_user.id, "closed")
        await c.bot.send_message(t["tg_id"], f"تيكيتك #{tid} تسد.")
        await c.answer("تسجل كمسدود")
    elif act == "more":
        await update_ticket(tid, status="need_more"); await log_event(tid, c.from_user.id, "need_more")
        key = StorageKey(bot_id=c.bot.id, chat_id=t["tg_id"], user_id=t["tg_id"])
        await dispatcher.storage.set_state(key, Form.proof)
        await dispatcher.storage.set_data(key, {"tid": tid, "proofs": []})
        await c.bot.send_message(t["tg_id"],
            f"التيكيت #{tid} محتاج إثبات إضافي. صيفط صور زيادة، ومن بعد اضغط الزر.")
        await c.answer("طلبنا إثبات إضافي")

    if act != "copy":
        await c.message.edit_reply_markup(reply_markup=None)

async def main():
    logging.basicConfig(level=logging.INFO)
    await db_init()
    bot = Bot(TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
    dp = Dispatcher(storage=MemoryStorage())
    dp.message.middleware(Throttle())
    dp.include_router(admin)
    dp.include_router(user)
    await bot.delete_webhook(drop_pending_updates=True)
    print("البوت خدام")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
