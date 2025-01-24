import telebot
from telebot import types
import requests

API_TOKEN = '8035522825:AAG8aD5qiaHTHKO4AtcnkzhR5_ypdSQ2Lr4'
API_KEY = '9fd04a03afb63b4726ae97947e3c41d59c677c90fde7db4cf6aa439967b4f259'
ADMIN_ID = 7567921223

bot = telebot.TeleBot(API_TOKEN)

# Боттун абалын сактоо
bot_active = True
payment_details = "Банк: О! Деньги\nНомер: 503201021"

# Функция пополнения
def deposit_to_1win(user_id, amount):
    url = "https://api.1win.win/v1/client/deposit"
    headers = {"X-API-KEY": API_KEY}
    data = {"userId": user_id, "amount": amount}
    response = requests.post(url, headers=headers, json=data)
    return response.json()

# Функция вывода
def withdraw_from_1win(user_id, code):
    url = "https://api.1win.win/v1/client/withdrawal"
    headers = {"X-API-KEY": API_KEY}
    data = {"userId": user_id, "code": code}
    response = requests.post(url, headers=headers, json=data)
    return response.json()

# Старт командасы
@bot.message_handler(commands=['start'])
def start_handler(message):
    global bot_active
    if not bot_active:
        bot.send_message(message.chat.id, "Бот учурда өчүрүлгөн.")
        return

    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("Пополнение", callback_data="popolnenie"))
    markup.add(types.InlineKeyboardButton("Вывод", callback_data="vyvod"))
    bot.send_message(
        message.chat.id,
        "Привет, \n\nДобро пожаловать в GYM KG 🫶\n\nМоментальное пополнение и вывод средств",
        reply_markup=markup,
    )

# Пополнение процессу
@bot.callback_query_handler(func=lambda call: call.data == "popolnenie")
def popolnenie_handler(call):
    bot.send_message(call.message.chat.id, "Введите ID вашего счета 1win:")
    bot.register_next_step_handler(call.message, get_popolnenie_id)

def get_popolnenie_id(message):
    user_id = message.text
    bot.send_message(message.chat.id, "Введите сумму пополнения в KGS (Минимум 100):")
    bot.register_next_step_handler(message, get_popolnenie_amount, user_id)

def get_popolnenie_amount(message, user_id):
    global payment_details
    amount = float(message.text)
    if amount < 100:
        bot.send_message(message.chat.id, "Минимальная сумма 100 KGS.")
        return
    bot.send_message(
        message.chat.id,
        f"Реквизиты для оплаты:\n\n{payment_details}\n\nПосле оплаты отправьте скриншот чека.",
    )
    bot.register_next_step_handler(message, get_popolnenie_check, user_id, amount)

def get_popolnenie_check(message, user_id, amount):
    check = message.photo[-1].file_id if message.photo else None
    if not check:
        bot.send_message(message.chat.id, "Скриншот обязателен! Попробуйте снова.")
        return

    markup = types.InlineKeyboardMarkup()
    markup.add(
        types.InlineKeyboardButton("✅ Одобрить", callback_data=f"approve_{user_id}_{amount}"),
        types.InlineKeyboardButton("❌ Отклонить", callback_data=f"decline_{message.chat.id}"),
    )
    bot.send_photo(
        ADMIN_ID,
        check,
        caption=f"Новая заявка на пополнение!\n\nID: {user_id}\nСумма: {amount} KGS\nПользователь: @{message.from_user.username}",
        reply_markup=markup,
    )
    bot.send_message(message.chat.id, "Ваша заявка отправлена на проверку.")

@bot.callback_query_handler(func=lambda call: call.data.startswith("approve_"))
def approve_popolnenie(call):
    try:
        _, user_id, amount = call.data.split("_")
        result = deposit_to_1win(user_id, float(amount))
        if result.get("errorMessage"):
            bot.send_message(call.message.chat.id, f"Ошибка: {result['errorMessage']}")
        else:
            bot.send_message(
                call.message.chat.id,
                f"Пополнение на сумму {amount} KGS для ID {user_id} успешно выполнено."
            )
            bot.send_message(
                int(user_id),
                f"Ваш запрос на пополнение на сумму {amount} KGS успешно выполнен!"
            )
    except Exception as e:
        bot.send_message(call.message.chat.id, f"Произошла ошибка: {str(e)}")

@bot.callback_query_handler(func=lambda call: call.data.startswith("decline_"))
def decline_popolnenie(call):
    try:
        _, user_chat_id = call.data.split("_")
        bot.send_message(
            int(user_chat_id),
            "Ваша заявка на пополнение отклонена. Пожалуйста, свяжитесь с поддержкой."
        )
        bot.send_message(call.message.chat.id, "Вы отклонили заявку на пополнение.")
    except Exception as e:
        bot.send_message(call.message.chat.id, f"Произошла ошибка: {str(e)}")

# Вывод процессу
@bot.callback_query_handler(func=lambda call: call.data == "vyvod")
def vyvod_handler(call):
    bot.send_message(call.message.chat.id, "Введите ID вашего счета 1win для вывода:")
    bot.register_next_step_handler(call.message, get_vyvod_id)

def get_vyvod_id(message):
    user_id = message.text
    bot.send_message(message.chat.id, "Введите сумму вывода (от 500 до 30 000 KGS):")
    bot.register_next_step_handler(message, get_vyvod_amount, user_id)

def get_vyvod_amount(message, user_id):
    amount = float(message.text)
    if amount < 500 or amount > 30000:
        bot.send_message(message.chat.id, "Сумма должна быть от 500 до 30 000 KGS.")
        return
    bot.send_message(message.chat.id, "Введите код для вывода (6-значный код):")
    bot.register_next_step_handler(message, get_vyvod_code, user_id, amount)

def get_vyvod_code(message, user_id, amount):
    code = message.text
    markup = types.InlineKeyboardMarkup()
    markup.add(
        types.InlineKeyboardButton("✅ Одобрить", callback_data=f"confirm_{user_id}_{amount}_{code}"),
        types.InlineKeyboardButton("❌ Отклонить", callback_data=f"reject_{message.chat.id}"),
    )
    bot.send_message(
        ADMIN_ID,
        f"Новая заявка на вывод средств!\n\nID: {user_id}\nСумма: {amount} KGS\nКод: {code}",
        reply_markup=markup,
    )
    bot.send_message(message.chat.id, "Заявка на вывод отправлена на проверку.")

# Одобрение вывода
@bot.callback_query_handler(func=lambda call: call.data.startswith("confirm_"))
def confirm_withdraw(call):
    try:
        _, user_id, amount, code = call.data.split("_")
        result = withdraw_from_1win(user_id, code)
        if result.get("errorMessage"):
            bot.send_message(call.message.chat.id, f"Ошибка: {result['errorMessage']}")
        else:
            bot.send_message(
                call.message.chat.id,
                f"Вывод на сумму {amount} KGS для ID {user_id} успешно выполнен."
            )
            bot.send_message(
                int(user_id),
                f"Ваш запрос на вывод {amount} KGS успешно выполнен!"
            )
    except Exception as e:
        bot.send_message(call.message.chat.id, f"Произошла ошибка: {str(e)}")

# Отклонение вывода
@bot.callback_query_handler(func=lambda call: call.data.startswith("reject_"))
def reject_withdraw(call):
    try:
        _, user_chat_id = call.data.split("_")
        bot.send_message(
            int(user_chat_id),
            "Ваш запрос на вывод отклонён. Пожалуйста, свяжитесь с поддержкой."
        )
        bot.send_message(call.message.chat.id, "Вы отклонили запрос на вывод.")
    except Exception as e:
        bot.send_message(call.message.chat.id, f"Произошла ошибка: {str(e)}")

# Админ командасы
@bot.message_handler(commands=['admin'])
def admin_handler(message):
    if message.chat.id != ADMIN_ID:
        bot.send_message(message.chat.id, "У вас нет доступа к этой команде.")
        return

    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("Изменить реквизиты", callback_data="change_rekvezity"))
    markup.add(types.InlineKeyboardButton("Отключить бота", callback_data="disable_bot"))
    markup.add(types.InlineKeyboardButton("Включить бота", callback_data="enable_bot"))
    bot.send_message(message.chat.id, "Админ панель:", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data == "change_rekvezity")
def change_rekvezity_handler(call):
    bot.send_message(call.message.chat.id, "Отправьте новые реквизиты:")
    bot.register_next_step_handler(call.message, save_rekvezity)

def save_rekvezity(message):
    global payment_details
    payment_details = message.text
    bot.send_message(message.chat.id, "Реквизиты успешно обновлены!")

@bot.callback_query_handler(func=lambda call: call.data == "disable_bot")
def disable_bot_handler(call):
    global bot_active
    bot_active = False
    bot.send_message(call.message.chat.id, "Бот успешно отключён.")

@bot.callback_query_handler(func=lambda call: call.data == "enable_bot")
def enable_bot_handler(call):
    global bot_active
    bot_active = True
    bot.send_message(call.message.chat.id, "Бот успешно включён.")

# Ботту иштетүү
bot.polling()

