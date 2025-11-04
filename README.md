# san4s
san4s
import sqlite3
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, MessageHandler, filters, CallbackContext, CallbackQueryHandler
import datetime
import matplotlib.pyplot as plt

# --- База данных (sqlite3) ---
conn = sqlite3.connect('misses.db', check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    telegram_id INTEGER UNIQUE,
    role TEXT,        -- admin, curator, monitor
    group_name TEXT,
    curator_id INTEGER
)""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS misses (
    id INTEGER PRIMARY KEY,
    user_id INTEGER,
    date TEXT,
    reason TEXT,
    justified INTEGER  -- 0 or 1
)""")

conn.commit()

# --- Роли ---
ROLES = ['admin', 'curator', 'monitor']


# --- Функции управления пользователями ---
def get_user_role(telegram_id):
    cursor.execute("SELECT role FROM users WHERE telegram_id=?", (telegram_id,))
    res = cursor.fetchone()
    return res[0] if res else None


def is_admin(telegram_id):
    return get_user_role(telegram_id) == 'admin'


def is_curator(telegram_id):
    return get_user_role(telegram_id) == 'curator'


# --- Команды ---

def start(update: Update, context: CallbackContext):
    update.message.reply_text("Привет! Это бот учёта пропусков.\nИспользуй /help для списка команд.")


def help_command(update: Update, context: CallbackContext):
    text = (
        "/misses - просмотр пропусков\n"
        "/report - отправить отчёт\n"
        "/export - экспорт данных\n"
        "/search - поиск пропусков\n"
        "/chart - диаграмма пропусков\n"
    )
    update.message.reply_text(text)


# Просмотр пропусков
def misses_command(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    role = get_user_role(user_id)

    if role is None:
        update.message.reply_text("Вы не зарегистрированы.")
        return

    if role == 'monitor':
        cursor.execute(
            "SELECT date, reason, justified FROM misses WHERE user_id=(SELECT id FROM users WHERE telegram_id=?)",
            (user_id,))
    elif role == 'curator':
        # Куратор видит пропуски своей группы
        cursor.execute("""
            SELECT m.date, m.reason, m.justified, u.telegram_id FROM misses m
            JOIN users u ON m.user_id=u.id
            WHERE u.curator_id=(SELECT id FROM users WHERE telegram_id=?)
        """, (user_id,))
    elif role == 'admin':
        cursor.execute("""
            SELECT m.date, m.reason, m.justified, u.telegram_id FROM misses m
            JOIN users u ON m.user_id=u.id
        """)
    else:
        update.message.reply_text("Нет доступа.")
        return

    rows = cursor.fetchall()
    if not rows:
        update.me

        ssage.reply_text("Пропуски не найдены.")
        return

    text = "Пропуски:\n"
    for row in rows:
        if role == 'monitor':
            date, reason, justified = row
            text += f"{date} - {reason} - {'Уважительно' if justified else 'Неуважительно'}\n"
        else:
            date, reason, justified, user_tid = row
            text += f"Пользователь: {user_tid}, Дата: {date}, {reason}, {'Уважительно' if justified else 'Неуважительно'}\n"
    update.message.reply_text(text)


# Отправка отчёта (пример: отправить администратору)
def report_command(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    role = get_user_role(user_id)
    if role not in ['curator', 'admin']:
        update.message.reply_text("Доступно только кураторам и администраторам.")
        return

    cursor.execute("""
                SELECT u.telegram_id, COUNT(*) FROM misses m
                JOIN users u ON m.user_id=u.id
                WHERE date >= date('now', '-7 days')
                GROUP BY u.telegram_id
            """)
    data = cursor.fetchall()
    report = "Отчёт по пропускам за неделю:\n"
    for user_tid, cnt in data:
        report += f"Пользователь {user_tid}: {cnt} пропуск(ов)\n"

    # Отправка администратору (предполагается один админ)
    cursor.execute("SELECT telegram_id FROM users WHERE role='admin'")
    admin = cursor.fetchone()
    if admin:
        context.bot.send_message(admin[0], report)
        update.message.reply_text("Отчёт отправлен админу.")
    else:
        update.message.reply_text("Администратор не найден.")


# Быстрый поиск пропусков по параметру (напр. по группе)
def search_command(update: Update, context: CallbackContext):
    args = context.args
    if not args:
        update.message.reply_text("Используйте /search <группа>, например /search Группа1")
        return
    group_name = args[0]
    cursor.execute("""
                SELECT u.telegram_id, m.date, m.reason FROM misses m
                JOIN users u ON m.user_id = u.id
                WHERE u.group_name=?
            """, (group_name,))
    data = cursor.fetchall()
    if not data:
        update.message.reply_text("По запросу ничего не найдено.")
        return

    text = f"Пропуски группы {group_name}:\n"
    for user_tid, date, reason in data:
        text += f"{user_tid}: {date} - {reason}\n"
    update.message.reply_text(text)


# Диаграмма пропусков
def chart_command(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    role = get_user_role(user_id)
    if role is None:
        update.message.reply_text("Неизвестный пользователь.")
        return

    # Выгружаем пропуски по дате
    cursor.execute("""
                SELECT date, COUNT(*) FROM misses
                GROUP BY date
            """)
    data = cursor.fetchall()
    if not data:
        update.message.reply_text("Данные отсутствуют.")
        return

    dates = [row[0] for row in data]
    counts = [row[1] for row in data]

    plt.bar(dates, counts)
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.title('Пропуски по датам')
    plt.savefig('chart.png')
    plt.clf()

    with open('chart.png', 'rb') as f:

        update.message.reply_photo(f)

        # Для простоты - API можно организовать через HTTP сервер (например Flask) отдельно

        # --- Запуск бота ---

    def main():
        updater = Updater('ваш_токен_от_BotFather', use_context=True)
        dp = updater.dispatcher

        dp.add_handler(CommandHandler('start', start))
        dp.add_handler(CommandHandler('help', help_command))
        dp.add_handler(CommandHandler('misses', misses_command))
        dp.add_handler(CommandHandler('report', report_command))
        dp.add_handler(CommandHandler('search', search_command))
        dp.add_handler(CommandHandler('chart', chart_command))

        updater.start_polling()
        updater.idle()

    if __name__ == '__main__':
        main()
