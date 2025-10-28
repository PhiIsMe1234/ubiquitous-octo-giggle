import re
import hashlib
import logging
import random
import json
import os
from typing import Optional, Tuple, Dict, Any, List
from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    filters,
)

# ====== CONFIG ======
BOT_TOKEN = "7703970209:AAFwr1KGIBP8qsjYoYg-qttzA90-1OC5JwU"  # <-- thay token bot của bạn
ADMIN_ID = 7499529916              # <-- thay bằng Telegram ID admin của bạn
ADMIN_USERNAME = "@cskh_PhiIsMe"   # <-- username admin để liên hệ mua xu

REQUIRED_CHANNELS = [
    {"id": "@cosplayssexvn", "link": "https://t.me/cosplayssexvn"},
    {"id": "@BoxPhiIsMe", "link": "https://t.me/BoxPhiIsMe"},
]
# ====================

DATA_FILE = "Phii.json"

logger = logging.getLogger(__name__)

HEX_RE = re.compile(r"[0-9a-fA-F]{32}")

# -------------------------
# Persistence
# -------------------------
def load_data() -> Tuple[Dict[int, int], Dict[str, Dict[str, int]], Dict[int, List[str]], Dict[int, int]]:
    """
    Returns (user_xu, codes, used_codes, referrals)
    """
    if not os.path.exists(DATA_FILE):
        return {}, {}, {}, {}
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            raw = json.load(f)
        user_xu = {int(k): int(v) for k, v in raw.get("user_xu", {}).items()}
        codes = raw.get("codes", {})
        used_codes = {int(k): v for k, v in raw.get("used_codes", {}).items()}
        referrals = {int(k): int(v) for k, v in raw.get("referrals", {}).items()}
        return user_xu, codes, used_codes, referrals
    except Exception as e:
        logger.exception("Lỗi khi load data.json: %s", e)
        return {}, {}, {}, {}

def load_data() -> Tuple[Dict[int, int], Dict[str, Dict[str, int]], Dict[int, List[str]], Dict[int, int]]:
    if not os.path.exists(DATA_FILE):
        return {}, {}, {}, {}
    try:
        with open(DATA_FILE, "r", encoding="utf-8") as f:
            raw = json.load(f)
        user_xu = {int(k): int(v) for k, v in raw.get("user_xu", {}).items()}
        codes = raw.get("codes", {})
        used_codes = {int(k): v for k, v in raw.get("used_codes", {}).items()}
        referrals = {int(k): int(v) for k, v in raw.get("referrals", {}).items()}
        return user_xu, codes, used_codes, referrals
    except Exception as e:
        logger.exception("Lỗi khi load data.json: %s", e)
        return {}, {}, {}, {}

# -------------------------
# In-memory data
# -------------------------
user_xu, codes, used_codes, referrals = load_data()

# -------------------------
# Utilities
# -------------------------
def extract_md5_hex(text: str) -> Optional[str]:
    if not text:
        return None
    m = HEX_RE.search(text)
    return m.group(0).lower() if m else None

def md5_of_text(text: str) -> str:
    return hashlib.md5(text.encode("utf-8")).hexdigest()

def compute_percent_from_md5_hex(hexstr: str) -> Tuple[int, int, str]:
    if not hexstr:
        hexstr = md5_of_text("")
    hs = hexstr.strip().lower()
    if not HEX_RE.fullmatch(hs):
        hs = md5_of_text(hs)
    last2 = hs[-2:]
    try:
        val = int(last2, 16) % 100
    except Exception:
        val = 0
    tai = val
    xiu = 100 - val
    choice = "TÀI" if val >= 50 else "XỈU"
    return tai, xiu, choice

def gen_code() -> str:
    alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"
    return "MP5-" + "".join(random.choices(alphabet, k=6))

# -------------------------
# Channel verification
# -------------------------
async def check_joined_all(context: ContextTypes.DEFAULT_TYPE, user_id: int) -> Tuple[bool, List[Dict[str, str]]]:
    not_joined = []
    for ch in REQUIRED_CHANNELS:
        try:
            member = await context.bot.get_chat_member(ch["id"], user_id)
            if member.status not in ["member", "administrator", "creator"]:
                not_joined.append(ch)
        except Exception as e:
            logger.debug("check_joined_all lỗi với %s: %s", ch["id"], e)
            not_joined.append(ch)
    return (len(not_joined) == 0), not_joined

# -------------------------
# Start / verify
# -------------------------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    args = context.args
    inviter_id = None

    # Nếu có mã mời trong link
    if args:
        try:
            inviter_id = int(args[0])
        except ValueError:
            inviter_id = None

    user_xu.setdefault(uid, 0)
    save_needed = False

    # Nếu có người mời và chưa từng được mời ai
    if inviter_id and inviter_id != uid and uid not in referrals:
        referrals[uid] = inviter_id
        # cộng xu cho cả hai
        user_xu[inviter_id] = user_xu.get(inviter_id, 0) + 5
        user_xu[uid] = user_xu.get(uid, 0) + 5
        save_needed = True
        await update.message.reply_text(
            f"🎉 Bạn đã tham gia từ lời mời của ID {inviter_id}!\n"
            f"🎁 Cả bạn và người mời đều nhận được +5 xu!"
        )

    if save_needed:
        save_data(user_xu, codes, used_codes, referrals)

    await show_join_verify(update, context)


async def show_join_verify(update_or_query, context: ContextTypes.DEFAULT_TYPE):
    links = "\n".join(f"➡️ {c['link']}" for c in REQUIRED_CHANNELS)
    keyboard = [[InlineKeyboardButton("🔍 Xác minh tham gia", callback_data="check_join")]]
    reply_markup = InlineKeyboardMarkup(keyboard)

    text = (
        f"👋 Chào mừng!\n\n"
        f"🎯 Để dùng bot, bạn vui lòng tham gia các kênh sau:\n{links}\n\n"
        f"Sau đó bấm nút dưới để xác minh ✅"
    )

    if isinstance(update_or_query, Update):
        await update_or_query.message.reply_text(text, reply_markup=reply_markup)
    else:
        try:
            await update_or_query.edit_message_text(text, reply_markup=reply_markup)
        except Exception:
            await update_or_query.message.reply_text(text, reply_markup=reply_markup)

# -------------------------
# Callback verification
# -------------------------
from telegram import ReplyKeyboardMarkup

async def verify_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    uid = query.from_user.id

    joined_all, not_joined = await check_joined_all(context, uid)
    if joined_all:
        user_xu.setdefault(uid, 0)

        # === bàn phím reply (hiện dưới ô chat) ===
        reply_keyboard = [
            ["👤 Tài khoản", "💰 Nhập code"],
            ["🎁 Mời bạn bè", "🛒 Mua xu"],
        ]
        markup = ReplyKeyboardMarkup(reply_keyboard, resize_keyboard=True)
        # ==========================================

        await query.message.reply_text(
            "✅ Bạn đã tham gia đầy đủ các kênh!\n\nChọn chức năng bên dưới 👇",
            reply_markup=markup,
        )

    else:
        links = "\n".join(f"➡️ {c['link']}" for c in not_joined)
        keyboard = [[InlineKeyboardButton("🔍 Xác minh lại", callback_data="check_join")]]
        await query.edit_message_text(
            f"⚠️ Bạn chưa tham gia đủ các kênh bắt buộc:\n{links}\n\n➡️ Sau khi tham gia, bấm nút để kiểm tra lại ✅",
            reply_markup=InlineKeyboardMarkup(keyboard),
        )


# -------------------------
# Menu callbacks
# -------------------------
async def menu_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    uid = query.from_user.id
    await query.answer()

    joined_all, _ = await check_joined_all(context, uid)
    if not joined_all:
        await query.message.reply_text("🚫 Bạn cần tham gia đầy đủ kênh trước khi dùng chức năng này.")
        return

    if query.data == "tk":
        xu = user_xu.get(uid, 0)
        await query.message.reply_text(f"💰 Số xu của bạn: {xu} xu")

    elif query.data == "nhapcode":
        await query.message.reply_text("💬 Gõ lệnh: /nhapcode <MÃ_CODE> để nạp xu.")

    elif query.data == "invite":
        invite_link = f"https://t.me/{(await context.bot.get_me()).username}?start={uid}"
        text = (
            "🎁 *Chương trình mời bạn bè nhận thưởng:*\n\n"
            "📨 Gửi link dưới đây cho bạn bè của bạn:\n"
            f"{invite_link}\n\n"
            "✅ Khi bạn bè bấm link này và bắt đầu bot, *bạn sẽ nhận được 5 xu!*\n"
            "💡 Người được mời cũng nhận được 5 xu chào mừng!"
        )
        await query.message.reply_text(text, parse_mode="Markdown")

    elif query.data == "taocode":
        if uid != ADMIN_ID:
            await query.message.reply_text("🚫 Bạn không có quyền tạo code.")
            return
        await query.message.reply_text("💬 Gõ: /taocode <xu> <lượt> để tạo code mới (ví dụ: /taocode 50 10).")

    elif query.data == "muaxu":
        text = (
            "💎 *Bảng giá xu hiện tại:*\n\n"
            "• 75 xu – 20.000đ\n"
            "• 150 xu – 40.000đ\n"
            "• 235 xu – 70.000đ\n"
            "• 500 xu – 130.000đ\n\n"
            "👉 Bấm nút dưới để liên hệ admin mua xu:"
        )
        admin_username = ADMIN_USERNAME if ADMIN_USERNAME.startswith("@") else f"@{ADMIN_USERNAME}"
        keyboard = [[InlineKeyboardButton("💳 Liên hệ mua xu", url=f"https://t.me/{admin_username.replace('@','')}")]]
        await query.message.reply_text(text, parse_mode="Markdown", reply_markup=InlineKeyboardMarkup(keyboard))

# -------------------------
# Commands
# -------------------------
async def tk_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    joined_all, _ = await check_joined_all(context, uid)
    if not joined_all:
        await show_join_verify(update, context)
        return
    xu = user_xu.get(uid, 0)
    await update.message.reply_text(f"💰 Bạn có {xu} xu")

async def nhapcode_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    joined_all, _ = await check_joined_all(context, uid)
    if not joined_all:
        await show_join_verify(update, context)
        return
    if len(context.args) < 1:
        await update.message.reply_text("⚠️ Dùng: /nhapcode <MÃ_CODE>")
        return
    code = context.args[0].upper()
    if code not in codes:
        await update.message.reply_text("❌ Mã code không hợp lệ.")
        return
    data = codes[code]
    if data.get("uses", 0) <= 0:
        await update.message.reply_text("❌ Mã code đã hết lượt.")
        return
    if uid in used_codes and code in used_codes[uid]:
        await update.message.reply_text("⚠️ Bạn đã nhập mã này rồi.")
        return
    user_xu[uid] = user_xu.get(uid, 0) + int(data.get("value", 0))
    data["uses"] = int(data.get("uses", 0)) - 1
    used_codes.setdefault(uid, []).append(code)
    save_data(user_xu, codes, used_codes, referrals)
    await update.message.reply_text(f"✅ Nhập code thành công! Bạn nhận được {data['value']} xu.")

async def taocode_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.effective_user.id
    if uid != ADMIN_ID:
        await update.message.reply_text("🚫 Bạn không có quyền dùng lệnh này.")
        return
    if len(context.args) < 2:
        await update.message.reply_text("⚠️ Dùng: /taocode <xu> <lượt>")
        return
    try:
        value = int(context.args[0])
        uses = int(context.args[1])
    except Exception:
        await update.message.reply_text("❌ Giá trị không hợp lệ.")
        return
    code = gen_code()
    codes[code] = {"value": value, "uses": uses}
    save_data(user_xu, codes, used_codes, referrals)
    await update.message.reply_text(f"🎁 Tạo code thành công:\n`{code}`\n💰 {value} xu – {uses} lượt", parse_mode="Markdown")

# -------------------------
# Handle message
# -------------------------
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.effective_message
    uid = msg.from_user.id
    text = msg.text or msg.caption or ""
    text = text.strip()

    # === Xử lý các nút bàn phím gửi nhanh ===
    if text == "👤 Tài khoản":
        xu = user_xu.get(uid, 0)
        await msg.reply_text(f"💰 Bạn hiện có {xu} xu")
        return

    elif text == "💰 Nhập code":
        await msg.reply_text("💬 Gõ lệnh: /nhapcode <MÃ_CODE> để nạp xu.")
        return

    elif text == "🎁 Mời bạn bè":
        invite_link = f"https://t.me/{(await context.bot.get_me()).username}?start={uid}"
        await msg.reply_text(
            f"🎁 Mời bạn bè nhận thưởng:\n\n"
            f"📨 Gửi link này cho bạn bè:\n{invite_link}\n\n"
            f"✅ Khi họ tham gia bot, cả hai sẽ nhận 5 xu!"
        )
        return

    elif text == "🛒 Mua xu":
        admin_username = ADMIN_USERNAME.replace("@", "")
        await msg.reply_text(
            f"💳 Liên hệ mua xu: https://t.me/{admin_username}\n\n"
            "75 xu – 20.000đ\n150 xu – 40.000đ\n235 xu – 70.000đ\n500 xu – 130.000đ"
        )
        return
    md5hex = extract_md5_hex(text)
    used = "Tìm MD5 trong nội dung" if md5hex else "Tạo MD5 từ chuỗi"
    if not md5hex:
        md5hex = md5_of_text(text)

    tai, xiu, choice = compute_percent_from_md5_hex(md5hex)
    emoji = "🔴" if choice == "TÀI" else "🟢"

    user_xu[uid] = user_xu.get(uid, 0) - 1
    save_data(user_xu, codes, used_codes, referrals)

    reply = (
        f"{used}:\n`{md5hex}`\n\n"
        f"🎯 Kết quả: {emoji} {choice}\n"
        f"Tài: {tai}% | Xỉu: {xiu}%\n\n"
        f"💰 Còn lại: {user_xu.get(uid,0)} xu"
    )
    await msg.reply_text(reply, parse_mode="Markdown")

# -------------------------
# Main
# -------------------------
def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("tk", tk_command))
    app.add_handler(CommandHandler("nhapcode", nhapcode_command))
    app.add_handler(CommandHandler("taocode", taocode_command))
    app.add_handler(CallbackQueryHandler(verify_callback, pattern="^check_join$"))
    app.add_handler(CallbackQueryHandler(menu_callback, pattern="^(tk|nhapcode|taocode|muaxu|invite)$"))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    logger.info("🤖 Bot đang chạy...")
    app.run_polling()

if __name__ == "__main__":
    main()
