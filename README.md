import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackContext
import sqlite3

# ====== تنظیمات اولیه ======
TOKEN = "توکن رباتت اینجا"
CHANNEL_USERNAME = "@WeirdWonders"
JOIN_LINK = "https://t.me/WeirdWonders"

# ====== دیتابیس ساده ======
conn = sqlite3.connect("sekkeyar.db", check_same_thread=False)
cursor = conn.cursor()
cursor.execute("CREATE TABLE IF NOT EXISTS users (user_id INTEGER PRIMARY KEY, referrals INTEGER DEFAULT 0, coins INTEGER DEFAULT 0)")
conn.commit()

# ====== استارت ======
async def start(update: Update, context: CallbackContext):
    user = update.effective_user
    cursor.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user.id,))
    conn.commit()

    keyboard = [[InlineKeyboardButton("دعوت دوستان", callback_data="invite")]]
    reply_markup = InlineKeyboardMarkup(keyboard)
    
    await update.message.reply_text(
        f"سلام {user.first_name}!\n"
        "به ربات سکه‌یار خوش اومدی!\n"
        "برای جمع‌آوری سکه و دریافت جایزه دوستانت رو دعوت کن.",
        reply_markup=reply_markup
    )

# ====== دعوت ======
async def invite_callback(update: Update, context: CallbackContext):
    user_id = update.callback_query.from_user.id
    link = f"https://t.me/{context.bot.username}?start={user_id}"
    
    await update.callback_query.answer()
    await update.callback_query.message.reply_text(
        f"این لینک دعوت اختصاصی شماست:\n{link}\n"
        "به ازای هر 5 دعوت، 3 سکه می‌گیری. با 30 سکه جایزه بگیر!"
    )

# ====== بررسی دعوت ======
async def start_with_ref(update: Update, context: CallbackContext):
    user = update.effective_user
    args = context.args
    
    cursor.execute("INSERT OR IGNORE INTO users (user_id) VALUES (?)", (user.id,))
    conn.commit()

    if args:
        referrer_id = int(args[0])
        if referrer_id != user.id:
            cursor.execute("SELECT * FROM users WHERE user_id=?", (referrer_id,))
            if cursor.fetchone():
                cursor.execute("UPDATE users SET referrals = referrals + 1 WHERE user_id = ?", (referrer_id,))
                cursor.execute("UPDATE users SET coins = coins + 3 WHERE user_id = ?", (referrer_id,))
                conn.commit()

# ====== بررسی سکه ======
async def coins(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    cursor.execute("SELECT referrals, coins FROM users WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    if result:
        refs, coins = result
        await update.message.reply_text(
            f"تعداد دعوت‌ها: {refs}\n"
            f"سکه‌های شما: {coins}"
        )
    else:
        await update.message.reply_text("اطلاعاتی یافت نشد. /start رو بزن.")

# ====== جایزه ======
async def prize(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    cursor.execute("SELECT coins FROM users WHERE user_id=?", (user_id,))
    result = cursor.fetchone()
    if result and result[0] >= 30:
        cursor.execute("UPDATE users SET coins = coins - 30 WHERE user_id=?", (user_id,))
        conn.commit()
        await update.message.reply_text("جایزه شما: ||عضو شدن یادت نره||", parse_mode="MarkdownV2")
    else:
        await update.message.reply_text("برای دریافت جایزه باید حداقل 30 سکه داشته باشی.")

# ====== بررسی عضویت در کانال ======
async def check_join(update: Update, context: CallbackContext):
    user_id = update.effective_user.id
    try:
        member = await context.bot.get_chat_member(CHANNEL_USERNAME, user_id)
        if member.status in ['member', 'administrator', 'creator']:
            await update.message.reply_text("عضو کانال هستی، ادامه بده!")
        else:
            await update.message.reply_text(f"لطفاً اول عضو کانال شو:\n{JOIN_LINK}")
    except:
        await update.message.reply_text("خطا در بررسی عضویت. بعداً دوباره امتحان کن.")

# ====== راه‌اندازی ======
def main():
    logging.basicConfig(level=logging.INFO)
    app = Application.builder().token(TOKEN).build()
    
    app.add_handler(CommandHandler("start", start_with_ref))
    app.add_handler(CommandHandler("coins", coins))
    app.add_handler(CommandHandler("prize", prize))
    app.add_handler(CommandHandler("check", check_join))
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackContext.handler("invite", invite_callback))
    
    print("ربات در حال اجراست...")
    app.run_polling()

if __name__ == "__main__":
    main()
