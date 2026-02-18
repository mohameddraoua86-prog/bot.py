# bot.py#
!/usr/bin/env python3
# -*- coding: utf-8 -*-
# CryptoMiner Pro - الإصدار المحدث 2026

import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import sqlite3
import time
import random
import requests
from datetime import datetime, timedelta
import os
import sys

# -------------------- الإعدادات الأساسية --------------------
BOT_TOKEN = "8443416735:AAHCBArK4hN6t0CXT8hBv6NPT0MWivfGzb8"
CHANNEL_USERNAME = "@ISLEM_TV"  # قناة الاشتراك الإجباري
ADMIN_ID = 6620908892  # ضع معرفك هنا
MONETAG_SCRIPT = '<script src="//pl123456.com/script.js"></script>'  # ضع كود Monetag هنا
CURRENCY_NAME = "⚡ عملة التعدين"
REFERRAL_BONUS = 0.2  # 20% عمولة إحالة
MIN_WITHDRAW = 1000
# -----------------------------------------------------------

# حذف أي webhook قديم
try:
    requests.post(f"https://api.telegram.org/bot{BOT_TOKEN}/deleteWebhook?drop_pending_updates=true")
    print("✅ تم حذف webhook القديم")
except Exception as e:
    print(f"⚠️ فشل حذف webhook: {e}")

bot = telebot.TeleBot(BOT_TOKEN)

# -------------------- قاعدة البيانات --------------------
conn = sqlite3.connect('miner.db', check_same_thread=False)
cursor = conn.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS users (
        user_id INTEGER PRIMARY KEY,
        username TEXT,
        first_name TEXT,
        balance REAL DEFAULT 0,
        mining_power INTEGER DEFAULT 1,
        level INTEGER DEFAULT 1,
        energy INTEGER DEFAULT 1000,
        max_energy INTEGER DEFAULT 1000,
        last_mining TIMESTAMP,
        referred_by INTEGER,
        referrals_count INTEGER DEFAULT 0,
        total_earned REAL DEFAULT 0,
        join_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS boosts (
        boost_id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        description TEXT,
        cost REAL,
        energy_bonus INTEGER,
        mining_bonus INTEGER,
        level_required INTEGER
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS user_boosts (
        user_id INTEGER,
        boost_id INTEGER,
        purchased_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        expires_at TIMESTAMP,
        FOREIGN KEY(user_id) REFERENCES users(user_id),
        FOREIGN KEY(boost_id) REFERENCES boosts(boost_id)
    )
''')

cursor.execute('''
    CREATE TABLE IF NOT EXISTS withdrawal_requests (
        request_id INTEGER PRIMARY KEY AUTOINCREMENT,
        user_id INTEGER,
        amount REAL,
        wallet_address TEXT,
        status TEXT DEFAULT 'pending',
        request_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        FOREIGN KEY(user_id) REFERENCES users(user_id)
    )
''')

# إضافة بعض البوستات الأساسية
cursor.execute("SELECT COUNT(*) FROM boosts")
if cursor.fetchone()[0] == 0:
    boosts = [
        ("⚡ شاحن الطاقة", "يزيد الطاقة القصوى بمقدار 500", 500, 500, 0, 1),
        ("⛏️ آلة تعدين", "يزيد قوة التعدين بمقدار 2", 1000, 0, 2, 2),
        ("💎 بطارية متطورة", "يزيد الطاقة القصوى بمقدار 1000", 2000, 1000, 0, 3),
        ("🚀 مولد عملاق", "يزيد قوة التعدين بمقدار 5", 5000, 0, 5, 5),
        ("🔋 نظام طاقة لامحدود", "طاقة لا نهائية لمدة 24 ساعة", 10000, 10000, 0, 10),
    ]
    cursor.executemany("INSERT INTO boosts (name, description, cost, energy_bonus, mining_bonus, level_required) VALUES (?,?,?,?,?,?)", boosts)
    conn.commit()

conn.commit()

# -------------------- دوال مساعدة --------------------
def get_user(user_id):
    cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    return cursor.fetchone()

def register_user(user_id, username, first_name, referred_by=None):
    try:
        cursor.execute('''
            INSERT INTO users (user_id, username, first_name, referred_by) 
            VALUES (?, ?, ?, ?)
        ''', (user_id, username, first_name, referred_by))
        conn.commit()
        return True
    except sqlite3.IntegrityError:
        return False

def update_balance(user_id, amount):
    cursor.execute("UPDATE users SET balance = balance + ? WHERE user_id = ?", (amount, user_id))
    conn.commit()

def get_balance(user_id):
    cursor.execute("SELECT balance FROM users WHERE user_id = ?", (user_id,))
    return cursor.fetchone()[0]

def add_referral_bonus(referrer_id, amount):
    cursor.execute("UPDATE users SET balance = balance + ?, total_earned = total_earned + ? WHERE user_id = ?", 
                   (amount, amount, referrer_id))
    conn.commit()

def is_member(user_id):
    try:
        member = bot.get_chat_member(CHANNEL_USERNAME, user_id)
        return member.status in ['creator', 'administrator', 'member', 'restricted']
    except:
        return False

# -------------------- لوحات المفاتيح --------------------
def main_keyboard():
    keyboard = InlineKeyboardMarkup(row_width=2)
    keyboard.add(
        InlineKeyboardButton("⛏️ ابدأ التعدين", callback_data="mine"),
        InlineKeyboardButton("💰 رصيدي", callback_data="balance")
    )
    keyboard.add(
        InlineKeyboardButton("📈 المتجر", callback_data="shop"),
        InlineKeyboardButton("👥 الإحالات", callback_data="referrals")
    )
    keyboard.add(
        InlineKeyboardButton("🏆 المستوى", callback_data="level"),
        InlineKeyboardButton("💳 سحب", callback_data="withdraw")
    )
    keyboard.add(
        InlineKeyboardButton("📺 شاهد إعلان", callback_data="watch_ad"),
        InlineKeyboardButton("🆘 مساعدة", callback_data="help")
    )
    return keyboard

def shop_keyboard():
    cursor.execute("SELECT boost_id, name, cost, level_required FROM boosts ORDER BY cost")
    boosts = cursor.fetchall()
    keyboard = InlineKeyboardMarkup(row_width=1)
    for boost in boosts:
        btn_text = f"{boost[1]} - {boost[2]} {CURRENCY_NAME} (مستوى {boost[3]}+)"
        keyboard.add(InlineKeyboardButton(btn_text, callback_data=f"buy_{boost[0]}"))
    keyboard.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_main"))
    return keyboard

def back_keyboard():
    keyboard = InlineKeyboardMarkup()
    keyboard.add(InlineKeyboardButton("🔙 رجوع", callback_data="back_main"))
    return keyboard

# -------------------- معالجات الأوامر --------------------
@bot.message_handler(commands=['start'])
def start_command(message):
    user_id = message.from_user.id
    username = message.from_user.username or ""
    first_name = message.from_user.first_name or ""

    # التحقق من وجود referrer
    referred_by = None
    if len(message.text.split()) > 1:
        try:
            referred_by = int(message.text.split()[1])
        except:
            pass

    # تسجيل المستخدم إذا لم يكن موجوداً
    if not get_user(user_id):
        register_user(user_id, username, first_name, referred_by)
        # مكافأة للمُحيل
        if referred_by and referred_by != user_id:
            cursor.execute("UPDATE users SET referrals_count = referrals_count + 1 WHERE user_id = ?", (referred_by,))
            add_referral_bonus(referred_by, 100)  # مكافأة فورية 100
            conn.commit()

    # التحقق من العضوية في القناة
    if not is_member(user_id):
        markup = InlineKeyboardMarkup()
        markup.add(InlineKeyboardButton("📢 انضم للقناة", url=f"https://t.me/{CHANNEL_USERNAME[1:]}"))
        markup.add(InlineKeyboardButton("✅ تحققت", callback_data="check_member"))
        bot.send_message(
            message.chat.id,
            f"⚠️ *يجب الانضمام إلى {CHANNEL_USERNAME} أولاً.*\n\nبعد الانضمام، اضغط 'تحققت'.",
            parse_mode="Markdown",
            reply_markup=markup
        )
        return

    # رسالة الترحيب
    welcome = f"✨ *مرحباً بك {first_name} في CryptoMiner Pro!*\n\n"
    welcome += "هذا البوت يتيح لك تعدين العملات الافتراضية وكسب أرباح حقيقية.\n"
    welcome += "ابدأ بالتعدين الآن، ادعُ أصدقاءك، واربح المزيد.\n\n"
    welcome += f"رصيدك الحالي: *{get_balance(user_id)} {CURRENCY_NAME}*"

    bot.send_message(message.chat.id, welcome, parse_mode="Markdown", reply_markup=main_keyboard())

@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    user_id = call.from_user.id
    data = call.data

    # التحقق من العضوية (عدا check_member)
    if data != "check_member" and not is_member(user_id):
        bot.answer_callback_query(call.id, "❌ يجب الانضمام للقناة أولاً.", show_alert=True)
        return

    # تسجيل المستخدم إذا لم يكن موجوداً
    if not get_user(user_id):
        register_user(user_id, "", "")
        bot.answer_callback_query(call.id, "تم تسجيلك بنجاح!")

    # -------------------- التحقق من العضوية --------------------
    if data == "check_member":
        if is_member(user_id):
            bot.answer_callback_query(call.id, "✅ تم التحقق!")
            start_command(call.message)
        else:
            bot.answer_callback_query(call.id, "❌ لم تنضم بعد.", show_alert=True)
        return

    # -------------------- التعدين --------------------
    if data == "mine":
        user = get_user(user_id)
        if not user:
            bot.answer_callback_query(call.id, "حدث خطأ، أعد /start")
            return

        # حساب الوقت المنقضي
        last_mining = user[7]  # last_mining
        now = datetime.now()
        mining_power = user[4]  # mining_power
        energy = user[6]  # energy

        if last_mining:
            last = datetime.strptime(last_mining, '%Y-%m-%d %H:%M:%S')
            elapsed = (now - last).total_seconds() / 60  # دقائق
            if elapsed < 1:
                bot.answer_callback_query(call.id, "❌ عليك الانتظار دقيقة بين كل تعدين.", show_alert=True)
                return

        # استهلاك الطاقة
        if energy < mining_power:
            bot.answer_callback_query(call.id, "❌ الطاقة غير كافية. اشترِ طاقة أو انتظر حتى تتجدد.", show_alert=True)
            return

        # حساب الأرباح
        earnings = random.randint(mining_power, mining_power * 5)
        update_balance(user_id, earnings)
        new_energy = energy - mining_power

        cursor.execute('''
            UPDATE users 
            SET last_mining = ?, energy = ?, total_earned = total_earned + ? 
            WHERE user_id = ?
        ''', (now.strftime('%Y-%m-%d %H:%M:%S'), new_energy, earnings, user_id))
        conn.commit()

        bot.answer_callback_query(call.id, f"✅ +{earnings} {CURRENCY_NAME}!")
        bot.send_message(
            call.message.chat.id,
            f"⛏️ لقد قمت بالتعدين وحصلت على *{earnings} {CURRENCY_NAME}*\n"
            f"الطاقة المتبقية: *{new_energy}*",
            parse_mode="Markdown",
            reply_markup=main_keyboard()
        )

    # -------------------- الرصيد --------------------
    elif data == "balance":
        user = get_user(user_id)
        if not user:
            bot.answer_callback_query(call.id, "حدث خطأ، أعد /start")
            return
        text = f"💰 *رصيدك:* {user[3]} {CURRENCY_NAME}\n"
        text += f"⚡ *الطاقة:* {user[6]}/{user[7]}\n"
        text += f"⛏️ *قوة التعدين:* {user[4]}\n"
        text += f"🏆 *المستوى:* {user[5]}\n"
        text += f"👥 *عدد الإحالات:* {user[10]}\n"
        text += f"📈 *الإجمالي المربوح:* {user[11]}"
        bot.edit_message_text(
            text,
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=back_keyboard()
        )

    # -------------------- المتجر --------------------
    elif data == "shop":
        bot.edit_message_text(
            "📈 *اختر البوست الذي تريد شراءه:*",
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=shop_keyboard()
        )

    # -------------------- شراء بوست --------------------
    elif data.startswith("buy_"):
        boost_id = int(data.split("_")[1])
        cursor.execute("SELECT * FROM boosts WHERE boost_id = ?", (boost_id,))
        boost = cursor.fetchone()
        if not boost:
            bot.answer_callback_query(call.id, "❌ هذا البوست غير موجود.")
            return

        user = get_user(user_id)
        if user[5] < boost[5]:  # مستوى المستخدم < المستوى المطلوب
            bot.answer_callback_query(call.id, f"❌ هذا البوست يتطلب مستوى {boost[5]} على الأقل.")
            return

        if user[3] < boost[3]:  # الرصيد < التكلفة
            bot.answer_callback_query(call.id, "❌ رصيدك غير كافٍ.")
            return

        # خصم الرصيد
        update_balance(user_id, -boost[3])
        # إضافة البوست للمستخدم
        cursor.execute('''
            INSERT INTO user_boosts (user_id, boost_id, expires_at) 
            VALUES (?, ?, ?)
        ''', (user_id, boost_id, (datetime.now() + timedelta(days=30)).strftime('%Y-%m-%d %H:%M:%S')))
        # تحديث إحصائيات المستخدم
        cursor.execute('''
            UPDATE users 
            SET max_energy = max_energy + ?, mining_power = mining_power + ? 
            WHERE user_id = ?
        ''', (boost[4], boost[5], user_id))
        conn.commit()

        bot.answer_callback_query(call.id, f"✅ تم شراء {boost[1]} بنجاح!")

    # -------------------- الإحالات --------------------
    elif data == "referrals":
        user = get_user(user_id)
        if not user:
            bot.answer_callback_query(call.id, "حدث خطأ، أعد /start")
            return
        referral_link = f"https://t.me/{(bot.get_me()).username}?start={user_id}"
        text = f"👥 *نظام الإحالات*\n\n"
        text += f"لقد دعوت *{user[10]}* شخصاً حتى الآن.\n"
        text += f"تكسب *{REFERRAL_BONUS*100}%* من أرباح كل من تدعوهم.\n"
        text += f"رابط الدعوة الخاص بك:\n`{referral_link}`\n\n"
        text += "شارك هذا الرابط مع أصدقائك لتربح المزيد!"
        bot.edit_message_text(
            text,
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=back_keyboard()
        )

    # -------------------- المستوى --------------------
    elif data == "level":
        user = get_user(user_id)
        if not user:
            bot.answer_callback_query(call.id, "حدث خطأ، أعد /start")
            return
        # نظام المستويات: كل 5000 أرباح ترتفع مستوى
        next_level = user[5] + 1
        required = next_level * 5000
        text = f"🏆 *المستوى الحالي:* {user[5]}\n"
        text += f"📊 *الأرباح الكلية:* {user[11]}\n"
        text += f"🎯 *المطلوب للمستوى التالي:* {required} {CURRENCY_NAME}\n"
        text += f"⚡ *الطاقة القصوى:* {user[7]}\n"
        text += f"⛏️ *قوة التعدين:* {user[4]}"
        bot.edit_message_text(
            text,
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=back_keyboard()
        )

    # -------------------- سحب --------------------
    elif data == "withdraw":
        user = get_user(user_id)
        if not user:
            bot.answer_callback_query(call.id, "حدث خطأ، أعد /start")
            return
        if user[3] < MIN_WITHDRAW:
            bot.answer_callback_query(call.id, f"❌ الحد الأدنى للسحب {MIN_WITHDRAW} {CURRENCY_NAME}.")
            return
        # هنا يمكن إضافة طلب سحب حقيقي
        bot.send_message(
            call.message.chat.id,
            "💰 *السحب*\n\n"
            "للسحب، يرجى إرسال عنوان محفظتك (USDT/TRC20) وسنقوم بمعالجة الطلب خلال 24 ساعة.\n"
            f"الحد الأدنى: {MIN_WITHDRAW} {CURRENCY_NAME}",
            parse_mode="Markdown"
        )

    # -------------------- مشاهدة إعلان --------------------
    elif data == "watch_ad":
        # هنا يمكن دمج إعلانات Monetag أو أي شبكة إعلانية
        bot.send_message(
            call.message.chat.id,
            "📺 *شاهد إعلان*\n\n"
            "يمكنك مشاهدة إعلان قصير وكسب 10 وحدات طاقة إضافية!\n"
            "[اضغط هنا لمشاهدة الإعلان](https://www.google.com)",
            parse_mode="Markdown",
            disable_web_page_preview=True
        )

    # -------------------- مساعدة --------------------
    elif data == "help":
        help_text = "🆘 *مساعدة*\n\n"
        help_text += "• اضغط '⛏️ ابدأ التعدين' لتكسب عملات.\n"
        help_text += "• الطاقة تتجدد تلقائياً كل دقيقة.\n"
        help_text += "• اشتري بوستات من المتجر لزيادة قوتك.\n"
        help_text += "• ادعُ أصدقاءك عبر رابط الإحالة لتربح عمولة.\n"
        help_text += "• اسحب أرباحك عندما تصل للحد الأدنى.\n"
        help_text += "• شاهد الإعلانات لتحصل على طاقة إضافية."
        bot.edit_message_text(
            help_text,
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=back_keyboard()
        )

    # -------------------- العودة للقائمة الرئيسية --------------------
    elif data == "back_main":
        bot.edit_message_text(
            "✨ *القائمة الرئيسية*",
            call.message.chat.id,
            call.message.message_id,
            parse_mode="Markdown",
            reply_markup=main_keyboard()
        )

# -------------------- تجديد الطاقة تلقائياً (مؤقت) --------------------
def energy_refresh():
    while True:
        time.sleep(60)  # كل دقيقة
        cursor.execute("UPDATE users SET energy = MIN(energy + 10, max_energy)")
        conn.commit()
        print("🔄 تم تجديد الطاقة لجميع المستخدمين")

import threading
threading.Thread(target=energy_refresh, daemon=True).start()

# -------------------- تشغيل البوت --------------------
if __name__ == "__main__":
    print("✅ البوت يعمل...")
    bot.infinity_polling()
