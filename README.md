import asyncio
from aiogram import Bot, Dispatcher, types, F
from aiogram.types import (
    InlineKeyboardMarkup, InlineKeyboardButton,
    LabeledPrice, PreCheckoutQuery
)
from aiogram.filters import Command
from aiogram.types import Message, CallbackQuery
from aiogram.client.default import DefaultBotProperties
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
import sqlite3
from datetime import datetime, timedelta

TOKEN = "8712350863:AAEFA-P-3jgnKlv6CW1NLXDGs4mQelpmdQM"
GROUP_ID = -1003846960941
ADMIN_ID = 7832555448
ADMIN_CMD = "admin123890"

TARIFFS = {
    "12h": {"hours": 12, "price": 50, "name": "📢 12 ЧАСОВ"},
    "24h": {"hours": 24, "price": 100, "name": "📢 24 ЧАСА"},
    "48h": {"hours": 48, "price": 150, "name": "📢 48 ЧАСОВ"},
}
DONATE_AMOUNTS = [10, 25, 50, 100, 250]

def init_db():
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("CREATE TABLE IF NOT EXISTS tags (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, username TEXT, tag TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)")
    c.execute("CREATE TABLE IF NOT EXISTS ads (ad_id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, username TEXT, text TEXT, duration_hours INTEGER, price INTEGER, message_id INTEGER, expire_time DATETIME, status TEXT DEFAULT 'pending', decline_reason TEXT, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)")
    c.execute("CREATE TABLE IF NOT EXISTS pins (pin_id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, username TEXT, ad_id INTEGER, price INTEGER, message_id INTEGER, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)")
    c.execute("CREATE TABLE IF NOT EXISTS donations (don_id INTEGER PRIMARY KEY AUTOINCREMENT, user_id INTEGER, username TEXT, amount INTEGER, created_at DATETIME DEFAULT CURRENT_TIMESTAMP)")
    c.execute("CREATE TABLE IF NOT EXISTS balances (user_id INTEGER PRIMARY KEY, username TEXT, balance INTEGER DEFAULT 0)")
    conn.commit()
    conn.close()

bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode="HTML"))
dp = Dispatcher(storage=MemoryStorage())

class AdStates(StatesGroup):
    waiting_ad_text = State()
    waiting_decline = State()

def main_menu():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🏷️ КУПИТЬ ТЕГ (50 ⭐)", callback_data="buy_tag")],
        [InlineKeyboardButton(text="📢 ЗАКАЗАТЬ РЕКЛАМУ", callback_data="ad_menu")],
        [InlineKeyboardButton(text="📌 ЗАКРЕПИТЬ РЕКЛАМУ (50 ⭐)", callback_data="pin_ad")],
        [InlineKeyboardButton(text="💎 ДОНАТ", callback_data="donate_menu")],
        [InlineKeyboardButton(text="🏆 ТОП ДОНАТЕРОВ", callback_data="top_donors")],
        [InlineKeyboardButton(text="ℹ️ ПОМОЩЬ", callback_data="help")],
    ])

@dp.message(Command("start"))
async def start_cmd(message: Message):
    await message.answer("🔥 <b>LOST SQUAD BOT</b>\n\n🏷️ Тег — 50 ⭐\n📢 Реклама — от 50 ⭐\n📌 Закреп — 50 ⭐\n💎 Донат — от 10 ⭐\n\n👇 Выбирай:", reply_markup=main_menu())

@dp.callback_query(F.data == "help")
async def help_cmd(c: CallbackQuery):
    await c.message.edit_text("ℹ️ <b>ПОМОЩЬ</b>\n\n🏷️ Тег — платишь, получаешь тег в группе\n📢 Реклама — платишь → текст → модерация → пост\n📌 Закреп — только если есть активная реклама\n💎 Донат — поддержи сквад\n\n❓ Вопросы к админу.", reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")]]))

# ==================== ТЕГ ====================
@dp.callback_query(F.data == "buy_tag")
async def buy_tag_start(c: CallbackQuery):
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT balance FROM balances WHERE user_id = ?", (c.from_user.id,))
    bal = c.fetchone()
    conn.close()
    if bal and bal[0] >= 50:
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("UPDATE balances SET balance = balance - 50 WHERE user_id = ?", (c.from_user.id,))
        conn.commit()
        conn.close()
        await give_tag(c.from_user.id, c.from_user.username)
        await c.message.edit_text("✅ Тег выдан с баланса!", reply_markup=main_menu())
        return
    await bot.send_invoice(chat_id=c.from_user.id, title="Покупка тега", description="Тег LOST SQUAD", payload=f"tag_{c.from_user.id}", provider_token="", currency="XTR", prices=[LabeledPrice(label="Тег", amount=50)], start_parameter="tag")
    await c.answer("✅ Счёт!")

async def give_tag(user_id, username):
    try:
        await bot.promote_chat_member(GROUP_ID, user_id, is_anonymous=False)
        await bot.set_chat_administrator_custom_title(GROUP_ID, user_id, "LOST | Участник")
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("INSERT INTO tags (user_id, username, tag) VALUES (?, ?, ?)", (user_id, username, "LOST | Участник"))
        conn.commit()
        conn.close()
        await bot.send_message(GROUP_ID, f"🏷️ @{username} получил тег: <b>LOST | Участник</b>")
    except Exception as e:
        await bot.send_message(ADMIN_ID, f"Ошибка тега: {e}")

# ==================== РЕКЛАМА ====================
@dp.callback_query(F.data == "ad_menu")
async def ad_menu(c: CallbackQuery):
    kb = [[InlineKeyboardButton(text=f"{v['name']} — {v['price']} ⭐", callback_data=f"ad_pay|{k}")] for k, v in TARIFFS.items()]
    kb.append([InlineKeyboardButton(text="🔄 У меня есть оплаченная", callback_data="retry_ad")])
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    await c.message.edit_text("📢 <b>ЗАКАЗ РЕКЛАМЫ</b>\n\nВыбери тариф:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("ad_pay|"))
async def ad_pay(c: CallbackQuery):
    tariff_key = c.data.split("|")[1]
    tariff = TARIFFS[tariff_key]
    await bot.send_invoice(chat_id=c.from_user.id, title=f"Реклама {tariff['hours']}ч", description=f"Реклама в LOST SQUAD на {tariff['hours']}ч", payload=f"ad_{c.from_user.id}_{tariff_key}", provider_token="", currency="XTR", prices=[LabeledPrice(label=f"Реклама {tariff['hours']}ч", amount=tariff["price"])], start_parameter="ad")
    await c.answer("✅ Счёт!")

@dp.callback_query(F.data == "retry_ad")
async def retry_ad(c: CallbackQuery, state: FSMContext):
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT ad_id, duration_hours, price FROM ads WHERE user_id = ? AND status = 'declined' ORDER BY created_at DESC LIMIT 1", (c.from_user.id,))
    ad = c.fetchone()
    conn.close()
    if not ad:
        await c.answer("❌ Нет отклонённых реклам.")
        return
    await state.update_data(ad_hours=ad[1], ad_price=ad[2], retry_ad_id=ad[0])
    await c.message.edit_text("📝 Отправь исправленный текст:", reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🔙 ОТМЕНА", callback_data="main_menu")]]))
    await state.set_state(AdStates.waiting_ad_text)

# ==================== ЗАКРЕП ====================
@dp.callback_query(F.data == "pin_ad")
async def pin_ad_start(c: CallbackQuery):
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT ad_id, text FROM ads WHERE user_id = ? AND status = 'active' ORDER BY created_at DESC LIMIT 5", (c.from_user.id,))
    ads = c.fetchall()
    conn.close()
    if not ads:
        await c.answer("❌ У тебя нет активной рекламы! Сначала закажи.", show_alert=True)
        return
    kb = [[InlineKeyboardButton(text=f"#{a[0]}: {a[1][:30]}...", callback_data=f"pin_pay|{a[0]}")] for a in ads]
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    await c.message.edit_text("📌 <b>ЗАКРЕПИТЬ РЕКЛАМУ (50 ⭐)</b>\n\nВыбери какую:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("pin_pay|"))
async def pin_pay(c: CallbackQuery):
    ad_id = int(c.data.split("|")[1])
    await bot.send_invoice(chat_id=c.from_user.id, title="Закреп рекламы", description=f"Закреп рекламы #{ad_id}", payload=f"pin_{c.from_user.id}_{ad_id}", provider_token="", currency="XTR", prices=[LabeledPrice(label="Закреп", amount=50)], start_parameter="pin")
    await c.answer("✅ Счёт!")

# ==================== ДОНАТ ====================
@dp.callback_query(F.data == "donate_menu")
async def donate_menu_cb(c: CallbackQuery):
    kb = [[InlineKeyboardButton(text=f"💎 {a} ⭐", callback_data=f"donate|{a}")] for a in DONATE_AMOUNTS]
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    await c.message.edit_text("💎 <b>ДОНАТ В LOST SQUAD</b>\n\nВыбери сумму:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("donate|"))
async def donate_pay(c: CallbackQuery):
    amount = int(c.data.split("|")[1])
    await bot.send_invoice(chat_id=c.from_user.id, title="Донат LOST SQUAD", description=f"Донат {amount} ⭐", payload=f"don_{c.from_user.id}_{amount}", provider_token="", currency="XTR", prices=[LabeledPrice(label=f"Донат {amount} ⭐", amount=amount)], start_parameter="don")
    await c.answer("✅ Счёт!")

@dp.callback_query(F.data == "top_donors")
async def top_donors(c: CallbackQuery):
    conn = sqlite3.connect("lost.db")
    cur = conn.cursor()
    cur.execute("SELECT username, SUM(amount) FROM donations GROUP BY user_id ORDER BY SUM(amount) DESC LIMIT 10")
    donors = cur.fetchall()
    conn.close()
    txt = "🏆 <b>ЛУЧШИЕ ДОНАТЕРЫ LOST SQUAD</b>\n\n"
    if not donors:
        txt += "Пока никто не донатил. Будь первым!"
    else:
        medals = ["🥇","🥈","🥉"] + ["🏅"]*7
        for i, d in enumerate(donors):
            txt += f"{medals[i]} @{d[0]} — <b>{d[1]} ⭐</b>\n"
    await c.message.edit_text(txt, reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="💎 ЗАДОНАТИТЬ", callback_data="donate_menu")],[InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")]]))

# ==================== ПЛАТЕЖИ ====================
@dp.pre_checkout_query()
async def pre_checkout(query: PreCheckoutQuery):
    await bot.answer_pre_checkout_query(query.id, ok=True)

@dp.message(F.successful_payment)
async def payment_success(message: Message, state: FSMContext):
    payload = message.successful_payment.invoice_payload
    parts = payload.split("_")
    ptype, user_id = parts[0], int(parts[1])

    if ptype == "tag":
        await give_tag(user_id, message.from_user.username)
        await message.answer("✅ <b>ТЕГ ВЫДАН!</b>\nТвой тег: <b>LOST | Участник</b>", reply_markup=main_menu())

    elif ptype == "ad":
        tariff_key = parts[2]
        tariff = TARIFFS[tariff_key]
        await state.update_data(ad_hours=tariff["hours"], ad_price=tariff["price"])
        await message.answer(f"✅ Оплачено {tariff['price']} ⭐\n📝 Отправь текст рекламы:", reply_markup=InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="🔙 ОТМЕНА", callback_data="main_menu")]]))
        await state.set_state(AdStates.waiting_ad_text)

    elif ptype == "pin":
        ad_id = int(parts[2])
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("SELECT message_id, status FROM ads WHERE ad_id = ? AND user_id = ?", (ad_id, user_id))
        ad = c.fetchone()
        if not ad or ad[1] != "active":
            conn.close()
            await message.answer("❌ Реклама не активна.", reply_markup=main_menu())
            return
        try:
            await bot.pin_chat_message(GROUP_ID, ad[0], disable_notification=True)
            c.execute("INSERT INTO pins (user_id, username, ad_id, price, message_id) VALUES (?, ?, ?, ?, ?)", (user_id, message.from_user.username, ad_id, 50, ad[0]))
            conn.commit()
            await message.answer(f"✅ Реклама #{ad_id} закреплена!", reply_markup=main_menu())
        except Exception as e:
            await message.answer(f"❌ Ошибка: {e}")
        conn.close()

    elif ptype == "don":
        amount = int(parts[2])
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("INSERT INTO donations (user_id, username, amount) VALUES (?, ?, ?)", (user_id, message.from_user.username, amount))
        conn.commit()
        conn.close()
        await message.answer(f"💎 Спасибо за донат <b>{amount} ⭐</b>!", reply_markup=main_menu())
        await bot.send_message(GROUP_ID, f"💎 @{message.from_user.username} задонатил <b>{amount} ⭐</b> в сквад!")

# ==================== ТЕКСТ РЕКЛАМЫ ====================
@dp.message(AdStates.waiting_ad_text)
async def receive_ad_text(message: Message, state: FSMContext):
    text = message.text or ""
    if len(text) < 5:
        await message.answer("❌ Минимум 5 символов.")
        return
    data = await state.get_data()
    hours = data.get("ad_hours", 12)
    price = data.get("ad_price", 50)
    retry_id = data.get("retry_ad_id")
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    if retry_id:
        c.execute("UPDATE ads SET text = ?, status = 'pending', decline_reason = NULL WHERE ad_id = ?", (text, retry_id))
        ad_id = retry_id
    else:
        c.execute("INSERT INTO ads (user_id, username, text, duration_hours, price, status) VALUES (?, ?, ?, ?, ?, 'pending')", (message.from_user.id, message.from_user.username, text, hours, price))
        ad_id = c.lastrowid
    conn.commit()
    conn.close()
    kb = InlineKeyboardMarkup(inline_keyboard=[[InlineKeyboardButton(text="✅ ОДОБРИТЬ", callback_data=f"approve|{ad_id}")],[InlineKeyboardButton(text="❌ ОТКЛОНИТЬ", callback_data=f"decline|{ad_id}")]])
    await bot.send_message(ADMIN_ID, f"📢 <b>РЕКЛАМА #{ad_id}</b>\n👤 @{message.from_user.username}\n⏱ {hours}ч | 💰 {price} ⭐\n\n📝 {text}", reply_markup=kb)
    await message.answer(f"✅ Реклама #{ad_id} на модерации.", reply_markup=main_menu())
    await state.clear()

# ==================== МОДЕРАЦИЯ ====================
@dp.callback_query(F.data.startswith("approve|"))
async def approve_ad(c: CallbackQuery):
    if c.from_user.id != ADMIN_ID: return
    ad_id = int(c.data.split("|")[1])
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT * FROM ads WHERE ad_id = ? AND status = 'pending'", (ad_id,))
    ad = c.fetchone()
    if not ad:
        conn.close()
        await c.answer("Уже обработана.")
        return
    try:
        msg = await bot.send_message(GROUP_ID, f"📢 <b>РЕКЛАМА</b>\n👤 @{ad[2]}\n⏱ {ad[4]}ч\n\n{ad[3]}")
        expire = datetime.now() + timedelta(hours=ad[4])
        c.execute("UPDATE ads SET status = 'active', message_id = ?, expire_time = ? WHERE ad_id = ?", (msg.message_id, expire, ad_id))
        conn.commit()
        conn.close()
        await c.message.edit_text(f"✅ #{ad_id} одобрена!")
        try:
            await bot.send_message(ad[1], f"✅ Реклама #{ad_id} в группе на {ad[4]}ч!")
        except:
            pass
        asyncio.create_task(auto_delete(msg.message_id, ad[4], ad_id))
    except Exception as e:
        await c.answer(f"Ошибка: {e}")

@dp.callback_query(F.data.startswith("decline|"))
async def decline_ad(c: CallbackQuery, state: FSMContext):
    if c.from_user.id != ADMIN_ID: return
    ad_id = int(c.data.split("|")[1])
    await state.update_data(decline_id=ad_id)
    await c.message.edit_text(f"❌ Отклоняешь #{ad_id}. Напиши причину:")
    await state.set_state(AdStates.waiting_decline)

@dp.message(AdStates.waiting_decline)
async def decline_reason(message: Message, state: FSMContext):
    data = await state.get_data()
    ad_id = data.get("decline_id")
    reason = message.text or "Без причины"
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("UPDATE ads SET status = 'declined', decline_reason = ? WHERE ad_id = ?", (reason, ad_id))
    c.execute("SELECT user_id FROM ads WHERE ad_id = ?", (ad_id,))
    u = c.fetchone()
    conn.commit()
    conn.close()
    await message.answer(f"✅ #{ad_id} отклонена.")
    if u:
        try:
            await bot.send_message(u[0], f"❌ Реклама #{ad_id} отклонена.\nПричина: {reason}\n✏️ Исправь и отправь снова — жми «У меня есть оплаченная».")
        except:
            pass
    await state.clear()

async def auto_delete(msg_id, hours, ad_id):
    await asyncio.sleep(hours * 3600)
    try:
        await bot.delete_message(GROUP_ID, msg_id)
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("UPDATE ads SET status = 'deleted' WHERE ad_id = ?", (ad_id,))
        conn.commit()
        conn.close()
    except:
        pass

# ==================== АДМИНКА ====================
@dp.message(Command(ADMIN_CMD))
async def admin_cmd(message: Message):
    if message.from_user.id != ADMIN_ID:
        return
    args = message.text.split()
    if len(args) == 1:
        await message.answer("🔐 <b>АДМИНКА LOST SQUAD</b>\n\n/admin123890 stats\n/admin123890 giveaway ID 50 — выдать ⭐\n/admin123890 broadcast текст")
    elif args[1] == "stats":
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("SELECT COUNT(*) FROM tags")
        tags = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM ads")
        ads = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(amount), 0) FROM donations")
        dons = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(price), 0) FROM ads WHERE status = 'active'")
        rev = c.fetchone()[0]
        conn.close()
        await message.answer(f"📊 <b>СТАТИСТИКА</b>\n\n🏷️ Тегов: {tags}\n📢 Реклам: {ads}\n💎 Донатов: {dons} ⭐\n💰 Выручка реклам: {rev} ⭐")
    elif args[1] == "giveaway" and len(args) >= 4:
        try:
            uid = int(args[2])
            amt = int(args[3])
        except:
            await message.answer("❌ /admin123890 giveaway ID СУММА")
            return
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("INSERT OR IGNORE INTO balances (user_id, username, balance) VALUES (?, ?, 0)", (uid, "unknown"))
        c.execute("UPDATE balances SET balance = balance + ? WHERE user_id = ?", (amt, uid))
        conn.commit()
        c.execute("SELECT balance FROM balances WHERE user_id = ?", (uid,))
        nb = c.fetchone()[0]
        conn.close()
        await message.answer(f"✅ +{amt} ⭐ юзеру <code>{uid}</code>\nБаланс: {nb} ⭐")
        try:
            await bot.send_message(uid, f"🎁 Админ выдал тебе <b>{amt} ⭐</b>!\nБаланс: {nb} ⭐\nМожешь купить тег или рекламу.")
        except:
            pass
    elif args[1] == "broadcast" and len(args) >= 3:
        text = " ".join(args[2:])
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("SELECT DISTINCT user_id FROM donations UNION SELECT DISTINCT user_id FROM ads UNION SELECT DISTINCT user_id FROM tags")
        users = c.fetchall()
        conn.close()
        s = 0
        for u in users:
            try:
                await bot.send_message(u[0], f"📢 {text}")
                s += 1
            except:
                pass
            await asyncio.sleep(0.05)
        await message.answer(f"✅ Рассылка: {s}/{len(users)}")

# ==================== ГЛАВНОЕ МЕНЮ ====================
@dp.callback_query(F.data == "main_menu")
async def main_menu_cb(c: CallbackQuery, state: FSMContext):
    await state.clear()
    await c.message.edit_text("🔥 <b>LOST SQUAD BOT</b>\n\n🏷️ Тег — 50 ⭐\n📢 Реклама — от 50 ⭐\n📌 Закреп — 50 ⭐\n💎 Донат — от 10 ⭐\n\n👇 Выбирай:", reply_markup=main_menu())

async def main():
    init_db()
    print("🔥 LOST SQUAD BOT ЗАПУЩЕН!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
