import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton, Message, InputMediaPhoto

TOKEN = "8115780530:AAEKEU0paRAq0IweApQBQwgOA6d9F9xZsWs"
ADMIN_ID = 7756926130  # Админдин Telegram ID'си

bot = telebot.TeleBot(TOKEN)

user_data = {}  # Колдонуучу маалыматтарын убактылуу сактоо

# Старт басканда
@bot.message_handler(commands=['start'])
def start(message):
    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("Пополнить", callback_data="deposit"),
        InlineKeyboardButton("Вывести", callback_data="withdraw")
    )
    bot.send_message(message.chat.id, "Добро пожаловать в XMeNT\n\nМоментальные пополнения", reply_markup=markup)

# Пополнение басканда
@bot.callback_query_handler(func=lambda call: call.data == "deposit")
def deposit(call):
    bot.send_message(call.message.chat.id, "Введите ID вашего счета Melbet:")
    bot.register_next_step_handler(call.message, get_melbet_id)

def get_melbet_id(message):
    user_data[message.chat.id] = {"melbet_id": message.text}
    bot.send_message(message.chat.id, "Введите сумму пополнения KGS\nМинимум: 50\nМаксимум: 50000")
    bot.register_next_step_handler(message, get_deposit_amount)

def get_deposit_amount(message):
    try:
        amount = int(message.text)
        if 50 <= amount <= 50000:
            user_data[message.chat.id]["amount"] = amount
            bot.send_message(message.chat.id, f"Сумма к оплате: {amount} сом\n\n"
                                              "Реквизиты 1:\nM BANK\nНомер телефона: +996554350666\nФИО: ЖУМАБЕК А\n"
                                              "Номер карты: Только по номер телефона\n\n"
                                              "Реквизиты 2:\nО! Деньги\nНомер телефона: +996508150785\nФИО: Динара Б\n"
                                              "Номер карты: Только по номер телефона\n\n"
                                              "После оплаты отправьте скриншот чека")
            bot.register_next_step_handler(message, get_payment_screenshot)
        else:
            bot.send_message(message.chat.id, "Сумма должна быть от 50 до 50000. Попробуйте еще раз.")
            bot.register_next_step_handler(message, get_deposit_amount)
    except ValueError:
        bot.send_message(message.chat.id, "Пожалуйста, введите число.")
        bot.register_next_step_handler(message, get_deposit_amount)

def get_payment_screenshot(message):
    if message.photo:
        user_data[message.chat.id]["screenshot"] = message.photo[-1].file_id
        data = user_data[message.chat.id]
        
        admin_text = (f"🔔 Новая заявка на пополнение:\n\n"
                      f"👤 Пользователь: @{message.from_user.username}\n"
                      f"💰 Сумма: {data['amount']} сом\n"
                      f"🎲 Melbet ID: {data['melbet_id']}\n"
                      f"🆔 Telegram ID: {message.chat.id}")

        markup = InlineKeyboardMarkup()
        markup.add(
            InlineKeyboardButton("✅ Одобрить", callback_data=f"approve_{message.chat.id}"),
            InlineKeyboardButton("❌ Отменить", callback_data=f"decline_{message.chat.id}")
        )

        bot.send_photo(ADMIN_ID, photo=data["screenshot"], caption=admin_text, reply_markup=markup)
        bot.send_message(message.chat.id, "✅ Ваша заявка отправлена администратору!")
    else:
        bot.send_message(message.chat.id, "Пожалуйста, отправьте скриншот чека.")
        bot.register_next_step_handler(message, get_payment_screenshot)

# Вывод средств
@bot.callback_query_handler(func=lambda call: call.data == "withdraw")
def withdraw(call):
    bot.send_message(call.message.chat.id, "Введите ID (Номер Счёта) MelBeT для вывода средств!")
    bot.register_next_step_handler(call.message, get_withdraw_id)

def get_withdraw_id(message):
    user_data[message.chat.id] = {"melbet_id": message.text}
    bot.send_message(message.chat.id, "Введите Банк и Реквизиты (например, МБАНК +996000000000)")
    bot.register_next_step_handler(message, get_withdraw_details)

def get_withdraw_details(message):
    user_data[message.chat.id]["bank_details"] = message.text
    bot.send_message(message.chat.id, "Как получить код:\n\n"
                                      "1. Заходим на сайт букмекера\n"
                                      "2. Вывести со счета\n"
                                      "3. Выбираем наличные\n"
                                      "4. Пишем сумму\n"
                                      "5. Город: СарыКолот\n"
                                      "6. Улица: XMENT\n\n"
                                      "После получения кода введите его здесь.")
    bot.register_next_step_handler(message, get_withdraw_code)

def get_withdraw_code(message):
    user_data[message.chat.id]["withdraw_code"] = message.text
    data = user_data[message.chat.id]

    admin_text = (f"🔔 Новая заявка на вывод:\n\n"
                  f"👤 Пользователь: @{message.from_user.username}\n"
                  f"🎲 Melbet ID: {data['melbet_id']}\n"
                  f"🏦 Реквизиты: {data['bank_details']}\n"
                  f"🔑 Код: {data['withdraw_code']}\n"
                  f"🆔 Telegram ID: {message.chat.id}")

    markup = InlineKeyboardMarkup()
    markup.add(
        InlineKeyboardButton("✅ Вывести", callback_data=f"withdraw_approve_{message.chat.id}"),
        InlineKeyboardButton("❌ Отклонить", callback_data=f"withdraw_decline_{message.chat.id}")
    )

    bot.send_message(ADMIN_ID, admin_text, reply_markup=markup)
    bot.send_message(message.chat.id, "✅ Ваша заявка отправлена администратору!")

# Админдин жооптору
@bot.callback_query_handler(func=lambda call: call.data.startswith("approve_"))
def approve_request(call):
    user_id = int(call.data.split("_")[1])
    bot.send_message(user_id, "✅ Ваша заявка одобрена. Деньги зачислены!")

@bot.callback_query_handler(func=lambda call: call.data.startswith("decline_"))
def decline_request(call):
    user_id = int(call.data.split("_")[1])
    bot.send_message(user_id, "❌ Ваша заявка отклонена. Свяжитесь с поддержкой.")

@bot.callback_query_handler(func=lambda call: call.data.startswith("withdraw_approve_"))
def approve_withdraw(call):
    user_id = int(call.data.split("_")[2])
    bot.send_message(user_id, "✅ Ваш вывод средств успешно выполнен!")

@bot.callback_query_handler(func=lambda call: call.data.startswith("withdraw_decline_"))
def decline_withdraw(call):
    user_id = int(call.data.split("_")[2])
    bot.send_message(user_id, "❌ Ваш запрос на вывод средств отклонен. Свяжитесь с поддержкой.")

bot.polling()