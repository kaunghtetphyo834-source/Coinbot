# Coinbot
Telegram Coin Bot for Business with Auto Reward, Referral &amp; Withdraw System
import sqlite3
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes

# ========== SETTINGS ==========
TOKEN = "8428578247:AAGPfitFAkFfiAH3UxjCK4JeMgom_FpEqn0"   # BotFather token
ADMIN_ID = 6554048431                                      # Telegram Admin ID
CHANNEL_USERNAME = "https://t.me/findmoneymanymore"       # Public Channel invite link

MIN_WITHDRAW = 200
DAILY_LIMIT = 50
REWARD_PER_WATCH = 10
REFERRAL_BONUS = 20

# ========== DATABASE ==========
conn = sqlite3.connect("coins.db")
c = conn.cursor()
c.execute("""CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    coins INTEGER DEFAULT 0,
    referrer INTEGER,
    watch_count INTEGER DEFAULT 0,
    last_watch_date TEXT,
    banned INTEGER DEFAULT 0
)""")
conn.commit()

# ========== UTILITIES ==========
async def check_join(user_id, bot):
    try:
        member = await bot.get_chat_member(CHANNEL_USERNAME, user_id)
        return member.status in ["member", "administrator", "creator"]
    except:
        return True  # Invite link / private channel → skip check

# ========== COMMANDS ==========
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.message.from_user.id

    if not await check_join(user_id, context.bot):
        keyboard = [[InlineKeyboardButton("📢 Join Channel", url=CHANNEL_USERNAME)]]
        await update.message.reply_text("🚫 Join Channel First!",
                                        reply_markup=InlineKeyboardMarkup(keyboard))
        return

    ref = int(context.args[0]) if context.args else None

    c.execute("SELECT * FROM users WHERE user_id=?", (user_id,))
    if not c.fetchone():
        c.execute("INSERT INTO users (user_id, coins, referrer) VALUES (?, ?, ?)",
                  (user_id, 0, ref))
        if ref:
            c.execute("UPDATE users SET coins = coins + ? WHERE user_id=?", (REFERRAL_BONUS, ref))
    conn.commit()

    keyboard = [
        [InlineKeyboardButton("🎥 Watch Ad", callback_data="watch")],
        [InlineKeyboardButton("💰 Balance", callback_data="balance")],
        [InlineKeyboardButton("🏧 Withdraw", callback_data="withdraw")],
        [InlineKeyboardButton("🏆 Leaderboard", callback_data="top")]
    ]
    await update.message.reply_text("💎 Professional Coin Bot",
                                    reply_markup=InlineKeyboardMarkup(keyboard))

# ========== BUTTON HANDLER ==========
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    await query.answer()

    c.execute("SELECT coins, watch_count, last_watch_date, banned FROM users WHERE user_id=?", (user_id,))
    user = c.fetchone()
    if not user:
        return

    coins, watch_count, last_watch_date, banned = user
    today = datetime.now().strftime("%Y-%m-%d")

    if last_watch_date != today:
        watch_count = 0  # reset daily

    if banned:
        await query.edit_message_text("🚫 You are banned.")
        return

    # ===== Watch Ad =====
    if query.data == "watch":
        if watch_count >= DAILY_LIMIT:
            await query.edit_message_text(f"⛔ Daily limit {DAILY_LIMIT} reached!")
            return

        coins += REWARD_PER_WATCH
        watch_count += 1
        c.execute("""UPDATE users 
                     SET coins=?, watch_count=?, last_watch_date=? 
                     WHERE user_id=?""",
                  (coins, watch_count, today, user_id))
        conn.commit()
        await query.edit_message_text(f"🎉 +{REWARD_PER_WATCH} Coins!\nToday's Count: {watch_count}/{DAILY_LIMIT}\nBalance: {coins}")

    # ===== Balance =====
    elif query.data == "balance":
        await query.edit_message_text(f"💰 Balance: {coins}")

    # ===== Withdraw =====
    elif query.data == "withdraw":
        if coins < MIN_WITHDRAW:
            await query.edit_message_text(f"❌ Not enough coins. Minimum: {MIN_WITHDRAW}")
            return

        keyboard = [[InlineKeyboardButton("✅ Approve", callback_data=f"approve_{user_id}")]]
        await context.bot.send_message(ADMIN_ID,
                                       f"Withdraw Request\nUser: {user_id}\nCoins: {coins}",
                                       reply_markup=InlineKeyboardMarkup(keyboard))
        await query.edit_message_text("✅ Request sent to admin.")

    # ===== Approve Withdraw =====
    elif query.data.startswith("approve_"):
        if user_id != ADMIN_ID:
            return
        uid = int(query.data.split("_")[1])
        c.execute("UPDATE users SET coins=0 WHERE user_id=?", (uid,))
        conn.commit()
        await query.edit_message_text("✅ Withdraw Approved & Coins Reset.")

    # ===== Leaderboard =====
    elif query.data == "top":
        c.execute("SELECT user_id, coins FROM users ORDER BY coins DESC LIMIT 5")
        top = c.fetchall()
        text = "🏆 Leaderboard\n\n"
        for i, u in enumerate(top, start=1):
            text += f"{i}. {u[0]} - {u[1]} coins\n"
        await query.edit_message_text(text)

# ========== APP RUN ==========
app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(button))
app.run_polling()
