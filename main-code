import asyncio
import os
import sqlite3
import time
from collections import defaultdict, deque

from aiogram import Bot, Dispatcher, F
from aiogram.enums import ChatType
from aiogram.filters import Command
from html import escape
from aiogram.types import (
    Message,
    CallbackQuery,
    InlineKeyboardMarkup,
    InlineKeyboardButton,
)

BOT_TOKEN = os.environ["BOT_TOKEN"]
ADMIN_ID = int(os.environ["ADMIN_ID"])
DB_PATH = os.getenv("DB_PATH", "bot.db")

START_TEXT = (
    "Отправьте своё сообщение, и мы постараемся ответить "
    "в ближайшее время.\n\n"
    "Создано на платформе @AnonCreatorBot"
)

SENT_TEXT = "Ваше сообщение отправлено. Ожидайте ответ."

db = sqlite3.connect(DB_PATH)
db.execute("""
CREATE TABLE IF NOT EXISTS users (
    user_id INTEGER PRIMARY KEY,
    username TEXT,
    first_name TEXT,
    last_name TEXT,
    blocked INTEGER NOT NULL DEFAULT 0
)
""")
db.commit()

dp = Dispatcher()
reply_targets = {}
rate_limits = defaultdict(deque)


def save_user(user):
    db.execute(
        """
        INSERT INTO users (user_id, username, first_name, last_name)
        VALUES (?, ?, ?, ?)
        ON CONFLICT(user_id) DO UPDATE SET
            username = excluded.username,
            first_name = excluded.first_name,
            last_name = excluded.last_name
        """,
        (user.id, user.username, user.first_name, user.last_name),
    )
    db.commit()


def is_blocked(user_id):
    row = db.execute(
        "SELECT blocked FROM users WHERE user_id = ?",
        (user_id,),
    ).fetchone()
    return bool(row and row[0])


def set_blocked(user_id, blocked):
    db.execute(
        "UPDATE users SET blocked = ? WHERE user_id = ?",
        (int(blocked), user_id),
    )
    db.commit()


def admin_keyboard(user_id):
    return InlineKeyboardMarkup(inline_keyboard=[
        [
            InlineKeyboardButton(
                text="💬 Ответить",
                callback_data=f"reply:{user_id}",
            ),
            InlineKeyboardButton(
                text="🚫 Заблокировать",
                callback_data=f"block:{user_id}",
            ),
        ]
    ])


def format_user(user):
    name = " ".join(
        part for part in [user.first_name, user.last_name] if part
    ) or "Не указано"

    username = f"@{user.username}" if user.username else "Нет"

    return (
        "📩 <b>Новое обращение</b>\n\n"
        f"Имя: {escape(name)}\n"
        f"Username: {escape(username)}\n"
        f"Telegram ID: <code>{user.id}</code>"
    )


def check_rate_limit(user_id):
    now = time.monotonic()
    events = rate_limits[user_id]

    while events and now - events[0] > 60:
        events.popleft()

    if len(events) >= 10:
        return False

    events.append(now)
    return True


@dp.message(Command("start"), F.chat.type == ChatType.PRIVATE)
async def start(message: Message):
    if message.from_user.id == ADMIN_ID:
        await message.answer(
            "👋 Панель администратора\n\n"
            "/users — количество пользователей\n"
            "/blocked — заблокированные пользователи\n"
            "/unblock ID — разблокировать пользователя\n"
            "/cancel — отменить ответ"
        )
        return

    save_user(message.from_user)
    await message.answer(START_TEXT)


@dp.message(Command("cancel"), F.chat.type == ChatType.PRIVATE)
async def cancel(message: Message):
    if message.from_user.id != ADMIN_ID:
        return

    reply_targets.pop(ADMIN_ID, None)
    await message.answer("Режим ответа отменён.")


@dp.message(Command("users"), F.chat.type == ChatType.PRIVATE)
async def users_command(message: Message):
    if message.from_user.id != ADMIN_ID:
        return

    count = db.execute(
        "SELECT COUNT(*) FROM users"
    ).fetchone()[0]

    await message.answer(f"Всего пользователей: {count}")


@dp.message(Command("blocked"), F.chat.type == ChatType.PRIVATE)
async def blocked_command(message: Message):
    if message.from_user.id != ADMIN_ID:
        return

    rows = db.execute(
        "SELECT user_id, username FROM users WHERE blocked = 1"
    ).fetchall()

    if not rows:
        await message.answer("Заблокированных пользователей нет.")
        return

    lines = [
        f"{user_id} — @{username}" if username else str(user_id)
        for user_id, username in rows
    ]

    await message.answer("🚫 Заблокированные:\n" + "\n".join(lines))


@dp.message(Command("unblock"), F.chat.type == ChatType.PRIVATE)
async def unblock_command(message: Message):
    if message.from_user.id != ADMIN_ID:
        return

    parts = (message.text or "").split(maxsplit=1)

    if len(parts) != 2 or not parts[1].isdigit():
        await message.answer("Формат команды: /unblock 123456789")
        return

    user_id = int(parts[1])

    exists = db.execute(
        "SELECT user_id FROM users WHERE user_id = ?",
        (user_id,),
    ).fetchone()

    if not exists:
        await message.answer("Пользователь не найден.")
        return

    set_blocked(user_id, False)
    await message.answer(f"Пользователь {user_id} разблокирован.")


@dp.callback_query(F.data.startswith("reply:"))
async def reply_callback(callback: CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer("Нет доступа.", show_alert=True)
        return

    user_id = int(callback.data.split(":")[1])
    reply_targets[ADMIN_ID] = user_id

    await callback.answer()
    await callback.message.answer(
        f"✍️ Напиши сообщение для пользователя {user_id}.\n"
        "Можно отправить текст, фото, видео, документ или другое вложение.\n\n"
        "Для отмены используй /cancel."
    )


@dp.callback_query(F.data.startswith("block:"))
async def block_callback(callback: CallbackQuery):
    if callback.from_user.id != ADMIN_ID:
        await callback.answer("Нет доступа.", show_alert=True)
        return

    user_id = int(callback.data.split(":")[1])

    set_blocked(user_id, True)

    if reply_targets.get(ADMIN_ID) == user_id:
        reply_targets.pop(ADMIN_ID, None)

    await callback.answer("Пользователь заблокирован.")
    await callback.message.answer(
        f"🚫 Пользователь {user_id} заблокирован."
    )


@dp.message(F.chat.type == ChatType.PRIVATE)
async def handle_private_message(message: Message, bot: Bot):
    user = message.from_user

    if not user:
        return

    # Сообщения администратора — ответы пользователям
    if user.id == ADMIN_ID:
        target_id = reply_targets.get(ADMIN_ID)

        if not target_id:
            return

        if is_blocked(target_id):
            reply_targets.pop(ADMIN_ID, None)
            await message.answer(
                "Этот пользователь заблокирован. Ответ не отправлен."
            )
            return

        try:
            await bot.copy_message(
                chat_id=target_id,
                from_chat_id=message.chat.id,
                message_id=message.message_id,
            )
            await message.answer(
                f"✅ Ответ отправлен пользователю {target_id}."
            )
        except Exception:
            await message.answer(
                "❌ Не удалось отправить сообщение. "
                "Возможно, пользователь заблокировал бота."
            )
        finally:
            reply_targets.pop(ADMIN_ID, None)

        return

    # Сообщения обычных пользователей
    save_user(user)

    if is_blocked(user.id):
        await message.answer("Отправка сообщений недоступна.")
        return

    if not check_rate_limit(user.id):
        await message.answer(
            "Слишком много сообщений. Попробуйте немного позже."
        )
        return

    try:
        await bot.send_message(
            chat_id=ADMIN_ID,
            text=format_user(user),
            reply_markup=admin_keyboard(user.id),
        )

        await bot.copy_message(
            chat_id=ADMIN_ID,
            from_chat_id=message.chat.id,
            message_id=message.message_id,
        )

        await message.answer(SENT_TEXT)

    except Exception:
        await message.answer(
            "Не удалось отправить сообщение. Попробуйте позже."
        )


async def main():
    bot = Bot(BOT_TOKEN)

    try:
        await dp.start_polling(bot)
    finally:
        await bot.session.close()
        db.close()


if __name__ == "__main__":
    asyncio.run(main())
