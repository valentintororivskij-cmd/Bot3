import telebot
import random
from telebot import types

TOKEN = "8740967424:AAHgoRaJdGnLfU4z34yBOysLaMesNSbcXmU"
bot = telebot.TeleBot(TOKEN)

players = {}


def get_user(uid, name="Гравець"):
    if uid not in players:
        players[uid] = {
            "name": name,
            "balance": 777,
            "exp": 0,
            "wins": 0,
            "spins": 0,
            "bet": 100
        }
    return players[uid]


def get_level(exp):
    return (exp // 200) + 1 if exp >= 0 else 0


def main_kb():
    kb = types.ReplyKeyboardMarkup(resize_keyboard=True)
    kb.add("🎰 СЛОТИ (HARD)", "🎲 КУБИК")
    kb.add("👤 ПРОФІЛЬ", "🎁 БОНУС")
    kb.add("📊 ТОП", "ℹ️ ШАНСИ")
    return kb


def slot_inline(bet):
    kb = types.InlineKeyboardMarkup()
    kb.add(types.InlineKeyboardButton(f"🎰 КРУТИТИ ({bet}$)", callback_data="spin"))
    kb.add(types.InlineKeyboardButton("💰 СТАВКА", callback_data="bet"))
    kb.add(types.InlineKeyboardButton("❌ ВИЙТИ", callback_data="exit"))
    return kb


def spin_logic():
    icons = ["🍎","🍎","🍋","🍋","🍒","🍒","🍇","🍉","💎","💎","7️⃣","💀","💀","💩","💣"]
    res = [random.choice(icons) for _ in range(3)]

    if res[0] == res[1] == res[2]:
        if res[0] == "7️⃣":
            coef = 50
        elif res[0] == "💎":
            coef = 20
        elif res[0] == "💀":
            coef = -10
        elif res[0] == "💣":
            coef = -20
        else:
            coef = 10
    elif len(set(res)) < 3:
        coef = 1.5 if "💀" not in res and "💣" not in res else 0
    else:
        coef = 0

    return res, coef


@bot.message_handler(commands=['start'])
def start(message):
    get_user(message.from_user.id, message.from_user.first_name)
    bot.send_message(
        message.chat.id,
        "🦾 HARDCORE CASINO V2\n\nТут виживають тільки щасливчики 😈",
        reply_markup=main_kb()
    )


@bot.message_handler(func=lambda m: m.text == "👤 ПРОФІЛЬ")
def profile(message):
    p = get_user(message.from_user.id)
    lv = get_level(p["exp"])

    text = (
        f"👤 {p['name']}\n"
        f"💰 Баланс: {p['balance']}$\n"
        f"🆙 Рівень: {lv}\n"
        f"⭐ XP: {p['exp']}\n"
        f"🎰 Ігор: {p['spins']}\n"
        f"🏆 Перемог: {p['wins']}\n"
        f"💵 Ставка: {p['bet']}$"
    )

    bot.send_message(message.chat.id, text)


@bot.message_handler(func=lambda m: m.text == "🎁 БОНУС")
def bonus(message):
    p = get_user(message.from_user.id)

    if random.randint(1, 10) > 2:
        amount = random.randint(50, 150)
        p["balance"] += amount
        bot.send_message(message.chat.id, f"💰 +{amount}$")
    else:
        bot.send_message(message.chat.id, "❌ Нема бонусу")


@bot.message_handler(func=lambda m: m.text == "📊 ТОП")
def top(message):
    top_list = sorted(players.values(), key=lambda x: x["balance"], reverse=True)[:5]

    text = "📊 ТОП ГРАВЦІВ\n\n"
    for i, u in enumerate(top_list, 1):
        text += f"{i}. {u['name']} — {u['balance']}$\n"

    bot.send_message(message.chat.id, text)


@bot.message_handler(func=lambda m: m.text == "ℹ️ ШАНСИ")
def chances(message):
    text = (
        "ℹ️ ШАНСИ В ІГРАХ\n\n"
        "🎰 СЛОТИ (HARD):\n"
        "• 3 однакових — великий виграш\n"
        "• 2 однакових — маленький виграш\n"
        "• 💀 / 💣 — штраф\n"
        "• шанс виграти ≈ 15-25%\n\n"
        "🎲 КУБИК:\n"
        "• виграш ≈ 42%\n"
        "• нічия ≈ 16%\n"
        "• програш ≈ 42%\n\n"
        "💡 Бот має невелику перевагу 😈"
    )

    bot.send_message(message.chat.id, text)


@bot.message_handler(func=lambda m: m.text == "🎰 СЛОТИ (HARD)")
def slots(message):
    p = get_user(message.from_user.id)

    bot.send_message(
        message.chat.id,
        f"🎰 СЛОТИ\nСтавка: {p['bet']}$",
        reply_markup=slot_inline(p["bet"])
    )


@bot.message_handler(func=lambda m: m.text == "🎲 КУБИК")
def dice_game(message):
    p = get_user(message.from_user.id)

    if p["balance"] < p["bet"]:
        bot.send_message(message.chat.id, "❌ Недостатньо грошей")
        return

    player = random.randint(1, 6)
    bot_roll = random.randint(1, 6)

    p["spins"] += 1

    text = f"🎲 Ти: {player}\n🤖 Бот: {bot_roll}\n\n"

    if player > bot_roll:
        p["balance"] += p["bet"]
        p["wins"] += 1
        p["exp"] += 20
        text += f"🔥 +{p['bet']}$"
    elif player < bot_roll:
        p["balance"] -= p["bet"]
        p["exp"] = max(0, p["exp"] - 10)
        text += f"💀 -{p['bet']}$"
    else:
        text += "🤝 Нічия"

    p["balance"] = max(0, p["balance"])
    text += f"\n\n💰 {p['balance']}$"

    bot.send_message(message.chat.id, text)


@bot.callback_query_handler(func=lambda call: True)
def callback(call):
    p = get_user(call.from_user.id)

    if call.data == "spin":
        if p["balance"] < p["bet"]:
            bot.answer_callback_query(call.id, "❌ Нема грошей!", show_alert=True)
            return

        p["balance"] -= p["bet"]
        p["spins"] += 1

        res, coef = spin_logic()
        win = int(p["bet"] * coef)
        p["balance"] += win
        p["balance"] = max(0, p["balance"])

        if coef > 1:
            p["wins"] += 1
            p["exp"] += 25
            result = f"🔥 +{win}$"
        elif coef < 0:
            p["exp"] = max(0, p["exp"] - 50)
            result = f"☢️ {win}$"
        else:
            p["exp"] += 2
            result = "💀"

        text = (
            f"🎰 | {res[0]} {res[1]} {res[2]} |\n\n"
            f"{result}\n"
            f"💰 {p['balance']}$"
        )

        bot.edit_message_text(
            text,
            call.message.chat.id,
            call.message.message_id,
            reply_markup=slot_inline(p["bet"])
        )

    elif call.data == "bet":
        bets = [100, 500, 2500, 5000]

        if p["bet"] not in bets:
            p["bet"] = bets[0]
        else:
            i = bets.index(p["bet"])
            p["bet"] = bets[(i + 1) % len(bets)]

        bot.edit_message_reply_markup(
            call.message.chat.id,
            call.message.message_id,
            reply_markup=slot_inline(p["bet"])
        )

    elif call.data == "exit":
        bot.delete_message(call.message.chat.id, call.message.message_id)
        bot.send_message(call.message.chat.id, "🏠 Меню", reply_markup=main_kb())


bot.infinity_polling()
wQFW