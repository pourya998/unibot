import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    CallbackQueryHandler,
    MessageHandler,
    filters,
    ContextTypes,
)

# تنظیمات اولیه
TOKEN = "7825941081:AAFHxMjtzOtocAhy6bKfphwiLknzkYU6p1c"  # توکن ربات
ADMIN_ID = 6929024145  # آیدی عددی ادمین
ADMIN_PASSWORD = "2029"  # رمز عبور مدیریت

# دیتابیس ساده برای ذخیره اطلاعات
data = {
    "educational_materials": {},  # جزوات آموزشی
    "student_section": {
        "sample_questions": {},
        "sample_projects": {},
        "sample_presentations": {},
    },
    "news": [],  # اخبار و اطلاعیه‌ها (می‌تواند شامل متن، عکس، فیلم و غیره باشد)
    "course_contents": {},  # محتواهای آموزشی
    "course_outlines": {},  # سرفصل دروس
    "social_media": {},  # کانال‌های شبکه‌های اجتماعی
}

# تنظیمات لاگ‌گیری
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO
)
logger = logging.getLogger(__name__)

# منوی اصلی
def main_menu():
    keyboard = [
        [InlineKeyboardButton("📚 جزوات آموزشی", callback_data="educational_materials")],
        [InlineKeyboardButton("🎓 بخش دانشجویی", callback_data="student_section")],
        [InlineKeyboardButton("📰 اخبار و اطلاعیه‌ها", callback_data="news")],
        [InlineKeyboardButton("📖 محتواهای آموزشی", callback_data="course_contents")],
        [InlineKeyboardButton("📜 سرفصل دروس", callback_data="course_outlines")],
        [InlineKeyboardButton("🌐 شبکه‌های اجتماعی", callback_data="social_media")],
        [InlineKeyboardButton("⚙️ پنل مدیریت", callback_data="admin_panel")],
    ]
    return InlineKeyboardMarkup(keyboard)

# منوی مدیریت
def admin_menu():
    keyboard = [
        [InlineKeyboardButton("📤 بارگذاری فایل", callback_data="upload_file")],
        [InlineKeyboardButton("🗑️ حذف فایل", callback_data="delete_file")],
        [InlineKeyboardButton("📝 افزودن خبر", callback_data="add_news")],
        [InlineKeyboardButton("🔗 افزودن لینک شبکه اجتماعی", callback_data="add_social_media")],
        [InlineKeyboardButton("🔙 بازگشت", callback_data="back_main")],
    ]
    return InlineKeyboardMarkup(keyboard)

# منوی انتخاب بخش برای بارگذاری فایل
def upload_section_menu():
    keyboard = [
        [InlineKeyboardButton("📚 جزوات آموزشی", callback_data="upload_educational_materials")],
        [InlineKeyboardButton("🎓 نمونه سوالات", callback_data="upload_sample_questions")],
        [InlineKeyboardButton("📂 نمونه پروژه‌ها", callback_data="upload_sample_projects")],
        [InlineKeyboardButton("📊 نمونه ارائه‌ها", callback_data="upload_sample_presentations")],
        [InlineKeyboardButton("📖 محتواهای آموزشی", callback_data="upload_course_contents")],
        [InlineKeyboardButton("📜 سرفصل دروس", callback_data="upload_course_outlines")],
        [InlineKeyboardButton("🔙 بازگشت", callback_data="back_admin")],
    ]
    return InlineKeyboardMarkup(keyboard)

# منوی بخش دانشجویی
def student_menu():
    keyboard = [
        [InlineKeyboardButton("📝 نمونه سوالات", callback_data="sample_questions")],
        [InlineKeyboardButton("📂 نمونه پروژه‌ها", callback_data="sample_projects")],
        [InlineKeyboardButton("📊 نمونه ارائه‌ها", callback_data="sample_presentations")],
        [InlineKeyboardButton("🔙 بازگشت", callback_data="back_main")],
    ]
    return InlineKeyboardMarkup(keyboard)

# تابع شروع
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(
        "🏫 به ربات آموزشی خوش آمدید! لطفا یک گزینه را انتخاب کنید:", reply_markup=main_menu()
    )

# مدیریت دکمه‌ها
async def button(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    query = update.callback_query
    await query.answer()

    if query.data == "educational_materials":
        await show_files(query, "educational_materials", "📚 جزوات آموزشی")
    elif query.data == "student_section":
        await query.message.reply_text("🎓 بخش دانشجویی:", reply_markup=student_menu())
    elif query.data == "news":
        await show_news(query)
    elif query.data == "course_contents":
        await show_files(query, "course_contents", "📖 محتواهای آموزشی")
    elif query.data == "course_outlines":
        await show_files(query, "course_outlines", "📜 سرفصل دروس")
    elif query.data == "social_media":
        await show_social_media(query)
    elif query.data == "admin_panel":
        await query.message.reply_text("🔑 لطفا رمز عبور را وارد کنید:")
        context.user_data["waiting_for_password"] = True
    elif query.data == "upload_file":
        await query.message.reply_text("📂 لطفا بخش مورد نظر را انتخاب کنید:", reply_markup=upload_section_menu())
    elif query.data.startswith("upload_"):
        section = query.data.replace("upload_", "")
        context.user_data["upload_section"] = section
        await query.message.reply_text(f"📤 لطفا فایل خود را برای بخش **{section}** ارسال کنید و نام آن را مشخص کنید.")
        context.user_data["waiting_for_file"] = True
    elif query.data == "delete_file":
        await query.message.reply_text("🗑️ لطفا نام فایل مورد نظر برای حذف را وارد کنید.")
        context.user_data["waiting_for_delete"] = True
    elif query.data == "add_news":
        await query.message.reply_text("📝 لطفا خبر جدید را به همراه فایل (اختیاری) ارسال کنید:")
        context.user_data["waiting_for_news"] = True
    elif query.data == "add_social_media":
        await query.message.reply_text("🔗 لطفا نام و لینک شبکه اجتماعی را به فرمت `نام:لینک` وارد کنید:")
        context.user_data["waiting_for_social_media"] = True
    elif query.data == "back_main":
        await query.message.reply_text("🏠 بازگشت به منوی اصلی", reply_markup=main_menu())
    elif query.data == "back_admin":
        await query.message.reply_text("🔙 بازگشت به پنل مدیریت", reply_markup=admin_menu())
    elif query.data in ["sample_questions", "sample_projects", "sample_presentations"]:
        await show_files(query, f"student_section/{query.data}", f"🎓 {query.data.replace('_', ' ')}")

# نمایش فایل‌ها
async def show_files(query, section, title):
    files = data.get(section, {})
    if files:
        keyboard = [
            [InlineKeyboardButton(f"📄 {file_name}", callback_data=f"download_{section}_{file_name}")]
            for file_name in files.keys()
        ]
        keyboard.append([InlineKeyboardButton("🔙 بازگشت", callback_data="back_main")])
        await query.message.reply_text(f"{title}:", reply_markup=InlineKeyboardMarkup(keyboard))
    else:
        await query.message.reply_text(f"❌ هیچ فایلی در بخش {title} وجود ندارد.")

# نمایش اخبار
async def show_news(query):
    news = data.get("news", [])
    if news:
        for item in news:
            if isinstance(item, dict):  # اگر خبر شامل فایل باشد
                await query.message.reply_text(f"📰 {item['text']}")
                if item.get("file_id"):
                    await query.message.reply_document(document=item["file_id"])
            else:  # اگر خبر فقط متن باشد
                await query.message.reply_text(f"📰 {item}")
    else:
        await query.message.reply_text("❌ هیچ خبری وجود ندارد.")

# نمایش شبکه‌های اجتماعی
async def show_social_media(query):
    social_media = data.get("social_media", {})
    if social_media:
        await query.message.reply_text(
            "🌐 شبکه‌های اجتماعی:\n" + "\n".join([f"{k}: {v}" for k, v in social_media.items()])
        )
    else:
        await query.message.reply_text("❌ هیچ لینکی وجود ندارد.")

# مدیریت ارسال فایل‌ها
async def handle_document(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if "waiting_for_file" in context.user_data:
        document = update.message.document or update.message.photo[-1] or update.message.video
        file_id = document.file_id
        file_name = update.message.caption or document.file_name or "file"
        section = context.user_data.get("upload_section", "educational_materials")
        data[section][file_name] = file_id
        del context.user_data["waiting_for_file"]
        del context.user_data["upload_section"]
        await update.message.reply_text(f"✅ فایل **{file_name}** در بخش **{section}** اضافه شد.", reply_markup=main_menu())
    elif "waiting_for_news" in context.user_data:
        file_id = None
        if update.message.document or update.message.photo or update.message.video:
            document = update.message.document or update.message.photo[-1] or update.message.video
            file_id = document.file_id
        text = update.message.caption or update.message.text
        data["news"].append({"text": text, "file_id": file_id})
        del context.user_data["waiting_for_news"]
        await update.message.reply_text(f"✅ خبر **{text}** اضافه شد.", reply_markup=main_menu())

# مدیریت دریافت رمز عبور و ورود به مدیریت
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    user_id = update.message.from_user.id
    text = update.message.text

    if context.user_data.get("waiting_for_password", False):
        if text == ADMIN_PASSWORD and user_id == ADMIN_ID:
            await update.message.reply_text("✅ ورود موفق! به پنل مدیریت خوش آمدید.", reply_markup=admin_menu())
        else:
            await update.message.reply_text("❌ رمز عبور اشتباه است.")
        context.user_data["waiting_for_password"] = False
    elif context.user_data.get("waiting_for_delete", False):
        file_name = text
        section = "educational_materials"  # یا هر بخش دیگری که نیاز دارید
        if file_name in data[section]:
            del data[section][file_name]
            await update.message.reply_text(f"✅ فایل **{file_name}** حذف شد.", reply_markup=main_menu())
        else:
            await update.message.reply_text("❌ فایل مورد نظر یافت نشد.")
        context.user_data["waiting_for_delete"] = False
    elif context.user_data.get("waiting_for_social_media", False):
        try:
            name, link = text.split(":", 1)
            data["social_media"][name.strip()] = link.strip()
            await update.message.reply_text(f"✅ لینک **{name}** اضافه شد.", reply_markup=main_menu())
        except ValueError:
            await update.message.reply_text("❌ فرمت ورودی نامعتبر است. لطفا از فرمت `نام:لینک` استفاده کنید.")
        context.user_data["waiting_for_social_media"] = False

# تابع اصلی
def main() -> None:
    application = Application.builder().token(TOKEN).build()
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CallbackQueryHandler(button))
    application.add_handler(MessageHandler(filters.Document.ALL | filters.PHOTO | filters.VIDEO, handle_document))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    logger.info("🚀 ربات در حال اجرا است...")
    application.run_polling()

if __name__ == "__main__":
    main()
