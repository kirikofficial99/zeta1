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
import sqlite3
import random
import string
from datetime import datetime

# ==================== КОНФИГ ====================
TOKEN = "8912412788:AAH-x35pYYW9gRUN9Pblhf6DtfvqSHKJiUY"
ADMIN_ID = 7832555448
CURATOR_USERNAME = "atlanov_bot"  # Юзернейм уже существующего куратора

# ==================== БАЗА ДАННЫХ ====================
def init_db():
    conn = sqlite3.connect("zeta.db")
    c = conn.cursor()
    
    c.execute("""CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        first_name TEXT,
        balance_stars INTEGER DEFAULT 0,
        total_spent INTEGER DEFAULT 0,
        orders_count INTEGER DEFAULT 0,
        referrer_id INTEGER DEFAULT NULL,
        referral_bonus_claimed INTEGER DEFAULT 0,
        referral_code TEXT UNIQUE,
        joined_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")
    
    c.execute("""CREATE TABLE IF NOT EXISTS orders (
        order_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        username TEXT,
        item TEXT,
        amount_stars INTEGER,
        status TEXT DEFAULT 'completed',
        city TEXT,
        district TEXT,
        address TEXT,
        description TEXT,
        promo_used TEXT DEFAULT NULL,
        timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")
    
    c.execute("""CREATE TABLE IF NOT EXISTS promo_codes (
        code TEXT PRIMARY KEY,
        discount INTEGER,
        uses_left INTEGER,
        created_by INTEGER,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )""")
    
    conn.commit()
    conn.close()

def get_user(user_id):
    conn = sqlite3.connect("zeta.db")
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    user = c.fetchone()
    conn.close()
    return user

def create_user(user_id, username, first_name, referrer_id=None):
    conn = sqlite3.connect("zeta.db")
    c = conn.cursor()
    ref_code = 'Z' + ''.join(random.choices(string.ascii_uppercase + string.digits, k=6))
    c.execute("""INSERT OR IGNORE INTO users (user_id, username, first_name, referral_code) 
                 VALUES (?, ?, ?, ?)""", (user_id, username, first_name, ref_code))
    if referrer_id and str(referrer_id) != str(user_id):
        c.execute("""UPDATE users SET referrer_id = ? 
                     WHERE user_id = ? AND referrer_id IS NULL""", (referrer_id, user_id))
    conn.commit()
    conn.close()
    return ref_code

# ==================== ТОВАРЫ И ГОРОДА ====================
CATALOG = {
    "zeta_haze_05": {"name": "🥬 Zeta Haze 0.5г", "price": 500, "desc": "Сортовая Zeta-конопля. Вкус — цитрус с дизелем. Улёт в космос."},
    "zeta_haze_1":  {"name": "🥬 Zeta Haze 1г",   "price": 1000, "desc": "Zeta Haze — вдвое больше удовольствия."},
    "purple_z_05":  {"name": "🍇 Purple Zeta 0.5г","price": 600, "desc": "Индичный сорт. Виноградный аромат. Глубокий релакс."},
    "purple_z_1":   {"name": "🍇 Purple Zeta 1г",  "price": 1100, "desc": "Purple Zeta — полный релакс на весь вечер."},
    "zeta_ice_05":  {"name": "❄️ Zeta Ice 0.5г",   "price": 700, "desc": "Чистейший ледяной стафф. Мгновенный эффект."},
    "zeta_ice_1":   {"name": "❄️ Zeta Ice 1г",     "price": 1300, "desc": "Ледяной шторм. Только для опытных."},
    "xtc_1":        {"name": "💊 Zeta XTC 1шт",    "price": 400, "desc": "Экстази с логотипом Z. Чистый MDMA."},
    "xtc_5":        {"name": "💊 Zeta XTC 5шт",    "price": 1800, "desc": "5 таблеток. На всю ночь и утро."},
    "shrooms_1":    {"name": "🍄 Zeta Shrooms 1г",  "price": 800, "desc": "Псилоцибиновые грибы. Трип 6-8 часов."},
    "lsd_1":        {"name": "🧪 Zeta LSD 1 марка", "price": 900, "desc": "Лизергиновый трип 12 часов. Не для новичков."},
}

CITIES = {
    "msk": {"name": "Москва", "districts": ["ЦАО", "САО", "ЮАО", "ВАО", "ЗАО", "СВАО"]},
    "spb": {"name": "Санкт-Петербург", "districts": ["Центр", "Север", "Юг", "Васька", "Петроградка"]},
    "kzn": {"name": "Казань", "districts": ["Центр", "Авиастрой", "Горки", "Дербышки"]},
    "ekb": {"name": "Екатеринбург", "districts": ["Центр", "Уралмаш", "ВИЗ", "ЖБИ"]},
    "nsk": {"name": "Новосибирск", "districts": ["Центр", "Левый берег", "Правый берег"]},
}

STREETS = [
    "ул. Ленина", "пр. Мира", "ул. Гагарина", "ул. Пушкина", 
    "пр. Победы", "ул. Советская", "ул. Центральная", "ул. Лесная",
    "пр. Космонавтов", "ул. Молодёжная", "ул. Садовая", "пр. Строителей"
]

HIDE_DESCS = [
    "🧱 За домом, под кустом у забора, магнитный пакет с логотипом Z.",
    "🚗 Под колесом припаркованной синей машины у 3-го подъезда.",
    "🌳 В дупле дерева справа от главного входа в парк, на высоте 1.5м.",
    "🪨 Под большим серым камнем у трансформаторной будки.",
    "🔩 За почтовыми ящиками на 1 этаже, приклеен скотчем к стене.",
    "🗑️ Под мусорным контейнером у дальнего угла площадки.",
    "🔌 В электрощитке на лестничной клетке 2 этажа.",
    "🚪 За дверью пожарного выхода, прикрыт кирпичом.",
]

# ==================== БОТ ====================
bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode="HTML"))
dp = Dispatcher(storage=MemoryStorage())

# ==================== КЛАВИАТУРЫ ====================
def main_menu():
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🛒 КАТАЛОГ", callback_data="catalog")],
        [InlineKeyboardButton(text="💰 БАЛАНС", callback_data="balance_menu")],
        [InlineKeyboardButton(text="📦 МОИ ЗАКАЗЫ", callback_data="orders")],
        [InlineKeyboardButton(text="👥 РЕФЕРАЛЫ", callback_data="referral")],
        [InlineKeyboardButton(text="🎟️ ПРОМОКОД", callback_data="promo_menu")],
        [InlineKeyboardButton(text="⭐ ОТЗЫВЫ", callback_data="reviews")],
        [InlineKeyboardButton(text="📞 ПОДДЕРЖКА", callback_data="support")],
    ])

def back_btn(callback="main_menu"):
    return InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔙 НАЗАД", callback_data=callback)]
    ])

# ==================== СТАРТ ====================
@dp.message(Command("start"))
async def start_cmd(message: Message):
    args = message.text.split()
    referrer_id = None
    
    if len(args) > 1:
        if args[1].startswith("ref_"):
            try:
                referrer_id = int(args[1].replace("ref_", ""))
            except:
                pass
    
    user = get_user(message.from_user.id)
    if not user:
        create_user(message.from_user.id, message.from_user.username, message.from_user.first_name, referrer_id)
    
    await message.answer(
        "🚀 <b>ZETA SHOP</b> — анонимный магазин премиум-стаффа.\n\n"
        "🔥 Работаем по всей России. Мгновенные клады.\n"
        "💎 Только оригинальная продукция.\n"
        "💳 Оплата: Telegram Stars (анонимно).\n\n"
        "<b>👥 Акция:</b> Пригласи друга — получи <b>+100 ⭐</b> на баланс!\n\n"
        "👇 Выбирай:",
        reply_markup=main_menu()
    )

# ==================== КАТАЛОГ ====================
@dp.callback_query(F.data == "catalog")
async def catalog(c: CallbackQuery):
    kb = []
    for k, v in CATALOG.items():
        kb.append([InlineKeyboardButton(text=f"{v['name']} — {v['price']} ⭐", callback_data=f"item_{k}")])
    kb.append([InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")])
    await c.message.edit_text("🛒 <b>КАТАЛОГ ZETA SHOP</b>\n\nВыбери товар:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("item_"))
async def show_item(c: CallbackQuery):
    item_id = c.data.replace("item_", "")
    item = CATALOG[item_id]
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text=f"💳 КУПИТЬ — {item['price']} ⭐", callback_data=f"buy_{item_id}")],
        [InlineKeyboardButton(text="🔙 К КАТАЛОГУ", callback_data="catalog")],
    ])
    
    await c.message.edit_text(
        f"<b>{item['name']}</b>\n\n"
        f"📝 {item['desc']}\n\n"
        f"💰 Цена: <b>{item['price']} ⭐</b>\n\n"
        f"👇 Выбери действие:",
        reply_markup=kb
    )

# ==================== БАЛАНС ====================
@dp.callback_query(F.data == "balance_menu")
async def balance_menu(c: CallbackQuery):
    user = get_user(c.from_user.id)
    balance = user[2] if user else 0
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💎 +500 ⭐", callback_data="topup_500")],
        [InlineKeyboardButton(text="💎 +1000 ⭐", callback_data="topup_1000")],
        [InlineKeyboardButton(text="💎 +3000 ⭐", callback_data="topup_3000")],
        [InlineKeyboardButton(text="💎 +5000 ⭐", callback_data="topup_5000")],
        [InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")],
    ])
    
    await c.message.edit_text(
        f"💰 <b>ТВОЙ БАЛАНС:</b> <code>{balance} ⭐</code>\n\n"
        f"📈 Пополни баланс для заказа:\n"
        f"• 500 ⭐ = 0.5г\n"
        f"• 1000 ⭐ = 1г\n"
        f"• Больше ⭐ = выгоднее курс!\n\n"
        f"👇 Выбери сумму:",
        reply_markup=kb
    )

@dp.callback_query(F.data.startswith("topup_"))
async def topup(c: CallbackQuery):
    amount = int(c.data.replace("topup_", ""))
    
    await bot.send_invoice(
        chat_id=c.from_user.id,
        title="Пополнение Zeta Shop",
        description=f"Зачисление {amount} Stars на баланс Zeta Shop",
        payload=f"topup_{c.from_user.id}_{amount}",
        provider_token="",
        currency="XTR",
        prices=[LabeledPrice(label=f"Пополнение {amount} ⭐", amount=amount)],
        start_parameter="zeta_topup",
    )
    await c.answer("✅ Счёт выставлен! Проверь чат с ботом.")

@dp.pre_checkout_query()
async def pre_checkout(query: PreCheckoutQuery):
    await bot.answer_pre_checkout_query(query.id, ok=True)

@dp.message(F.successful_payment)
async def payment_success(message: Message):
    payload = message.successful_payment.invoice_payload
    _, uid, amount = payload.split("_")
    user_id = int(uid)
    amount = int(amount)
    
    conn = sqlite3.connect("zeta.db")
    c = conn.cursor()
    c.execute("UPDATE users SET balance_stars = balance_stars + ? WHERE user_id = ?", (amount, user_id))
    c.execute("INSERT OR IGNORE INTO users (user_id, username, first_name) VALUES (?, ?, ?)", 
              (user_id, message.from_user.username, message.from_user.first_name))
    
    # Реферальный бонус
    c.execute("SELECT referrer_id, referral_bonus_claimed FROM users WHERE user_id = ?", (user_id,))
    ref_data = c.fetchone()
    if ref_data and ref_data[0] and ref_data[1] == 0 and amount >= 500:
        c.execute("UPDATE users SET balance_stars = balance_stars + 100 WHERE user_id = ?", (ref_data[0],))
        c.execute("UPDATE users SET referral_bonus_claimed = 1 WHERE user_id = ?", (user_id,))
        try:
            await bot.send_message(ref_data[0], "🎉 <b>РЕФЕРАЛЬНЫЙ БОНУС +100 ⭐!</b>\nТвой друг пополнил баланс от 500 ⭐ — ты получил 100 ⭐.")
        except:
            pass
        await bot.send_message(ADMIN_ID, f"👥 Реф.бонус: +100 ⭐ юзеру {ref_data[0]} от {user_id}")
    
    conn.commit()
    conn.close()
    
    await message.answer(f"✅ <b>БАЛАНС ПОПОЛНЕН НА {amount} ⭐!</b>\n\nТеперь можешь заказывать. Выбирай товар в каталоге.", reply_markup=main_menu())
    await bot.send_message(ADMIN_ID, f"💰 <b>+{amount} ⭐</b>\nОт: @{message.from_user.username}\nID: <code>{user_id}</code>")

# ==================== ПОКУПКА ====================
@dp.callback_query(F.data.startswith("buy_"))
async def buy_start(c: CallbackQuery):
    item_id = c.data.replace("buy_", "")
    item = CATALOG[item_id]
    user = get_user(c.from_user.id)
    
    if not user or user[2] < item["price"]:
        await c.answer(f"❌ Не хватает звёзд! Нужно: {item['price']} ⭐. Пополни баланс.", show_alert=True)
        return
    
    kb = []
    for ck, cv in CITIES.items():
        kb.append([InlineKeyboardButton(text=f"📍 {cv['name']}", callback_data=f"city_{item_id}_{ck}")])
    kb.append([InlineKeyboardButton(text="🔙 К товару", callback_data=f"item_{item_id}")])
    
    await c.message.edit_text(f"📍 Выбери город для <b>{item['name']}</b>:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("city_"))
async def choose_district(c: CallbackQuery):
    _, item_id, city_key = c.data.split("_", 2)
    
    if city_key not in CITIES:
        await c.answer("Ошибка, попробуй снова.")
        return
    
    city = CITIES[city_key]
    kb = []
    for d in city["districts"]:
        kb.append([InlineKeyboardButton(text=f"🏘️ {d}", callback_data=f"dist_{item_id}_{city_key}_{d}")])
    kb.append([InlineKeyboardButton(text="🔙 К выбору города", callback_data=f"buy_{item_id}")])
    
    await c.message.edit_text(f"🏙️ <b>{city['name']}</b>\nВыбери район:", reply_markup=InlineKeyboardMarkup(inline_keyboard=kb))

@dp.callback_query(F.data.startswith("dist_"))
async def confirm_order(c: CallbackQuery):
    _, item_id, city_key, district = c.data.split("_", 3)
    item = CATALOG[item_id]
    city = CITIES[city_key]
    user = get_user(c.from_user.id)
    
    if not user or user[2] < item["price"]:
        await c.answer(f"❌ Недостаточно звёзд! Нужно: {item['price']} ⭐", show_alert=True)
        return
    
    # Списываем звёзды
    conn = sqlite3.connect("zeta.db")
    cur = conn.cursor()
    cur.execute("UPDATE users SET balance_stars = balance_stars - ?, total_spent = total_spent + ?, orders_count = orders_count + 1 WHERE user_id = ?",
                (item["price"], item["price"], c.from_user.id))
    
    # Генерируем фейковый адрес
    street = random.choice(STREETS)
    house = random.randint(1, 100)
    building = random.choice(["", f"к{random.randint(1,5)}", f"стр{random.randint(1,3)}"])
    full_addr = f"г. {city['name']}, {street}, д. {house}{' ' + building if building else ''}"
    hide_desc = random.choice(HIDE_DESCS)
    
    # Добавляем случайную задержку "закладки" от 5 до 25 минут
    delay_min = random.randint(5, 25)
    ready_time = datetime.now() + timedelta(minutes=delay_min)
    ready_time_str = ready_time.strftime("%H:%M")
    
    cur.execute("""INSERT INTO orders (user_id, username, item, amount_stars, city, district, address, description, status) 
                   VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'completed')""",
                (c.from_user.id, c.from_user.username, item["name"], item["price"], city["name"], district, full_addr, hide_desc))
    order_id = cur.lastrowid
    
    # Система лояльности: +200 ⭐ за каждые 5000 потраченных
    total_spent = (user[3] or 0) + item["price"]
    old_loyalty = (user[3] or 0) // 5000
    new_loyalty = total_spent // 5000
    loyalty_bonus = 0
    if new_loyalty > old_loyalty:
        loyalty_bonus = (new_loyalty - old_loyalty) * 200
        cur.execute("UPDATE users SET balance_stars = balance_stars + ? WHERE user_id = ?", (loyalty_bonus, c.from_user.id))
    
    conn.commit()
    conn.close()
    
    txt = (
        f"✅ <b>ЗАКАЗ #{order_id} ОФОРМЛЕН!</b>\n\n"
        f"📦 Товар: <b>{item['name']}</b>\n"
        f"💰 Сумма: <b>{item['price']} ⭐</b>\n"
        f"📍 Город: <b>{city['name']}</b>\n"
        f"🏘️ Район: <b>{district}</b>\n\n"
        f"⏳ <b>Клад будет заложен через {delay_min} мин.</b>\n"
        f"Ожидай до <b>{ready_time_str}</b>, затем адрес появится.\n\n"
        f"📌 Адрес: <code>{full_addr}</code>\n"
        f"🔍 Описание: {hide_desc}\n\n"
        f"⚠️ <b>ЗАБЕРИ В ТЕЧЕНИЕ 2 ЧАСОВ!</b>\n"
        f"После адрес сгорит.\n\n"
        f"📞 По любым вопросам пиши куратору:\n"
        f"👉 <b>@{CURATOR_USERNAME}</b>\n"
        f"Укажи номер заказа: <code>#{order_id}</code>"
    )
    
    if loyalty_bonus > 0:
        txt += f"\n\n🎖️ <b>БОНУС ЛОЯЛЬНОСТИ!</b>\nТы потратил более {new_loyalty * 5000} ⭐ за всё время.\n+{loyalty_bonus} ⭐ начислено на баланс!"
    
    await c.message.edit_text(txt, reply_markup=main_menu())
    
    # Уведомление тебе
    await bot.send_message(ADMIN_ID, 
        f"🔥 <b>НОВЫЙ ЗАКАЗ #{order_id}</b>\n"
        f"👤 @{c.from_user.username} (ID: {c.from_user.id})\n"
        f"📦 {item['name']}\n"
        f"💰 {item['price']} ⭐\n"
        f"📍 {city['name']}, {district}\n"
        f"🏠 {full_addr}\n"
        f"⏳ Закладка через {delay_min} мин."
    )

# ==================== МОИ ЗАКАЗЫ ====================
@dp.callback_query(F.data == "orders")
async def my_orders(c: CallbackQuery):
    conn = sqlite3.connect("zeta.db")
    cur = conn.cursor()
    cur.execute("SELECT order_id, item, amount_stars, city, district, timestamp FROM orders WHERE user_id = ? ORDER BY timestamp DESC LIMIT 10", 
                (c.from_user.id,))
    orders = cur.fetchall()
    conn.close()
    
    if not orders:
        txt = "📦 У тебя пока нет заказов.\n\nПерейди в каталог и сделай первый заказ!"
    else:
        txt = "📦 <b>ТВОИ ЗАКАЗЫ:</b>\n\n"
        for o in orders:
            txt += f"<code>#{o[0]}</code> | {o[1]} | {o[2]} ⭐ | {o[3]}, {o[4]} | {o[5]}\n"
        txt += f"\n📞 Вопросы? Пиши куратору: <b>@{CURATOR_USERNAME}</b>"
    
    await c.message.edit_text(txt, reply_markup=main_menu())

# ==================== РЕФЕРАЛЫ ====================
@dp.callback_query(F.data == "referral")
async def referral(c: CallbackQuery):
    bot_info = await bot.me()
    ref_link = f"https://t.me/{bot_info.username}?start=ref_{c.from_user.id}"
    
    conn = sqlite3.connect("zeta.db")
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*) FROM users WHERE referrer_id = ?", (c.from_user.id,))
    ref_count = cur.fetchone()[0]
    cur.execute("SELECT COUNT(*) FROM users WHERE referrer_id = ? AND referral_bonus_claimed = 1", (c.from_user.id,))
    bonus_count = cur.fetchone()[0]
    conn.close()
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📋 КОПИРОВАТЬ ССЫЛКУ", callback_data="copy_ref")],
        [InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")],
    ])
    
    await c.message.edit_text(
        f"👥 <b>РЕФЕРАЛЬНАЯ СИСТЕМА</b>\n\n"
        f"🔥 Приведи друга — получи <b>+100 ⭐</b>\n"
        f"Условие: друг пополняет баланс от 500 ⭐\n"
        f"Бонус начисляется автоматически!\n\n"
        f"🔗 Твоя ссылка:\n<code>{ref_link}</code>\n\n"
        f"📊 Статистика:\n"
        f"👤 Приглашено: <b>{ref_count}</b>\n"
        f"💰 Бонусов получено: <b>{bonus_count * 100} ⭐</b>",
        reply_markup=kb
    )

@dp.callback_query(F.data == "copy_ref")
async def copy_ref(c: CallbackQuery):
    bot_info = await bot.me()
    ref_link = f"https://t.me/{bot_info.username}?start=ref_{c.from_user.id}"
    await c.message.answer(f"🔗 Твоя реферальная ссылка:\n<code>{ref_link}</code>\n\nОтправь другу или размести в чате!")
    await c.answer("✅ Ссылка отправлена в чат!")

# ==================== ПРОМОКОДЫ ====================
@dp.callback_query(F.data == "promo_menu")
async def promo_menu(c: CallbackQuery):
    user = get_user(c.from_user.id)
    total_spent = user[3] if user else 0
    
    kb = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🎟️ СОЗДАТЬ ПРОМОКОД (скидка 15%)", callback_data="create_promo")],
        [InlineKeyboardButton(text="🔙 НАЗАД", callback_data="main_menu")],
    ])
    
    await c.message.edit_text(
        f"🎟️ <b>ПРОМОКОДЫ</b>\n\n"
        f"Создай персональный промокод на скидку 15%\n"
        f"Можно использовать 5 раз.\n"
        f"Минимальный общий расход: 1000 ⭐\n\n"
        f"💰 Твой общий расход: <b>{total_spent} ⭐</b>\n\n"
        f"👇 Выбери действие:",
        reply_markup=kb
    )

@dp.callback_query(F.data == "create_promo")
async def create_promo(c: CallbackQuery):
    user = get_user(c.from_user.id)
    total_spent = user[3] if user else 0
    
    if total_spent < 1000:
        await c.answer("❌ Недостаточно! Нужно потратить минимум 1000 ⭐ для создания промокода.", show_alert=True)
        return
    
    code = f"ZETA{random.randint(1000,9999)}"
    conn = sqlite3.connect("zeta.db")
    cur = conn.cursor()
    cur.execute("INSERT OR REPLACE INTO promo_codes (code, discount, uses_left, created_by) VALUES (?, 15, 5, ?)", (code, c.from_user.id))
    conn.commit()
    conn.close()
    
    await c.message.edit_text(
        f"🎟️ <b>ПРОМОКОД СОЗДАН!</b>\n\n"
        f"Код: <code>{code}</code>\n"
        f"Скидка: 15%\n"
        f"Использований осталось: 5\n\n"
        f"📝 Как использовать:\n"
        f"При оформлении заказа отправь этот код в чат с ботом.\n"
        f"Цена автоматически пересчитается со скидкой.",
        reply_markup=main_menu()
    )

# Обработчик текста (для промокодов)
@dp.message(F.text)
async def handle_text(message: Message):
    text = message.text.strip().upper()
    
    # Проверяем промокод
    conn = sqlite3.connect("zeta.db")
    c = conn.cursor()
    c.execute("SELECT * FROM promo_codes WHERE code = ? AND uses_left > 0", (text,))
    promo = c.fetchone()
    
    if promo:
        c.execute("UPDATE promo_codes SET uses_left = uses_left - 1 WHERE code = ?", (text,))
        conn.commit()
        conn.close()
        
        await message.answer(
            f"🎟️ <b>ПРОМОКОД АКТИВИРОВАН!</b>\n\n"
            f"Код: <code>{text}</code>\n"
            f"Скидка: {promo[1]}%\n"
            f"Осталось использований: {promo[2] - 1}\n\n"
            f"Теперь при оформлении заказа введи этот код ещё раз, и цена снизится на {promo[1]}%.",
            reply_markup=main_menu()
        )
        return
    else:
        conn.close()
    
    # Если не промокод — просто показываем меню
    await message.answer("Используй кнопки меню для навигации. 👇", reply_markup=main_menu())

# ==================== ОТЗЫВЫ ====================
@dp.callback_query(F.data == "reviews")
async def reviews(c: CallbackQuery):
    fakes = [
        ("⭐⭐⭐⭐⭐", "Аноним", "Всё чётко! Забрал за 10 минут. Качество — пушка! 🔥"),
        ("⭐⭐⭐⭐⭐", "DarkStar", "Третий заказ, всё на высоте. Лучший шоп в телеге!"),
        ("⭐⭐⭐⭐", "Psychonaut", "Немного задержали адрес, но стафф реально топ."),
        ("⭐⭐⭐⭐⭐", "IceKing", "Zeta Ice — это нечто! Эффект просто ураган!"),
        ("⭐⭐⭐⭐⭐", "MollyGirl", "XTC — чистейший кайф! Спасибо Zeta! 💊"),
        ("⭐⭐⭐⭐", "GreenMan", "Брал Purple Zeta — очень вкусный и мягкий сорт."),
        ("⭐⭐⭐⭐⭐", "TripMaster", "Грибы — полный космос! Доставили быстро и чётко."),
    ]
    
    txt = "⭐ <b>ОТЗЫВЫ НАШИХ КЛИЕНТОВ</b>\n\n"
    for stars, name, text in fakes:
        txt += f"{stars} <b>{name}</b>\n<i>{text}</i>\n\n"
    
    txt += f"📞 Есть вопросы? Пиши куратору: <b>@{CURATOR_USERNAME}</b>"
    
    await c.message.edit_text(txt, reply_markup=main_menu())

# ==================== ПОДДЕРЖКА ====================
@dp.callback_query(F.data == "support")
async def support(c: CallbackQuery):
    await c.message.edit_text(
        "📞 <b>ПОДДЕРЖКА ZETA SHOP</b>\n\n"
        f"По всем вопросам пиши нашему куратору:\n"
        f"👉 <b>@{CURATOR_USERNAME}</b>\n\n"
        "⚠️ <i>Не сообщай никому личные данные.\n"
        "Мы никогда не просим пароли или коды подтверждения.</i>\n\n"
        "🕐 Время ответа: до 30 минут.",
        reply_markup=main_menu()
    )

# ==================== ГЛАВНОЕ МЕНЮ ====================
@dp.callback_query(F.data == "main_menu")
async def main_menu_cb(c: CallbackQuery):
    await c.message.edit_text(
        "🚀 <b>ZETA SHOP</b> — главное меню\n\n"
        "💎 Премиум-стафф с доставкой по РФ\n"
        "💳 Анонимная оплата Telegram Stars\n"
        "⚡ Мгновенные клады в твоём городе\n\n"
        "👇 Выбирай действие:",
        reply_markup=main_menu()
    )

# ==================== АДМИНКА ====================
@dp.message(Command("admin"))
async def admin_cmd(message: Message):
    if message.from_user.id != ADMIN_ID:
        await message.answer("🚫 Доступ запрещён.")
        return
    
    args = message.text.split()
    
    if len(args) == 1:
        txt = (
            "🔐 <b>АДМИН-ПАНЕЛЬ ZETA SHOP</b>\n\n"
            "<b>Команды:</b>\n"
            "/admin stats — общая статистика\n"
            "/admin orders — последние 10 заказов\n"
            "/admin order 123 — детали заказа\n"
            "/admin users — топ-10 юзеров\n"
            "/admin user 123456 — инфо о юзере\n"
            "/admin promo — все промокоды\n"
            "/admin broadcast текст — рассылка всем\n"
            "/admin addbalance 123456 500 — начислить ⭐"
        )
        await message.answer(txt)
    
    elif args[1] == "stats":
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT COUNT(*) FROM users")
        total_users = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM orders")
        total_orders = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(amount_stars), 0) FROM orders")
        total_revenue = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(balance_stars), 0) FROM users")
        total_balance = c.fetchone()[0]
        c.execute("SELECT COUNT(*) FROM users WHERE referrer_id IS NOT NULL")
        ref_users = c.fetchone()[0]
        
        # За сегодня
        c.execute("SELECT COUNT(*) FROM orders WHERE date(timestamp) = date('now')")
        orders_today = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(amount_stars), 0) FROM orders WHERE date(timestamp) = date('now')")
        revenue_today = c.fetchone()[0]
        conn.close()
        
        await message.answer(
            f"📊 <b>СТАТИСТИКА ZETA SHOP</b>\n\n"
            f"👤 Всего юзеров: <b>{total_users}</b>\n"
            f"👥 Пришли по рефке: <b>{ref_users}</b>\n"
            f"📦 Всего заказов: <b>{total_orders}</b>\n"
            f"💰 Общая выручка: <b>{total_revenue} ⭐</b>\n"
            f"💎 Баланс у юзеров: <b>{total_balance} ⭐</b>\n\n"
            f"📅 <b>ЗА СЕГОДНЯ:</b>\n"
            f"📦 Заказов: <b>{orders_today}</b>\n"
            f"💰 Выручка: <b>{revenue_today} ⭐</b>"
        )
    
    elif args[1] == "orders":
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT order_id, username, item, amount_stars, city, timestamp FROM orders ORDER BY timestamp DESC LIMIT 10")
        orders = c.fetchall()
        conn.close()
        
        if not orders:
            await message.answer("📦 Заказов пока нет.")
            return
        
        txt = "📦 <b>ПОСЛЕДНИЕ 10 ЗАКАЗОВ:</b>\n\n"
        for o in orders:
            txt += f"<code>#{o[0]}</code> | @{o[1]} | {o[2]} | {o[3]} ⭐ | {o[4]} | {o[5]}\n"
        
        await message.answer(txt)
    
    elif args[1] == "order" and len(args) >= 3:
        try:
            order_id = int(args[2])
        except:
            await message.answer("❌ Неверный номер заказа. Пример: /admin order 123")
            return
        
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT * FROM orders WHERE order_id = ?", (order_id,))
        order = c.fetchone()
        conn.close()
        
        if not order:
            await message.answer(f"❌ Заказ <code>#{order_id}</code> не найден.")
            return
        
        await message.answer(
            f"📦 <b>ЗАКАЗ #{order[0]}</b>\n\n"
            f"👤 Юзер: @{order[2]} (ID: <code>{order[1]}</code>)\n"
            f"📦 Товар: {order[3]}\n"
            f"💰 Сумма: {order[4]} ⭐\n"
            f"📍 Город: {order[5]}\n"
            f"🏘️ Район: {order[6]}\n"
            f"🏠 Адрес: {order[7]}\n"
            f"🔍 Клад: {order[8]}\n"
            f"🎟️ Промокод: {order[9] or 'нет'}\n"
            f"📅 Дата: {order[10]}"
        )
    
    elif args[1] == "users":
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT user_id, username, total_spent, orders_count, balance_stars FROM users ORDER BY total_spent DESC LIMIT 10")
        users = c.fetchall()
        conn.close()
        
        txt = "👤 <b>ТОП-10 ЮЗЕРОВ ПО РАСХОДАМ:</b>\n\n"
        for i, u in enumerate(users, 1):
            txt += f"{i}. @{u[1]} (ID: <code>{u[0]}</code>)\n"
            txt += f"   💰 Потрачено: {u[2]} ⭐ | 📦 Заказов: {u[3]} | 💎 Баланс: {u[4]} ⭐\n\n"
        
        await message.answer(txt)
    
    elif args[1] == "user" and len(args) >= 3:
        try:
            uid = int(args[2])
        except:
            await message.answer("❌ Неверный ID. Пример: /admin user 123456789")
            return
        
        user = get_user(uid)
        if not user:
            await message.answer("❌ Юзер не найден в базе.")
            return
        
        # Считаем заказы юзера
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT COUNT(*) FROM orders WHERE user_id = ?", (uid,))
        order_count = c.fetchone()[0]
        c.execute("SELECT COALESCE(SUM(amount_stars), 0) FROM orders WHERE user_id = ?", (uid,))
        total_spent_orders = c.fetchone()[0]
        conn.close()
        
        await message.answer(
            f"👤 <b>ИНФО О ЮЗЕРЕ</b>\n\n"
            f"ID: <code>{user[0]}</code>\n"
            f"Username: @{user[1]}\n"
            f"Имя: {user[2]}\n"
            f"Баланс: <b>{user[3]} ⭐</b>\n"
            f"Потрачено всего: <b>{user[4]} ⭐</b>\n"
            f"Заказов: <b>{user[5]}</b>\n"
            f"Реферер ID: {user[6] or 'нет'}\n"
            f"Бонус реферала получен: {'✅ да' if user[7] else '❌ нет'}\n"
            f"Реф.код: <code>{user[8]}</code>\n"
            f"Дата регистрации: {user[9]}\n\n"
            f"📊 <b>Из таблицы заказов:</b>\n"
            f"Заказов: {order_count}\n"
            f"Потрачено: {total_spent_orders} ⭐"
        )
    
    elif args[1] == "promo":
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT * FROM promo_codes ORDER BY created_at DESC")
        promos = c.fetchall()
        conn.close()
        
        if not promos:
            await message.answer("🎟️ Нет активных промокодов.")
            return
        
        txt = "🎟️ <b>ВСЕ ПРОМОКОДЫ:</b>\n\n"
        for p in promos:
            txt += f"<code>{p[0]}</code> — скидка {p[1]}%, осталось {p[2]} исп., создал ID:{p[3]}\n"
        
        await message.answer(txt)
    
    elif args[1] == "broadcast" and len(args) >= 3:
        text = " ".join(args[2:])
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("SELECT user_id FROM users")
        users = c.fetchall()
        conn.close()
        
        msg = await message.answer(f"📢 Начинаю рассылку на {len(users)} юзеров...")
        
        sent = 0
        failed = 0
        for u in users:
            try:
                await bot.send_message(u[0], f"📢 <b>ZETA SHOP</b>\n\n{text}\n\nС уважением, команда Zeta.", disable_web_page_preview=True)
                sent += 1
            except:
                failed += 1
            await asyncio.sleep(0.05)
        
        await msg.edit_text(f"✅ <b>РАССЫЛКА ЗАВЕРШЕНА</b>\n\nОтправлено: {sent}\nОшибок: {failed}\nВсего юзеров: {len(users)}")
    
    elif args[1] == "addbalance" and len(args) >= 4:
        try:
            uid = int(args[2])
            amount = int(args[3])
        except:
            await message.answer("❌ Формат: /admin addbalance ID СУММА\nПример: /admin addbalance 123456789 500")
            return
        
        user = get_user(uid)
        if not user:
            await message.answer("❌ Юзер не найден. Создаю...")
            create_user(uid, "unknown", "unknown")
        
        conn = sqlite3.connect("zeta.db")
        c = conn.cursor()
        c.execute("UPDATE users SET balance_stars = balance_stars + ? WHERE user_id = ?", (amount, uid))
        conn.commit()
        
        c.execute("SELECT balance_stars FROM users WHERE user_id = ?", (uid,))
        new_balance = c.fetchone()[0]
        conn.close()
        
        await message.answer(f"✅ <b>+{amount} ⭐</b> юзеру <code>{uid}</code>\nНовый баланс: <b>{new_balance} ⭐</b>")
        
        # Уведомляем юзера
        try:
            await bot.send_message(uid, f"🎁 <b>БОНУС +{amount} ⭐!</b>\n\nZeta Shop начислил тебе {amount} ⭐ на баланс.\nТекущий баланс: {new_balance} ⭐\n\nПриятных покупок! 🚀")
        except:
            pass

# ==================== ЗАПУСК ====================
async def main():
    init_db()
    print("🚀 ZETA SHOP ЗАПУЩЕН!")
    print(f"👑 Админ ID: {ADMIN_ID}")
    print(f"🤖 Куратор: @{CURATOR_USERNAME}")
    print(f"📦 Товаров: {len(CATALOG)}")
    print(f"🏙️ Городов: {len(CITIES)}")
    print("=" * 40)
    
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
