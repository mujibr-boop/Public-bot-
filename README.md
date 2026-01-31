
import random
import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, ContextTypes

# --- DATABASE SETUP ---
conn = sqlite3.connect("/mnt/data/easyearnify.db")  # Railway persistent path
c = conn.cursor()
c.execute("""CREATE TABLE IF NOT EXISTS users(
            user_id INTEGER PRIMARY KEY,
            username TEXT,
            points REAL DEFAULT 0,
            tasks_completed INTEGER DEFAULT 0,
            invites INTEGER DEFAULT 0
            )""")
conn.commit()

# --- CONSTANTS ---
CHANNELS = [
    "https://t.me/incomelooops",
    "https://t.me/reviewworklive",
    "https://t.me/earnbyrealbotnow"
]

TASKS = [
    "https://web.earnbox.net/h5/#/?salt=E2psxuKVMn",
    "https://www.effectivegatecpm.com/u39c97tf?key=311b89465707b46bd5a609ac8ff9466c",
    "https://web.cashin.life/?salt=ltZV7TKeyz",
    "https://r.navi.com/7NFTJB",
    "https://www.effectivegatecpm.com/u39c97tf?key=311b89465707b46bd5a609ac8ff9466c"
]

MAX_DAILY_TASKS = 5
MIN_WITHDRAW = 50.0
REFERRAL_PERCENT = 0.10

# --- HELPER FUNCTIONS ---
def get_user(user_id, username):
    c.execute("SELECT * FROM users WHERE user_id=?", (user_id,))
    data = c.fetchone()
    if not data:
        c.execute("INSERT INTO users(user_id, username) VALUES (?,?)", (user_id, username))
        conn.commit()
        return get_user(user_id, username)
    return data

def random_points():
    return round(random.uniform(8.1, 9.9), 2)

# --- COMMAND HANDLERS ---
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    username = update.effective_user.username or update.effective_user.first_name
    get_user(user_id, username)
    
    text = f"👋 Hello {username}, welcome to EasyEarnify Bot!\n\n" \
           "To start earning, please join all 3 required channels and verify:"
    buttons = [[InlineKeyboardButton(f"Join Channel {i+1}", url=link)] for i, link in enumerate(CHANNELS)]
    buttons.append([InlineKeyboardButton("✅ Verify", callback_data="verify_channels")])
    keyboard = InlineKeyboardMarkup(buttons)
    
    await update.message.reply_text(text, reply_markup=keyboard)

# --- CALLBACK HANDLER ---
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    username = query.from_user.username or query.from_user.first_name
    data = query.data
    user = get_user(user_id, username)
    
    if data == "verify_channels":
        # Simulate verification (replace with real check if needed)
        await query.answer("Channels verified (simulated).")
        await show_tasks(query, user)
    
    elif data.startswith("task_"):
        task_index = int(data.split("_")[1])
        tasks_done = user[3]  # tasks_completed
        if tasks_done >= MAX_DAILY_TASKS:
            await query.edit_message_text(
                "⏳ You’ve reached today’s task limit!\n"
                "Come back tomorrow for new tasks, or invite friends to earn 10% of their points."
            )
            return
        
        points_earned = random_points()
        c.execute("UPDATE users SET points=points+?, tasks_completed=tasks_completed+1 WHERE user_id=?",
                  (points_earned, user_id))
        conn.commit()
        await query.answer(f"Task completed! You earned {points_earned} points.")
        await show_tasks(query, get_user(user_id, username))  # show remaining tasks

    elif data == "withdraw":
        if user[2] < MIN_WITHDRAW:
            await query.answer(f"❌ You need at least {MIN_WITHDRAW} points to withdraw.")
        else:
            await query.answer("✅ Withdrawal request received. Admin will manually review.")

    elif data == "refer":
        invite_link = f"https://t.me/@easyearnifybot?start={user_id}"
        await query.answer(f"Invite friends using this link: {invite_link}")

    elif data == "warning":
        await query.answer(
            "⚠️ Complete tasks honestly. Bot cannot verify automatically. Payment may be held."
        )

# --- SHOW TASKS ---
async def show_tasks(query, user):
    tasks_done = user[3]
    buttons = []
    for i, link in enumerate(TASKS):
        if i >= tasks_done:
            buttons.append([InlineKeyboardButton(f"🧩 Task {i+1}", url=link, callback_data=f"task_{i}")])
    # Add navigation buttons
    buttons.append([InlineKeyboardButton("💵 Withdraw", callback_data="withdraw")])
    buttons.append([InlineKeyboardButton("🤝 Refer & Earn", callback_data="refer")])
    buttons.append([InlineKeyboardButton("⚠️ Warning", callback_data="warning")])
    
    keyboard = InlineKeyboardMarkup(buttons)
    await query.edit_message_text("💰 Select a task to earn points:", reply_markup=keyboard)

# --- MAIN ---
if __name__ == "__main__":
    app = ApplicationBuilder().token("8241094570:AAE-o2lCKaKlCnxTdOA7XEtw29P9i_EOi-4").build()
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    
    print("Bot is running...")
    app.run_polling()
    python-telegram-bot==20.4
    
