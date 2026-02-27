# my-bot
import json
import logging
from datetime import datetime, timedelta, timezone

from telegram import (
    Update,
    ChatPermissions,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
)

from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    MessageHandler,
    CallbackQueryHandler,
    filters,
)

# ---------------- CONFIG ----------------

TOKEN = "YOUR_BOT_TOKEN"
DATA_FILE = "data.json"

logging.basicConfig(level=logging.INFO)

# ---------------- DATABASE ----------------

def load():
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {}

def save():
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

data = load()

def chat_data(chat_id):

    chat_id = str(chat_id)

    if chat_id not in data:

        data[chat_id] = {

            "owner": None,
            "admins": [],
            "helpers": [],

            "lock_chat": False,
            "lock_link": False,
            "lock_media": False,

            "filters": [],

            "welcome": "خوش آمدی {name} 🌹",

            "warns": {}

        }

    return data[chat_id]

# ---------------- ACCESS ----------------

def is_owner(chat, user_id):
    return chat["owner"] == user_id

def is_admin(chat, user_id):
    return user_id in chat["admins"]

def is_helper(chat, user_id):
    return user_id in chat["helpers"]

def is_mod(chat, user_id):
    return (
        is_owner(chat, user_id)
        or is_admin(chat, user_id)
        or is_helper(chat, user_id)
    )

# ---------------- WELCOME ----------------

async def welcome(update: Update, context: ContextTypes.DEFAULT_TYPE):

    chat = chat_data(update.effective_chat.id)

    for user in update.message.new_chat_members:

        text = chat["welcome"].replace("{name}", user.first_name)

        await update.message.reply_text(text)

# ---------------- BUTTON PANEL ----------------

async def panel_buttons(update, context):

    query = update.callback_query
    await query.answer()

    chat = chat_data(query.message.chat.id)

    if query.data == "lock_chat":

        chat["lock_chat"] = True
        save()
        await query.edit_message_text("چت قفل شد")

    elif query.data == "unlock_chat":

        chat["lock_chat"] = False
        save()
        await query.edit_message_text("چت باز شد")

    elif query.data == "status":

        text = f"""
قفل چت: {chat['lock_chat']}
قفل لینک: {chat['lock_link']}
قفل رسانه: {chat['lock_media']}
"""
        await query.edit_message_text(text)

# ---------------- PERSIAN COMMANDS ----------------

async def persian(update: Update, context: ContextTypes.DEFAULT_TYPE):

    if not update.message or not update.message.text:
        return

    text = update.message.text.strip()

    chat = chat_data(update.effective_chat.id)
    user_id = update.effective_user.id

    # -------- تنظیم مالک --------

    if text == "تنظیم مالک":

        if chat["owner"] is None:

            chat["owner"] = user_id
            save()

            await update.message.reply_text("مالک تنظیم شد")

        return

    # -------- افزودن ادمین --------

    if text == "افزودن ادمین":

        if not is_owner(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        if target not in chat["admins"]:
            chat["admins"].append(target)
            save()

        await update.message.reply_text("ادمین اضافه شد")
        return

    # -------- حذف ادمین --------

    if text == "حذف ادمین":

        if not is_owner(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        if target in chat["admins"]:
            chat["admins"].remove(target)
            save()

        await update.message.reply_text("ادمین حذف شد")
        return

    # -------- افزودن کمک ادمین --------

    if text == "افزودن کمک ادمین":

        if not is_owner(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        if target not in chat["helpers"]:
            chat["helpers"].append(target)
            save()

        await update.message.reply_text("کمک ادمین اضافه شد")
        return

    # -------- حذف کمک ادمین --------

    if text == "حذف کمک ادمین":

        if not is_owner(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        if target in chat["helpers"]:
            chat["helpers"].remove(target)
            save()

        await update.message.reply_text("کمک ادمین حذف شد")
        return

    # -------- لیست ادمین --------

    if text == "لیست ادمین":

        await update.message.reply_text(str(chat))
        return

    # -------- سکوت --------

    if text.startswith("سکوت"):

        if not is_mod(chat, user_id):
            return

        if not update.message.reply_to_message:
            return

        minutes = int(text.split()[1])

        target = update.message.reply_to_message.from_user.id

        until = datetime.now(timezone.utc) + timedelta(minutes=minutes)

        await context.bot.restrict_chat_member(

            update.effective_chat.id,
            target,
            ChatPermissions(can_send_messages=False),
            until_date=until
        )

        await update.message.reply_text(f"سکوت {minutes} دقیقه اعمال شد")
        return

    # -------- آزاد --------

    if text == "آزاد":

        if not is_mod(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        await context.bot.restrict_chat_member(

            update.effective_chat.id,
            target,
            ChatPermissions(
                can_send_messages=True,
                can_send_media_messages=True,
                can_send_other_messages=True,
                can_add_web_page_previews=True
            )
        )

        await update.message.reply_text("کاربر آزاد شد")
        return

    # -------- بن --------

    if text == "بن":

        if not is_admin(chat, user_id) and not is_owner(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        await context.bot.ban_chat_member(
            update.effective_chat.id,
            target
        )

        await update.message.reply_text("کاربر بن شد")
        return

    # -------- آنبن --------

    if text.startswith("آنبن"):

        if not is_admin(chat, user_id) and not is_owner(chat, user_id):
            return

        target = int(text.split()[1])

        await context.bot.unban_chat_member(
            update.effective_chat.id,
            target
        )

        await update.message.reply_text("کاربر آنبن شد")
        return

    # -------- اخطار --------

    if text == "اخطار":

        if not is_mod(chat, user_id):
            return

        target = update.message.reply_to_message.from_user.id

        warns = chat["warns"]

        warns[str(target)] = warns.get(str(target), 0) + 1

        save()

        if warns[str(target)] >= 3:

            await context.bot.ban_chat_member(
                update.effective_chat.id,
                target
            )

            await update.message.reply_text("کاربر بن شد")

        else:

            await update.message.reply_text(
                f"اخطار {warns[str(target)]}/3"
            )

        return

    # -------- پاکسازی --------

    if text.startswith("پاکسازی"):

        if not is_mod(chat, user_id):
            return

        count = int(text.split()[1])

        await update.message.reply_text("در حال پاکسازی...")

        async for msg in context.bot.get_chat_history(
            update.effective_chat.id,
            limit=count
        ):
            try:
                await context.bot.delete_message(
                    update.effective_chat.id,
                    msg.message_id
                )
            except:
                pass

        return

    # -------- قفل ها --------

    if text == "قفل چت":
        chat["lock_chat"] = True
        save()
        await update.message.reply_text("قفل شد")

    if text == "بازکردن چت":
        chat["lock_chat"] = False
        save()
        await update.message.reply_text("باز شد")

    if text == "قفل لینک":
        chat["lock_link"] = True
        save()
        await update.message.reply_text("قفل شد")

    if text == "بازکردن لینک":
        chat["lock_link"] = False
        save()
        await update.message.reply_text("باز شد")

    if text == "قفل رسانه":
        chat["lock_media"] = True
        save()
        await update.message.reply_text("قفل شد")

    if text == "بازکردن رسانه":
        chat["lock_media"] = False
        save()
        await update.message.reply_text("باز شد")

    # -------- فیلتر --------

    if text.startswith("افزودن فیلتر"):

        word = text.replace("افزودن فیلتر ", "")
        chat["filters"].append(word)
        save()

        await update.message.reply_text("اضافه شد")

    if text.startswith("حذف فیلتر"):

        word = text.replace("حذف فیلتر ", "")

        if word in chat["filters"]:
            chat["filters"].remove(word)
            save()

        await update.message.reply_text("حذف شد")

    if text == "لیست فیلتر":

        await update.message.reply_text("\n".join(chat["filters"]))

    # -------- خوش آمد --------

    if text.startswith("تنظیم خوشآمدی"):

        chat["welcome"] = text.replace(
            "تنظیم خوشآمدی ", ""
        )

        save()

        await update.message.reply_text("تنظیم شد")

    # -------- پنل --------

    if text == "پنل":

        keyboard = [

            [InlineKeyboardButton("قفل چت", callback_data="lock_chat")],
            [InlineKeyboardButton("بازکردن چت", callback_data="unlock_chat")],
            [InlineKeyboardButton("وضعیت", callback_data="status")],

        ]

        await update.message.reply_text(
            "پنل مدیریت",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

# ---------------- MESSAGE FILTER ----------------

async def check(update, context):

    if not update.message:
        return

    chat = chat_data(update.effective_chat.id)

    user_id = update.effective_user.id

    if is_mod(chat, user_id):
        return

    if chat["lock_chat"]:
        await update.message.delete()
        return

    if update.message.text:

        for word in chat["filters"]:

            if word in update.message.text:
                await update.message.delete()
                return

        if chat["lock_link"] and "http" in update.message.text:
            await update.message.delete()

    if chat["lock_media"]:

        if update.message.photo or update.message.video:
            await update.message.delete()

# ---------------- RUN ----------------

app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(
    MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, welcome)
)

app.add_handler(
    CallbackQueryHandler(panel_buttons)
)

app.add_handler(
    MessageHandler(filters.TEXT, persian)
)

app.add_handler(
    MessageHandler(filters.ALL, check)
)

print("Bot running...")

app.run_polling()
