import logging
import time
import random
import asyncio
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    CommandHandler,
    MessageHandler,
    filters
)

# إعداد السجلات
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

TOKEN = "8543985827:AAFzH2U1vA5MVpvClqzL-ycUWmFG-5aT_Ng"
DEV_USERNAME = "Mllk313"
DEV_NAME = "Ħ"

# قاعدة بيانات مؤقتة لتخزين بيانات الأعضاء
db = {}

# قائمة الكلمات المحرمة والممنوعة
BAD_WORDS = [
    "كس", "كسمك", "كسختك", "كسعرضك", "خرب", "زنديق", "منيوج", "منيوك", 
    "عير", "كسيس", "خنيث", "مبعبص", "بعبوص", "شرموط", "شرموطه", "عرص", 
    "عاهرة", "كحبه", "استنياج", "زب", "زبزوب", "بربوك", "بربوء", "اه"
]

# نصوص فلسفية
PHILOSOPHY_QUOTES = [
    "فلسفة نيتشه: ما لا يقتلني يجعلني أقوى، والإنسان حبل متوتر بين الحيوان والإنسان الفائق.",
    "فلسفة الوجودية: الوجود يسبق الماهية، أنت حر إذن أنت مسؤول عن اختيارك.",
    "فلسفة العبث: الحياة بلا معنى مسبق، وعليك أن تخلق معناك الخاص وسط هذا العبث.",
    "فلسفة شوبنهاور: الحياة رقصة بين الألم والملل، والرغبة هي أصل الشقاء."
]

# نصوص دينية وأحاديث/حكم
RELIGIOUS_TEXTS = [
    "قال رسول الله صلى الله عليه وسلم: (إنما الأعمال بالنيات، وإنما لكل امرئ ما نوى).",
    "تذكر دائماً قوله تعالى: (ألا بذكر الله تطمئن القلوب).",
    "احرص على صلاتك فهي عماد دينك ونور دربك.",
    "قال الإمام علي عليه السلام: (الكلمة أسريرة، فإذا خرجت ملكتك)."
]

def init_user(user_id):
    if user_id not in db:
        db[user_id] = {
            "warnings": 0,
            "mutes": 0,
            "game_points": 0,
            "chat_points": 0,
            "last_messages": [],
            "name": "",
            "username": ""
        }

# أمر /start الترحيب والقوانين المختصرة
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    init_user(user.id)
    db[user.id]["name"] = user.first_name
    db[user.id]["username"] = user.username or "لا يوجد"
    
    await update.message.reply_text(
        f"أهلاً بك يا {user.first_name} في بوت حماية وتسلية المجموعات الأقوى! 🛡️\n\n"
        f"🔹 بوت متكامل لحماية الكروب، التفاعل، الفلسفة، الدين، والألعاب.\n"
        f"🔹 اكتب /help لمعرفة الأوامر المتاحة."
    )

# أمر القوانين
async def rules(update: Update, context: ContextTypes.DEFAULT_TYPE):
    rules_text = (
        "📜 **قوانين الكروب الأساسية:**\n\n"
        "1️⃣ ممنوع السب والشتم نهائياً.\n"
        "2️⃣ ممنوع الترويج لغير كروب أو نشر الروابط.\n"
        "3️⃣ ممنوع الزحف والمضايقات.\n"
        "4️⃣ يمنع نشر الأمور الطائفية أو المخالفة للدين.\n"
        "⚠️ مخالفة هذه القوانين تعرضك للإنذار أو الكتم التلقائي!"
    )
    await update.message.reply_text(rules_text, parse_mode="Markdown")

# أمر معلومات المطور
async def developer_info(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        f"👨‍💻 **معلومات المطور الأساسي:**\n\n"
        f"• الاسم: {DEV_NAME}\n"
        f"• المعرف: @{DEV_USERNAME}\n"
        f"• نبذة: مبرمج ومطور هذا البوت الذكي لحماية المجموعات."
    )

# مراقبة الرسائل للكلمات الممنوعة، السبام، والتفاعل
async def monitor_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not update.message or not update.message.text:
        return
    
    user = update.effective_user
    chat = update.effective_chat
    text = update.message.text
    user_id = user.id
    
    init_user(user_id)
    db[user_id]["name"] = user.first_name
    db[user_id]["username"] = user.username or "لا يوجد"
    db[user_id]["chat_points"] += 1

    # 1. فحص الكلمات الممنوعة
    for word in BAD_WORDS:
        if word in text:
            try:
                await update.message.delete()
            except:
                pass
            
            db[user_id]["warnings"] += 1
            warn_count = db[user_id]["warnings"]
            
            if warn_count >= 3:
                db[user_id]["warnings"] = 0
                db[user_id]["mutes"] += 1
                try:
                    await chat.restrict_member(
                        user_id,
                        permissions={
                            "can_send_messages": False,
                            "can_send_media_messages": False,
                            "can_send_other_messages": False
                        },
                        until_date=int(time.time()) + 300
                    )
                    await chat.send_message(
                        f"🚫 العضو {user.mention_html()} وصل إلى 3 إنذارات بسبب الألفاظ الممنوعة وتم كتمه لمدة 5 دقائق!",
                        parse_mode="HTML"
                    )
                except Exception as e:
                    await chat.send_message(f"عذراً، لا أملك صلاحية كتم العضو! الخطأ: {e}")
            else:
                await chat.send_message(
                    f"⚠️ تنبيه ({warn_count}/3) يا {user.mention_html()}، ممنوع استخدام هذه الألفاظ!",
                    parse_mode="HTML"
                )
            return

    # 2. فحص السبام
    current_time = time.time()
    timestamps = db[user_id]["last_messages"]
    timestamps.append(current_time)
    timestamps = [t for t in timestamps if current_time - t < 3]
    db[user_id]["last_messages"] = timestamps

    if len(timestamps) >= 5:
        db[user_id]["last_messages"] = []
        db[user_id]["warnings"] += 1
        try:
            await update.message.delete()
        except:
            pass
        await chat.send_message(
            f"⚠️ {user.mention_html()}، تم إعطاؤك إنذار بسبب التكرار المزعج (السبام)!",
            parse_mode="HTML"
        )
        return

    # 3. الرد الذكي على اسم "ايسكانور"
    if "ايسكانور" in text or (update.message.reply_to_message and update.message.reply_to_message.from_user.id == context.bot.id):
        responses = [
            "عاشت ايدك يا بطل، وياك أسطورة الحماية ايسكانور 🦁",
            "أنا هنا أراقب الكروب وأحميه من كل العابثين!",
            "امرني عزيزي، شنو محتاج؟",
            "موجود طال عمرك، الكروب بأمان تامة معي."
        ]
        await update.message.reply_text(random.choice(responses))

# أمر الكشف عن العضو
async def inspect_user(update: Update, context: ContextTypes.DEFAULT_TYPE):
    message = update.message
    target_user = None
    
    if message.reply_to_message:
        target_user = message.reply_to_message.from_user
    elif context.args:
        query = context.args[0].replace("@", "")
        for uid, data in db.items():
            if data["username"].lower() == query.lower():
                target_user = await context.bot.get_chat(uid)
                break
                
    if not target_user:
        await message.reply_text("❌ استخدم الأمر بالرد على رسالة العضو أو كتابة المعرف بجانب الأمر، مثلاً: `/c @username`")
        return

    uid = target_user.id
    init_user(uid)
    u_data = db[uid]
    
    info_text = (
        f"🔍 **كشف معلومات العضو:**\n\n"
        f"• الاسم: {u_data['name']}\n"
        f"• اليوزر: @{u_data['username']}\n"
        f"• الأيدي: `{uid}`\n"
        f"• نقاط الألعاب: {u_data['game_points']}\n"
        f"• نقاط التفاعل: {u_data['chat_points']}\n"
        f"• عدد الإنذارات: {u_data['warnings']}\n"
        f"• مرات الكتم: {u_data['mutes']}"
    )
    await message.reply_text(info_text, parse_mode="Markdown")

# أمر الفلسفة (تم تعديل الأمر إلى philosophy)
async def philosophy_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    quote = random.choice(PHILOSOPHY_QUOTES)
    await update.message.reply_text(f"🧠 **محطة الفلسفة:**\n\n{quote}")

# أمر الدين (تم تعديل الأمر إلى religion)
async def religion_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = random.choice(RELIGIOUS_TEXTS)
    await update.message.reply_text(f"📖 **إضاءة دينية:**\n\n{text}")

# أمر التوب (تم تعديل الأمر إلى top)
async def top_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not db:
        await update.message.reply_text("📊 لا توجد بيانات كافية حتى الآن في لوحة الصدارة.")
        return

    sorted_game = sorted(db.items(), key=lambda x: x[1]['game_points'], reverse=True)[:5]
    sorted_chat = sorted(db.items(), key=lambda x: x[1]['chat_points'], reverse=True)[:5]

    game_text = "🏆 **توب الاحتراف (الألعاب):**\n"
    for idx, (uid, data) in enumerate(sorted_game, 1):
        game_text += f"{idx}. {data['name']} - النقاط: {data['game_points']}\n"

    chat_text = "\n🔥 **توب التفاعل (نشاط الكروب):**\n"
    for idx, (uid, data) in enumerate(sorted_chat, 1):
        chat_text += f"{idx}. {data['name']} - التفاعل: {data['chat_points']}\n"

    await update.message.reply_text(game_text + chat_text)

# أوامر الرتب والمشرفين (تم تعديل الأمر إلى admins)
async def admins_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👑 **رتب وإدارة الكروب:**\n\n"
        "• المالك الأساسي وحامي الكروب.\n"
        "• المشرفون / الادمنية المسؤولون عن المراقبة.\n"
        "• تاك للرتبة: استخدم الإشارة للتواصل مع الإدارة."
    )

# لعبة خفيفة (تم تعديل الأمر إلى game)
async def play_game(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    init_user(user.id)
    
    start_time = time.time()
    db[user.id]["game_points"] += 10
    elapsed = round(time.time() - start_time, 2)
    
    await update.message.reply_text(f"🎮 بدأت لعبة التخمين يا {user.first_name}!\n✨ مبروك الفوز! تم إضافة 10 نقاط لسجل احترافك في زمن قدره {elapsed} ثانية.")

def main():
    app = ApplicationBuilder().token(TOKEN).build()

    # تسجيل الأوامر (تمت مراجعتها لتكون باللغة الإنجليزية حصراً)
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("rules", rules))
    app.add_handler(CommandHandler("motto", developer_info))
    app.add_handler(CommandHandler("dev", developer_info))
    app.add_handler(CommandHandler("c", inspect_user))
    app.add_handler(CommandHandler("philosophy", philosophy_command))
    app.add_handler(CommandHandler("religion", religion_command))
    app.add_handler(CommandHandler("top", top_command))
    app.add_handler(CommandHandler("admins", admins_command))
    app.add_handler(CommandHandler("game", play_game))

    # مراقبة جميع الرسائل
    app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), monitor_messages))

    print("جاري تشغيل بوت ايسكانور بكامل الخصائص...")
    
    while True:
        try:
            app.run_polling(drop_pending_updates=True)
        except Exception as e:
            print(f"حدث خطأ: {e}, جاري إعادة المحاولة خلال 5 ثوانٍ...")
            time.sleep(5)

if __name__ == '__main__':
    main()
