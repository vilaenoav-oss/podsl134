import sqlite3
import html

from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
)
from telegram.constants import ParseMode
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ContextTypes,
    ConversationHandler,
    filters,
)


# =========================================================
# НАСТРОЙКИ
# =========================================================

BOT_TOKEN = "8893013516:AAEpj1SQc3wsK85IYQrYf6zLe1xSmf53dAs"

ADMIN_IDS = {
    7457259687,
    7335476150,
    1800354129,
}

CHANNEL_ID = "@podsl134"
CHANNEL_LINK = "https://t.me/podsl134"
CHANNEL_NAME = "От Подслушано"

DB_NAME = "news.db"


# =========================================================
# БАЗА ДАННЫХ
# =========================================================

def init_db():
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS news (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            username TEXT,
            text TEXT,
            media_type TEXT,
            file_id TEXT,
            hide_author INTEGER DEFAULT 0,
            status TEXT DEFAULT 'pending',
            author_chosen INTEGER DEFAULT 0
        )
    """)

    # Проверяем, есть ли author_chosen в старой базе
    cursor.execute("PRAGMA table_info(news)")
    columns = [column[1] for column in cursor.fetchall()]

    if "author_chosen" not in columns:
        cursor.execute("""
            ALTER TABLE news
            ADD COLUMN author_chosen INTEGER DEFAULT 0
        """)

    # Таблица заблокированных пользователей
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS banned_users (
            user_id INTEGER PRIMARY KEY
        )
    """)

    connection.commit()
    connection.close()


# =========================================================
# РАБОТА С НОВОСТЯМИ
# =========================================================

def create_news(user_id, username, text, media_type=None, file_id=None):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute("""
        INSERT INTO news (
            user_id,
            username,
            text,
            media_type,
            file_id,
            hide_author,
            status,
            author_chosen
        )
        VALUES (?, ?, ?, ?, ?, 0, 'pending', 0)
    """, (
        user_id,
        username,
        text,
        media_type,
        file_id
    ))

    news_id = cursor.lastrowid

    connection.commit()
    connection.close()

    return news_id


def get_news(news_id):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute("""
        SELECT
            id,
            user_id,
            username,
            text,
            media_type,
            file_id,
            hide_author,
            status,
            author_chosen
        FROM news
        WHERE id = ?
    """, (news_id,))

    result = cursor.fetchone()

    connection.close()

    return result


def update_news_text(news_id, new_text):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "UPDATE news SET text = ? WHERE id = ?",
        (new_text, news_id)
    )

    connection.commit()
    connection.close()


def toggle_hide_author(news_id):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "SELECT hide_author FROM news WHERE id = ?",
        (news_id,)
    )

    result = cursor.fetchone()

    if result is None:
        connection.close()
        return False

    current_value = result[0]
    new_value = 0 if current_value else 1

    cursor.execute(
        "UPDATE news SET hide_author = ? WHERE id = ?",
        (new_value, news_id)
    )

    connection.commit()
    connection.close()

    return bool(new_value)


def choose_author(news_id, hide_author):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute("""
        UPDATE news
        SET
            hide_author = ?,
            author_chosen = 1
        WHERE id = ?
          AND author_chosen = 0
    """, (
        1 if hide_author else 0,
        news_id
    ))

    changed = cursor.rowcount > 0

    connection.commit()
    connection.close()

    return changed


def set_status(news_id, status):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "UPDATE news SET status = ? WHERE id = ?",
        (status, news_id)
    )

    connection.commit()
    connection.close()


# =========================================================
# БАН / РАЗБАН
# =========================================================

def is_user_banned(user_id):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "SELECT 1 FROM banned_users WHERE user_id = ?",
        (user_id,)
    )

    result = cursor.fetchone()

    connection.close()

    return result is not None


def ban_user(user_id):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "INSERT OR IGNORE INTO banned_users (user_id) VALUES (?)",
        (user_id,)
    )

    connection.commit()
    connection.close()


def unban_user(user_id):
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "DELETE FROM banned_users WHERE user_id = ?",
        (user_id,)
    )

    removed = cursor.rowcount > 0

    connection.commit()
    connection.close()

    return removed


def get_banned_users():
    connection = sqlite3.connect(DB_NAME)
    cursor = connection.cursor()

    cursor.execute(
        "SELECT user_id FROM banned_users ORDER BY user_id"
    )

    users = cursor.fetchall()

    connection.close()

    return [user[0] for user in users]


# =========================================================
# КЛАВИАТУРЫ
# =========================================================

def user_author_keyboard(news_id):
    keyboard = [[
        InlineKeyboardButton(
            "🕵️ Анонимно",
            callback_data=f"user_anonymous:{news_id}"
        ),
        InlineKeyboardButton(
            "👤 Автор виден",
            callback_data=f"user_visible:{news_id}"
        )
    ]]

    return InlineKeyboardMarkup(keyboard)


def admin_keyboard(news_id, hide_author=False):
    author_button = (
        "👤 Показать автора"
        if hide_author
        else
        "👤 Скрыть автора"
    )

    keyboard = [
        [
            InlineKeyboardButton(
                "✏️ Редактировать",
                callback_data=f"edit:{news_id}"
            )
        ],

        [
            InlineKeyboardButton(
                author_button,
                callback_data=f"author:{news_id}"
            )
        ],

        [
            InlineKeyboardButton(
                "🚫 Заблокировать пользователя",
                callback_data=f"ban:{news_id}"
            )
        ],

        [
            InlineKeyboardButton(
                "✅ Опубликовать",
                callback_data=f"publish:{news_id}"
            ),
            InlineKeyboardButton(
                "❌ Отклонить",
                callback_data=f"reject:{news_id}"
            )
        ]
    ]

    return InlineKeyboardMarkup(keyboard)


# =========================================================
# ИНФОРМАЦИЯ ДЛЯ АДМИНА
# =========================================================

def admin_caption(news):
    (
        news_id,
        user_id,
        username,
        text,
        media_type,
        file_id,
        hide_author,
        status,
        author_chosen
    ) = news

    if username:
        author = f"@{html.escape(username)}"
    else:
        author = f"ID: {user_id}"

    text = text or "Без текста"

    author_status = (
        "скрыт"
        if hide_author
        else
        "виден"
    )

    return (
        "📰 <b>НОВАЯ НОВОСТЬ</b>\n\n"
        f"<b>Автор:</b> {author}\n"
        f"<b>Telegram ID:</b> {user_id}\n"
        f"<b>Автор в публикации:</b> {author_status}\n\n"
        f"<b>Текст:</b>\n"
        f"{html.escape(text)}"
    )


# =========================================================
# START
# =========================================================

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):

    user_id = update.effective_user.id

    if is_user_banned(user_id):
        await update.message.reply_text(
            "🚫 Вы не можете отправлять материалы в этот бот."
        )
        return

    await update.message.reply_text(
        "👋 Привет!\n\n"
        "Отправь сюда свой текст, информацию, фото или видео.\n\n"
        "После отправки выбери, указывать ли автора.\n\n"
        "После проверки администраторами материал "
        "может появиться в нашем канале."
    )


# =========================================================
# ПОЛУЧЕНИЕ НОВОСТИ
# =========================================================

async def receive_news(update: Update, context: ContextTypes.DEFAULT_TYPE):

    message = update.message
    user = update.effective_user

    # Проверяем бан
    if is_user_banned(user.id):
        await message.reply_text(
            "🚫 Вы не можете отправлять материалы в этот бот."
        )
        return

    # Текст
    if message.text:

        news_id = create_news(
            user.id,
            user.username,
            message.text,
            None,
            None
        )

    # Фото
    elif message.photo:

        photo = message.photo[-1]

        news_id = create_news(
            user.id,
            user.username,
            message.caption,
            "photo",
            photo.file_id
        )

    # Видео
    elif message.video:

        news_id = create_news(
            user.id,
            user.username,
            message.caption,
            "video",
            message.video.file_id
        )

    # Документ
    elif message.document:

        news_id = create_news(
            user.id,
            user.username,
            message.caption,
            "document",
            message.document.file_id
        )

    # Аудио
    elif message.audio:

        news_id = create_news(
            user.id,
            user.username,
            message.caption,
            "audio",
            message.audio.file_id
        )

    # Голосовое
    elif message.voice:

        news_id = create_news(
            user.id,
            user.username,
            None,
            "voice",
            message.voice.file_id
        )

    else:

        await message.reply_text(
            "❌ Этот тип сообщения пока не поддерживается."
        )

        return

    await message.reply_text(
        "👤 Как указать автора публикации?",
        reply_markup=user_author_keyboard(news_id)
    )


# =========================================================
# АНОНИМНО
# =========================================================

async def user_anonymous(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    user_id = query.from_user.id

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:
        await query.answer(
            "Материал не найден.",
            show_alert=True
        )
        return

    (
        news_id_db,
        owner_id,
        username,
        text,
        media_type,
        file_id,
        hide_author,
        status,
        author_chosen
    ) = news

    if user_id != owner_id:
        await query.answer(
            "Это не ваш материал.",
            show_alert=True
        )
        return

    if author_chosen:
        await query.answer(
            "Вы уже выбрали вариант.",
            show_alert=True
        )
        return

    changed = choose_author(
        news_id,
        hide_author=True
    )

    if not changed:
        await query.answer(
            "Выбор уже сделан.",
            show_alert=True
        )
        return

    try:

        await query.edit_message_text(
            "🕵️ Автор скрыт.\n\n"
            "✅ Материал отправлен на проверку администрации."
        )

    except Exception:
        pass

    await send_news_to_admins(
        context,
        news_id
    )


# =========================================================
# АВТОР ВИДЕН
# =========================================================

async def user_visible(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    user_id = query.from_user.id

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:
        await query.answer(
            "Материал не найден.",
            show_alert=True
        )
        return

    (
        news_id_db,
        owner_id,
        username,
        text,
        media_type,
        file_id,
        hide_author,
        status,
        author_chosen
    ) = news

    if user_id != owner_id:
        await query.answer(
            "Это не ваш материал.",
            show_alert=True
        )
        return

    if author_chosen:
        await query.answer(
            "Вы уже выбрали вариант.",
            show_alert=True
        )
        return

    changed = choose_author(
        news_id,
        hide_author=False
    )

    if not changed:
        await query.answer(
            "Выбор уже сделан.",
            show_alert=True
        )
        return

    try:

        await query.edit_message_text(
            "👤 Автор будет указан.\n\n"
            "✅ Материал отправлен на проверку администрации."
        )

    except Exception:
        pass

    await send_news_to_admins(
        context,
        news_id
    )


# =========================================================
# ОТПРАВКА АДМИНАМ
# =========================================================

async def send_news_to_admins(
    context: ContextTypes.DEFAULT_TYPE,
    news_id
):

    news = get_news(news_id)

    if not news:
        return

    caption = admin_caption(news)

    keyboard = admin_keyboard(
        news_id,
        hide_author=bool(news[6])
    )

    media_type = news[4]
    file_id = news[5]

    for admin_id in ADMIN_IDS:

        try:

            if media_type == "photo":

                await context.bot.send_photo(
                    chat_id=admin_id,
                    photo=file_id,
                    caption=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

            elif media_type == "video":

                await context.bot.send_video(
                    chat_id=admin_id,
                    video=file_id,
                    caption=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

            elif media_type == "document":

                await context.bot.send_document(
                    chat_id=admin_id,
                    document=file_id,
                    caption=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

            elif media_type == "audio":

                await context.bot.send_audio(
                    chat_id=admin_id,
                    audio=file_id,
                    caption=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

            elif media_type == "voice":

                await context.bot.send_voice(
                    chat_id=admin_id,
                    voice=file_id,
                    caption=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

            else:

                await context.bot.send_message(
                    chat_id=admin_id,
                    text=caption,
                    parse_mode=ParseMode.HTML,
                    reply_markup=keyboard
                )

        except Exception as error:

            print(
                f"Ошибка отправки админу "
                f"{admin_id}: {error}"
            )


# =========================================================
# РЕДАКТИРОВАНИЕ
# =========================================================

EDITING = 1

editing_news = {}


async def edit_news(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    admin_id = query.from_user.id

    if admin_id not in ADMIN_IDS:

        await query.answer(
            "У вас нет доступа.",
            show_alert=True
        )

        return ConversationHandler.END

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:

        await query.answer(
            "Новость не найдена.",
            show_alert=True
        )

        return ConversationHandler.END

    editing_news[admin_id] = news_id

    await query.message.reply_text(
        "✏️ <b>Режим редактирования</b>\n\n"
        "Отправьте мне новый текст новости.\n\n"
        "Медиафайл при этом сохранится.",
        parse_mode=ParseMode.HTML
    )

    return EDITING


async def save_edited_text(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    admin_id = update.effective_user.id

    if admin_id not in editing_news:
        return ConversationHandler.END

    news_id = editing_news[admin_id]

    new_text = update.message.text

    if not new_text:

        await update.message.reply_text(
            "❌ Отправьте именно текст."
        )

        return EDITING

    update_news_text(
        news_id,
        new_text
    )

    del editing_news[admin_id]

    await update.message.reply_text(
        "✅ Текст новости изменён."
    )

    return ConversationHandler.END


# =========================================================
# СКРЫТЬ / ПОКАЗАТЬ АВТОРА
# =========================================================

async def author_button(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    admin_id = query.from_user.id

    if admin_id not in ADMIN_IDS:

        await query.answer(
            "У вас нет доступа.",
            show_alert=True
        )

        return

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:

        await query.answer(
            "Новость не найдена.",
            show_alert=True
        )

        return

    new_hide_status = toggle_hide_author(
        news_id
    )

    news = get_news(news_id)

    keyboard = admin_keyboard(
        news_id,
        hide_author=new_hide_status
    )

    caption = admin_caption(news)

    try:

        await query.edit_message_caption(
            caption=caption,
            parse_mode=ParseMode.HTML,
            reply_markup=keyboard
        )

    except Exception:

        try:

            await query.edit_message_text(
                text=caption,
                parse_mode=ParseMode.HTML,
                reply_markup=keyboard
            )

        except Exception as error:

            print(error)


# =========================================================
# БАН ПОЛЬЗОВАТЕЛЯ
# =========================================================

async def ban_user_button(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    admin_id = query.from_user.id

    if admin_id not in ADMIN_IDS:

        await query.answer(
            "У вас нет доступа.",
            show_alert=True
        )

        return

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:

        await query.answer(
            "Новость не найдена.",
            show_alert=True
        )

        return

    user_id = news[1]
    username = news[2]

    # Нельзя заблокировать администратора
    if user_id in ADMIN_IDS:

        await query.answer(
            "Нельзя заблокировать администратора.",
            show_alert=True
        )

        return

    ban_user(user_id)

    if username:
        name = f"@{username}"
    else:
        name = f"ID {user_id}"

    await query.answer(
        "Пользователь заблокирован.",
        show_alert=True
    )

    await query.message.reply_text(
        "🚫 <b>Пользователь заблокирован.</b>\n\n"
        f"Пользователь: {html.escape(name)}\n"
        f"Telegram ID: <code>{user_id}</code>",
        parse_mode=ParseMode.HTML
    )

async def ban_command(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):
    admin_id = update.effective_user.id

    if admin_id not in ADMIN_IDS:
        await update.message.reply_text(
            "❌ У вас нет доступа."
        )
        return

    if not context.args:
        await update.message.reply_text(
            "Использование:\n\n"
            "/ban 123456789"
        )
        return

    try:
        user_id = int(context.args[0])
    except ValueError:
        await update.message.reply_text(
            "❌ ID должен состоять только из цифр."
        )
        return

    if user_id in ADMIN_IDS:
        await update.message.reply_text(
            "❌ Нельзя заблокировать администратора."
        )
        return

    ban_user(user_id)

    await update.message.reply_text(
        f"🚫 Пользователь {user_id} заблокирован."
    )

# =========================================================
# РАЗБАН
# =========================================================

async def unban_command(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    admin_id = update.effective_user.id

    if admin_id not in ADMIN_IDS:

        await update.message.reply_text(
            "❌ У вас нет доступа."
        )

        return

    if not context.args:

        await update.message.reply_text(
            "Использование:\n\n"
            "/unban 123456789"
        )

        return

    try:

        user_id = int(
            context.args[0]
        )

    except ValueError:

        await update.message.reply_text(
            "❌ ID должен состоять только из цифр."
        )

        return

    removed = unban_user(user_id)

    if removed:

        await update.message.reply_text(
            f"✅ Пользователь {user_id} разблокирован."
        )

    else:

        await update.message.reply_text(
            f"ℹ️ Пользователь {user_id} "
            f"не был заблокирован."
        )


# =========================================================
# СПИСОК ЗАБЛОКИРОВАННЫХ
# =========================================================

async def banned_command(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    admin_id = update.effective_user.id

    if admin_id not in ADMIN_IDS:

        await update.message.reply_text(
            "❌ У вас нет доступа."
        )

        return

    users = get_banned_users()

    if not users:

        await update.message.reply_text(
            "📋 Заблокированных пользователей нет."
        )

        return

    text = "🚫 <b>Заблокированные пользователи:</b>\n\n"

    for user_id in users:

        text += f"• <code>{user_id}</code>\n"

    await update.message.reply_text(
        text,
        parse_mode=ParseMode.HTML
    )


# =========================================================
# ПУБЛИКАЦИЯ
# =========================================================

async def publish_news(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    admin_id = query.from_user.id

    if admin_id not in ADMIN_IDS:

        await query.answer(
            "У вас нет доступа.",
            show_alert=True
        )

        return

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:

        await query.answer(
            "Новость не найдена.",
            show_alert=True
        )

        return

    (
        news_id,
        user_id,
        username,
        text,
        media_type,
        file_id,
        hide_author,
        status,
        author_chosen
    ) = news

    if status != "pending":

        await query.answer(
            "Эта новость уже обработана.",
            show_alert=True
        )

        return

    text = text or ""

    safe_text = html.escape(text)

    if hide_author:

        author_block = ""

    else:

        if username:
            author = f"@{html.escape(username)}"
        else:
            author = f"ID: {user_id}"

        author_block = (
            f"\n\n<b>Автор:</b> {author}"
        )

    post = (
        f"<b>{html.escape(CHANNEL_NAME)}</b>\n\n"
        f"<blockquote>{safe_text}</blockquote>"
        f"{author_block}\n\n"
        f'<a href="{CHANNEL_LINK}">'
        f"Подписаться на Подслушано 134"
        f"</a>"
    )

    try:

        if media_type == "photo":

            await context.bot.send_photo(
                chat_id=CHANNEL_ID,
                photo=file_id,
                caption=post,
                parse_mode=ParseMode.HTML
            )

        elif media_type == "video":

            await context.bot.send_video(
                chat_id=CHANNEL_ID,
                video=file_id,
                caption=post,
                parse_mode=ParseMode.HTML
            )

        elif media_type == "document":

            await context.bot.send_document(
                chat_id=CHANNEL_ID,
                document=file_id,
                caption=post,
                parse_mode=ParseMode.HTML
            )

        elif media_type == "audio":

            await context.bot.send_audio(
                chat_id=CHANNEL_ID,
                audio=file_id,
                caption=post,
                parse_mode=ParseMode.HTML
            )

        elif media_type == "voice":

            await context.bot.send_voice(
                chat_id=CHANNEL_ID,
                voice=file_id,
                parse_mode=ParseMode.HTML
            )

        else:

            await context.bot.send_message(
                chat_id=CHANNEL_ID,
                text=post,
                parse_mode=ParseMode.HTML
            )

        set_status(
            news_id,
            "published"
        )

        await query.edit_message_reply_markup(
            reply_markup=None
        )

        try:

            await query.message.reply_text(
                "✅ <b>Новость опубликована.</b>",
                parse_mode=ParseMode.HTML
            )

        except Exception:
            pass

    except Exception as error:

        print(
            f"Ошибка публикации: {error}"
        )

        await query.message.reply_text(
            f"❌ Ошибка публикации:\n\n{error}"
        )


# =========================================================
# ОТКЛОНЕНИЕ
# =========================================================

async def reject_news(
    update: Update,
    context: ContextTypes.DEFAULT_TYPE
):

    query = update.callback_query

    await query.answer()

    admin_id = query.from_user.id

    if admin_id not in ADMIN_IDS:

        await query.answer(
            "У вас нет доступа.",
            show_alert=True
        )

        return

    news_id = int(
        query.data.split(":")[1]
    )

    news = get_news(news_id)

    if not news:

        await query.answer(
            "Новость не найдена.",
            show_alert=True
        )

        return

    if news[7] != "pending":

        await query.answer(
            "Эта новость уже обработана.",
            show_alert=True
        )

        return

    set_status(
        news_id,
        "rejected"
    )

    try:

        await query.edit_message_reply_markup(
            reply_markup=None
        )

        await query.message.reply_text(
            "❌ Новость отклонена."
        )

    except Exception:
        pass


# =========================================================
# ОШИБКИ
# =========================================================

async def error_handler(
    update,
    context
):

    print(
        "Произошла ошибка:",
        context.error
    )


# =========================================================
# ЗАПУСК
# =========================================================

def main():

    init_db()

    application = (
        Application
        .builder()
        .token(BOT_TOKEN)
        .build()
    )

    # /start
    application.add_handler(
        CommandHandler(
            "start",
            start
        )
    )

    # /ban
    application.add_handler(
    CommandHandler(
        "ban",
        ban_command
    )
)

    # /unban
    application.add_handler(
        CommandHandler(
            "unban",
            unban_command
        )
    )

    # /banned
    application.add_handler(
        CommandHandler(
            "banned",
            banned_command
        )
    )

    # Редактирование
    edit_conversation = ConversationHandler(

        entry_points=[
            CallbackQueryHandler(
                edit_news,
                pattern=r"^edit:\d+$"
            )
        ],

        states={

            EDITING: [
                MessageHandler(
                    filters.TEXT & ~filters.COMMAND,
                    save_edited_text
                )
            ]

        },

        fallbacks=[]
    )

    application.add_handler(
        edit_conversation
    )

    # Анонимно
    application.add_handler(
        CallbackQueryHandler(
            user_anonymous,
            pattern=r"^user_anonymous:\d+$"
        )
    )

    # Автор виден
    application.add_handler(
        CallbackQueryHandler(
            user_visible,
            pattern=r"^user_visible:\d+$"
        )
    )

    # Скрыть / показать автора
    application.add_handler(
        CallbackQueryHandler(
            author_button,
            pattern=r"^author:\d+$"
        )
    )

    # Бан
    application.add_handler(
        CallbackQueryHandler(
            ban_user_button,
            pattern=r"^ban:\d+$"
        )
    )

    # Публикация
    application.add_handler(
        CallbackQueryHandler(
            publish_news,
            pattern=r"^publish:\d+$"
        )
    )

    # Отклонение
    application.add_handler(
        CallbackQueryHandler(
            reject_news,
            pattern=r"^reject:\d+$"
        )
    )

    # Получение сообщений
    application.add_handler(
        MessageHandler(

            (
                filters.TEXT
                | filters.PHOTO
                | filters.VIDEO
                | filters.Document.ALL
                | filters.AUDIO
                | filters.VOICE
            )
            & ~filters.COMMAND,

            receive_news
        )
    )

    application.add_error_handler(
        error_handler
    )

    print("==============================")
    print("БОТ ЗАПУЩЕН")
    print("==============================")

    application.run_polling()


if __name__ == "__main__":
    main()
