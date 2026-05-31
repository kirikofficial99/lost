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

# ==================== КОНФИГ ====================
TOKEN = "8712350863:AAEFA-P-3jgnKlv6CW1NLXDGs4mQelpmdQM"
GROUP_ID = -1003846960941
ADMIN_ID = 7832555448

TARIFFS = {
    "12h": {"hours": 12, "price": 50,  "name": "📢 12 ЧАСОВ"},
    "24h": {"hours": 24, "price": 100, "name": "📢 24 ЧАСА"},
    "48h": {"hours": 48, "price": 150, "name": "📢 48 ЧАСОВ"},
}

DONATE_AMOUNTS = [10, 25, 50, 100, 250]

# ==================== БАЗА ДАННЫХ ====================
def init_db():
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()

    c.execute("""CREATE TABLE IF NOT EXISTS tags (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        tag TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")

    c.execute("""CREATE TABLE IF NOT EXISTS ads (
        ad_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        text TEXT,
        duration_hours INTEGER,
        price INTEGER,
        message_id INTEGER,
        expire_time DATETIME,
        status TEXT DEFAULT 'pending',
        decline_reason TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")

    c.execute("""CREATE TABLE IF NOT EXISTS pins (
        pin_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        ad_id INTEGER,
        price INTEGER,
        message_id INTEGER,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")

    c.execute("""CREATE TABLE IF NOT EXISTS donations (
        don_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        amount INTEGER,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")

    conn.commit()
    conn.close()

# ==================== БОТ ====================
bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode="HTML"))
dp = Dispatcher(storage=MemoryStorage())

# ==================== СОСТОЯНИЯ ====================
class AdStates(StatesGroup):
    waiting_ad_text = State()
    waiting_pin_ad = State()

# ==================== КЛАВИАТУРЫ ====================
def main_menu():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🏷️ КУПИТЬ ТЕГ (50 ⭐)", callback_data="buy_tag")],
        [InlineKeyboardButton(text="📢 ЗАКАЗАТЬ РЕКЛАМУ", callback_data="ad_menu")],
        [InlineKeyboardButton(text="📌 ЗАКРЕПИТЬ РЕКЛАМУ (50 ⭐)", callback_data="pin_ad")],
        [InlineKeyboardButton(text="💎 ДОНАТ", callback_data="donate_menu")],
        [InlineKeyboardButton(text="🏆 ТОП ДОНАТЕРОВ", callback_data="top_donors")],
        [InlineKeyboardButton(text="ℹ️ ПОМОЩЬ", callback_data="help")],
    ])

def ad_tariffs_menu():
    kb = []
    for k, v in TARIFFS.items():
        kb.append([InlineKeyboardButton(text=f"{v['name']} — {v['price']} ⭐", callback_data=f"ad_pay|{k}")])
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    return InlineKeyboardMarkup(inline_keyboard=kb)

def donate_menu():
    kb = []
    for amount in DONATE_AMOUNTS:
        kb.append([InlineKeyboardButton(text=f"💎 {amount} ⭐", callback_data=f"donate|{amount}")])
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    return InlineKeyboardMarkup(inline_keyboard=kb)

# ==================== СТАРТ ====================
@dp.message(Command("start"))
async def start_cmd(message: Message):
    await message.answer(
        "🔥 <b>LOST SQUAD BOT</b> 🔥\n\n"
        "🏷️ Купи тег — 50 ⭐\n"
        "📢 Закажи рекламу — от 50 ⭐\n"
        "📌 Закрепи рекламу — 50 ⭐\n"
        "💎 Задонать в сквад — от 10 ⭐\n\n"
        "👇 Выбирай:",
        reply_markup=main_menu()
    )

# ==================== ПОМОЩЬ ====================
@dp.callback_query(F.data == "help")
async def help_cmd(c: CallbackQuery):
    await c.message.edit_text(
        "ℹ️ <b>ПОМОЩЬ ПО БОТУ</b>\n\n"
        "🏷️ <b>Тег:</b> платишь 50 ⭐ — получаешь тег в группе\n"
        "📢 <b>Реклама:</b> платишь → пишешь текст → админ проверяет → пост в группе на 12/24/48 часов\n"
        "📌 <b>Закреп:</b> только если уже есть реклама — 50 ⭐\n"
        "💎 <b>Донат:</b> поддержи сквад, попади в топ\n\n"
        "❓ Вопросы? Пиши админу.",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=[
            [InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")],
        ])
    )

# ==================== ТЕГ ====================
@dp.callback_query(F.data == "buy_tag")
async def buy_tag_start(c: CallbackQuery):
    await bot.send_invoice(
        chat_id=c.from_user.id,
        title="Покупка тега LOST SQUAD",
        description="Тег участника в группе LOST SQUAD",
        payload=f"tag_{c.from_user.id}",
        provider_token="",
        currency="XTR",
        prices=[LabeledPrice(label="Тег участника", amount=50)],
        start_parameter="buy_tag",
    )
    await c.answer("✅ Счёт выставлен!")

# ==================== РЕКЛАМА ====================
@dp.callback_query(F.data == "ad_menu")
async def ad_menu(c: CallbackQuery):
    await c.message.edit_text(
        "📢 <b>ЗАКАЗ РЕКЛАМЫ</b>\n\n"
        "Выбери тариф:\n"
        "📢 12 часов — 50 ⭐\n"
        "📢 24 часа — 100 ⭐\n"
        "📢 48 часов — 150 ⭐\n\n"
        "После оплаты отправь текст рекламы.",
        reply_markup=ad_tariffs_menu()
    )

@dp.callback_query(F.data.startswith("ad_pay|"))
async def ad_pay(c: CallbackQuery):
    tariff_key = c.data.split("|")[1]
    tariff = TARIFFS[tariff_key]

    await bot.send_invoice(
        chat_id=c.from_user.id,
        title=f"Реклама {tariff['hours']}ч",
        description=f"Реклама в группе LOST SQUAD на {tariff['hours']} часов",
        payload=f"ad_{c.from_user.id}_{tariff_key}",
        provider_token="",
        currency="XTR",
        prices=[LabeledPrice(label=f"Реклама {tariff['hours']}ч", amount=tariff["price"])],
        start_parameter="ad_order",
    )
    await c.answer("✅ Счёт выставлен!")

# ==================== ЗАКРЕП ====================
@dp.callback_query(F.data == "pin_ad")
async def pin_ad_start(c: CallbackQuery):
    # Проверяем есть ли активная реклама у юзера
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT ad_id, text FROM ads WHERE user_id = ? AND status = 'active' ORDER BY created_at DESC LIMIT 5",
              (c.from_user.id,))
    ads = c.fetchall()
    conn.close()

    if not ads:
        await c.answer("❌ У тебя нет активной рекламы! Сначала закажи рекламу.", show_alert=True)
        return

    kb = []
    for ad in ads:
        short_text = ad[1][:40] + "..." if len(ad[1]) > 40 else ad[1]
        kb.append([InlineKeyboardButton(text=f"#{ad[0]}: {short_text}", callback_data=f"pin_pay|{ad[0]}")])
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])

    await c.message.edit_text(
        "📌 <b>ЗАКРЕПИТЬ РЕКЛАМУ</b>\n\n"
        "Выбери какую рекламу закрепить:\n"
        "Стоимость: 50 ⭐",
        reply_markup=InlineKeyboardMarkup(inline_keyboard=kb)
    )

@dp.callback_query(F.data.startswith("pin_pay|"))
async def pin_pay(c: CallbackQuery):
    ad_id = int(c.data.split("|")[1])

    await bot.send_invoice(
        chat_id=c.from_user.id,
        title="Закреп рекламы",
        description=f"Закреп рекламы #{ad_id} в группе",
        payload=f"pin_{c.from_user.id}_{ad_id}",
        provider_token="",
        currency="XTR",
        prices=[LabeledPrice(label="Закреп рекламы", amount=50)],
        start_parameter="pin_ad",
    )
    await c.answer("✅ Счёт выставлен!")

# ==================== ДОНАТ ====================
@dp.callback_query(F.data == "donate_menu")
async def donate_menu_cb(c: CallbackQuery):
    await c.message.edit_text(
        "💎 <b>ДОНАТ В LOST SQUAD</b>\n\n"
        "Поддержи сквад и попади в топ донатеров!\n\n"
        "Выбери сумму:",
        reply_markup=donate_menu()
    )

@dp.callback_query(F.data.startswith("donate|"))
async def donate_pay(c: CallbackQuery):
    amount = int(c.data.split("|")[1])

    await bot.send_invoice(
        chat_id=c.from_user.id,
        title="Донат LOST SQUAD",
        description=f"Донат в размере {amount} ⭐",
        payload=f"don_{c.from_user.id}_{amount}",
        provider_token="",
        currency="XTR",
        prices=[LabeledPrice(label=f"Донат {amount} ⭐", amount=amount)],
        start_parameter="donate",
    )
    await c.answer("✅ Счёт выставлен!")

# ==================== ТОП ДОНАТЕРОВ ====================
@dp.callback_query(F.data == "top_donors")
async def top_donors(c: CallbackQuery):
    conn = sqlite3.connect("lost.db")
    cur = conn.cursor()
    cur.execute("SELECT username, SUM(amount) as total FROM donations GROUP BY user_id ORDER BY total DESC LIMIT 10")
    donors = cur.fetchall()
    conn.close()

    if not donors:
        txt = "🏆 <b>ТОП ДОНАТЕРОВ LOST SQUAD</b>\n\nПока никто не донатил. Будь первым! 💎"
    else:
        txt = "🏆 <b>ЛУЧШИЕ ДОНАТЕРЫ LOST SQUAD</b>\n\n"
        medals = ["🥇", "🥈", "🥉"] + ["🏅"] * 7
        for i, d in enumerate(donors):
            txt += f"{medals[i]} @{d[0]} — <b>{d[1]} ⭐</b>\n"

    await c.message.edit_text(txt, reply_markup=InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💎 ЗАДОНАТИТЬ", callback_data="donate_menu")],
        [InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")],
    ]))

# ==================== ОБРАБОТКА ПЛАТЕЖЕЙ ====================
@dp.pre_checkout_query()
async def pre_checkout(query: PreCheckoutQuery):
    await bot.answer_pre_checkout_query(query.id, ok=True)

@dp.message(F.successful_payment)
async def payment_success(message: Message, state: FSMContext):
    payload = message.successful_payment.invoice_payload
    parts = payload.split("_")
    ptype = parts[0]
    user_id = int(parts[1])

    # === ТЕГ ===
    if ptype == "tag":
        try:
            # Выдаём тег в группе
            tag_title = "LOST | Участник"
            await bot.promote_chat_member(
                GROUP_ID, user_id,
                can_change_info=False,
                can_post_messages=False,
                can_edit_messages=False,
                can_delete_messages=False,
                can_invite_users=False,
                can_restrict_members=False,
                can_pin_messages=False,
                can_manage_topics=False,
                is_anonymous=False
            )
            # Устанавливаем кастомный титул
            await bot.set_chat_administrator_custom_title(GROUP_ID, user_id, tag_title)

            conn = sqlite3.connect("lost.db")
            c = conn.cursor()
            c.execute("INSERT INTO tags (user_id, username, tag) VALUES (?, ?, ?)",
                      (user_id, message.from_user.username, tag_title))
            conn.commit()
            conn.close()

            await message.answer(f"✅ <b>ТЕГ ВЫДАН!</b>\n\nТвой тег в группе: <b>{tag_title}</b>", reply_markup=main_menu())
            await bot.send_message(GROUP_ID, f"🏷️ @{message.from_user.username} получил тег: <b>{tag_title}</b>")
        except Exception as e:
            await message.answer(f"❌ Ошибка выдачи тега. Бот должен быть админом группы с правами на изменение профилей.\n{e}")

    # === РЕКЛАМА ===
    elif ptype == "ad":
        tariff_key = parts[2]
        tariff = TARIFFS[tariff_key]
        await state.update_data(ad_hours=tariff["hours"], ad_price=tariff["price"], ad_pending=True)
        await message.answer(
            f"✅ <b>ОПЛАЧЕНО {tariff['price']} ⭐</b>\n\n"
            "📝 Теперь отправь текст рекламы.\n"
            "Он уйдёт на модерацию админу.",
            reply_markup=InlineKeyboardMarkup(inline_keyboard=[
                [InlineKeyboardButton(text="🔙 ОТМЕНА", callback_data="main_menu")],
            ])
        )
        await state.set_state(AdStates.waiting_ad_text)

    # === ЗАКРЕП ===
    elif ptype == "pin":
        ad_id = int(parts[2])

        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("SELECT message_id, status FROM ads WHERE ad_id = ? AND user_id = ?", (ad_id, user_id))
        ad = c.fetchone()

        if not ad:
            conn.close()
            await message.answer("❌ Реклама не найдена.", reply_markup=main_menu())
            return

        if ad[1] != "active":
            conn.close()
            await message.answer("❌ Реклама уже не активна.", reply_markup=main_menu())
            return

        try:
            await bot.pin_chat_message(GROUP_ID, ad[0], disable_notification=True)
            c.execute("INSERT INTO pins (user_id, username, ad_id, price, message_id) VALUES (?, ?, ?, ?, ?)",
                      (user_id, message.from_user.username, ad_id, 50, ad[0]))
            conn.commit()
            conn.close()
            await message.answer(f"✅ <b>РЕКЛАМА #{ad_id} ЗАКРЕПЛЕНА!</b>", reply_markup=main_menu())
        except Exception as e:
            conn.close()
            await message.answer(f"❌ Ошибка закрепления. Проверь права бота.\n{e}")

    # === ДОНАТ ===
    elif ptype == "don":
        amount = int(parts[2])
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("INSERT INTO donations (user_id, username, amount) VALUES (?, ?, ?)",
                  (user_id, message.from_user.username, amount))
        conn.commit()
        conn.close()

        await message.answer(
            f"💎 <b>СПАСИБО ЗА ДОНАТ!</b>\n\n"
            f"Ты задонатил <b>{amount} ⭐</b> в LOST SQUAD!\n"
            f"Ты попал в топ донатеров! 🏆",
            reply_markup=main_menu()
        )
        await bot.send_message(GROUP_ID, f"💎 @{message.from_user.username} задонатил <b>{amount} ⭐</b> в сквад! 🏆")

# ==================== ПОЛУЧЕНИЕ ТЕКСТА РЕКЛАМЫ ====================
@dp.message(AdStates.waiting_ad_text)
async def receive_ad_text(message: Message, state: FSMContext):
    text = message.text or message.caption or ""
    data = await state.get_data()
    hours = data.get("ad_hours", 12)
    price = data.get("ad_price", 50)

    if len(text) < 5:
        await message.answer("❌ Слишком короткий текст. Минимум 5 символов.")
        return

    # Сохраняем в базу как pending
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("INSERT INTO ads (user_id, username, text, duration_hours, price, status) VALUES (?, ?, ?, ?, ?, 'pending')",
              (message.from_user.id, message.from_user.username, text, hours, price))
    ad_id = c.lastrowid
    conn.commit()
    conn.close()

    # Отправляем админу на модерацию
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="✅ ОДОБРИТЬ", callback_data=f"approve|{ad_id}")],
        [InlineKeyboardButton(text="❌ ОТКЛОНИТЬ", callback_data=f"decline|{ad_id}")],
    ])

    await bot.send_message(ADMIN_ID,
        f"📢 <b>НОВАЯ РЕКЛАМА НА МОДЕРАЦИИ</b>\n\n"
        f"👤 @{message.from_user.username}\n"
        f"⏱ {hours} часов | 💰 {price} ⭐\n\n"
        f"📝 <b>Текст:</b>\n{text}\n\n"
        f"👇 Выбери действие:",
        reply_markup=kb
    )

    await message.answer(
        f"✅ <b>РЕКЛАМА #{ad_id} ОТПРАВЛЕНА НА МОДЕРАЦИЮ</b>\n\n"
        "Админ проверит её в ближайшее время.\n"
        "Если отклонят — сможешь исправить без повторной оплаты.",
        reply_markup=main_menu()
    )
    await state.clear()

# ==================== МОДЕРАЦИЯ ====================
@dp.callback_query(F.data.startswith("approve|"))
async def approve_ad(c: CallbackQuery):
    if c.from_user.id != ADMIN_ID:
        await c.answer("❌ Только админ!")
        return

    ad_id = int(c.data.split("|")[1])

    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT * FROM ads WHERE ad_id = ?", (ad_id,))
    ad = c.fetchone()

    if not ad or ad[8] != "pending":
        conn.close()
        await c.answer("Реклама уже обработана.")
        return

    # Отправляем в группу
    try:
        msg = await bot.send_message(GROUP_ID,
            f"📢 <b>РЕКЛАМА</b>\n"
            f"👤 @{ad[2]}\n"
            f"⏱ {ad[4]} часов\n\n"
            f"{ad[3]}"
        )

        expire_time = datetime.now() + timedelta(hours=ad[4])
        c.execute("UPDATE ads SET status = 'active', message_id = ?, expire_time = ? WHERE ad_id = ?",
                  (msg.message_id, expire_time, ad_id))
        conn.commit()
        conn.close()

        await c.message.edit_text(
            f"✅ <b>РЕКЛАМА #{ad_id} ОДОБРЕНА!</b>\n"
            f"Пост отправлен в группу.\n"
            f"Удалится через {ad[4]} часов."
        )

        # Уведомляем юзера
        try:
            await bot.send_message(ad[1], f"✅ <b>РЕКЛАМА #{ad_id} ОДОБРЕНА!</b>\nПост уже в группе на {ad[4]} часов.")
        except:
            pass

        # Автоудаление
        asyncio.create_task(delete_ad_later(msg.message_id, ad[4], ad_id))

    except Exception as e:
        await c.answer(f"Ошибка: {e}")

@dp.callback_query(F.data.startswith("decline|"))
async def decline_ad(c: CallbackQuery, state: FSMContext):
    if c.from_user.id != ADMIN_ID:
        await c.answer("❌ Только админ!")
        return

    ad_id = int(c.data.split("|")[1])
    await state.update_data(decline_ad_id=ad_id)
    await c.message.edit_text(f"❌ Отклоняешь рекламу #{ad_id}. Напиши причину:")
    await state.set_state(AdStates.waiting_pin_ad)

@dp.message(AdStates.waiting_pin_ad)
async def decline_reason(message: Message, state: FSMContext):
    data = await state.get_data()
    ad_id = data.get("decline_ad_id")

    reason = message.text or "Без причины"

    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("UPDATE ads SET status = 'declined', decline_reason = ? WHERE ad_id = ?", (reason, ad_id))
    c.execute("SELECT user_id FROM ads WHERE ad_id = ?", (ad_id,))
    user = c.fetchone()
    conn.commit()
    conn.close()

    await message.answer(f"✅ Реклама #{ad_id} отклонена с причиной: {reason}")

    if user:
        try:
            await bot.send_message(user[0],
                f"❌ <b>РЕКЛАМА #{ad_id} ОТКЛОНЕНА</b>\n\n"
                f"Причина: {reason}\n\n"
                f"✏️ Можешь исправить и отправить снова. Нажми /start и выбери «Заказать рекламу».\n"
                f"Повторная оплата не нужна — нажми «У меня есть оплаченная реклама»."
            )
        except:
            pass

    await state.clear()

# ==================== ПОВТОРНАЯ ОТПРАВКА ПОСЛЕ ОТКЛОНЕНИЯ ====================
@dp.callback_query(F.data == "retry_ad")
async def retry_ad(c: CallbackQuery, state: FSMContext):
    conn = sqlite3.connect("lost.db")
    c = conn.cursor()
    c.execute("SELECT ad_id, duration_hours, price FROM ads WHERE user_id = ? AND status = 'declined' ORDER BY created_at DESC LIMIT 1",
              (c.from_user.id,))
    ad = c.fetchone()
    conn.close()

    if not ad:
        await c.answer("❌ Нет отклонённых реклам.")
        return

    await state.update_data(ad_hours=ad[1], ad_price=ad[2], ad_pending=True, retry_ad_id=ad[0])
    await c.message.edit_text("📝 Отправь исправленный текст рекламы:")
    await state.set_state(AdStates.waiting_ad_text)

# ==================== АВТОУДАЛЕНИЕ ====================
async def delete_ad_later(message_id, hours, ad_id):
    await asyncio.sleep(hours * 3600)
    try:
        await bot.delete_message(GROUP_ID, message_id)
        conn = sqlite3.connect("lost.db")
        c = conn.cursor()
        c.execute("UPDATE ads SET status = 'deleted' WHERE ad_id = ?", (ad_id,))
        conn.commit()
        conn.close()
    except:
        pass

# ==================== ГЛАВНОЕ МЕНЮ ====================
@dp.callback_query(F.data == "main_menu")
async def main_menu_cb(c: CallbackQuery, state: FSMContext):
    await state.clear()
    await c.message.edit_text(
        "🔥 <b>LOST SQUAD BOT</b> 🔥\n\n"
        "🏷️ Купи тег — 50 ⭐\n"
        "📢 Закажи рекламу — от 50 ⭐\n"
        "📌 Закрепи рекламу — 50 ⭐\n"
        "💎 Задонать в сквад — от 10 ⭐\n\n"
        "👇 Выбирай:",
        reply_markup=main_menu()
    )

# ==================== ЗАПУСК ====================
async def main():
    init_db()
    print("🔥 LOST SQUAD BOT ЗАПУЩЕН!")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
