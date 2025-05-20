# telegram-bot-job
import telebot from telebot import types import openai import requests from bs4 import BeautifulSoup import json import time import os

تحميل المتغيرات من .env

from dotenv import load_dotenv load_dotenv()

TELEGRAM_TOKEN = os.getenv("7615887131:AAEE5xirTbNL9oT5Fq562ZQpTGwQfzCIa-c") OPENAI_API_KEY = os.getenv("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")

bot = telebot.TeleBot(TELEGRAM_TOKEN) openai.api_key = OPENAI_API_KEY

ملفات التخزين

USERS_FILE = 'users.json' USAGE_FILE = 'usage.json'

تحميل أو إنشاء ملفات

try: with open(USERS_FILE, 'r') as f: users = json.load(f) except: users = {}

try: with open(USAGE_FILE, 'r') as f: usage = json.load(f) except: usage = {}

دوال مساعدة

def save_users(): with open(USERS_FILE, 'w') as f: json.dump(users, f)

def save_usage(): with open(USAGE_FILE, 'w') as f: json.dump(usage, f)

def user_is_paid(user_id): return users.get(str(user_id), {}).get("paid", False)

def update_usage(user_id): usage[str(user_id)] = time.time() save_usage()

def can_use_free(user_id): last = usage.get(str(user_id), 0) return (time.time() - last) > 86400

جلب عروض عمل (مثال من Emploitic فقط)

def get_emploitic_jobs(): try: url = "https://www.emploitic.com/offres-d-emploi" r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}) soup = BeautifulSoup(r.text, 'html.parser') jobs = [] for offer in soup.select("div.job-title > a")[:5]: jobs.append(f"- {offer.text.strip()}\nhttps://www.emploitic.com{offer['href']}") return "\n\n".join(jobs) if jobs else "لا توجد عروض حالياً." except: return "حدث خطأ أثناء جلب العروض."

اختيار اللغة للمستخدم

languages = { 'ar': '🇩🇿 عربي', 'fr': '🇫🇷 Français', 'en': '🇬🇧 English' }

messages = { 'ar': { 'welcome': "مرحباً بك! اختر خدمة من القائمة:", 'job_title': "عروض العمل", 'scholarship': "منح دراسية", 'courses': "كورسات", 'cv': "سيرة ذاتية بالذكاء", 'subscribe': "اشترك 3$", 'donate': "تبرع", 'support': "دعم فني", 'free_wait': "الخدمة المجانية متاحة مرة كل 24 ساعة. اشترك لرؤية المزيد.", 'cv_prompt': "ارسل معلوماتك (الاسم، التخصص، المهارات):", 'cv_error': "خطأ في إنشاء السيرة الذاتية. حاول لاحقاً.", 'cv_result': "سيرتك الذاتية:", 'paid_link': "ادفع عبر Binance:", 'thanks': "شكراً لدعمك!", 'support_contact': "راسلنا عبر الإيميل: zahaffouad826@gmail.com", 'language_prompt': "اختر لغتك:" }, 'fr': { 'welcome': "Bienvenue! Choisissez un service:", 'job_title': "Offres d'emploi", 'scholarship': "Bourses d'études", 'courses': "Cours", 'cv': "CV par IA", 'subscribe': "S'abonner 3$", 'donate': "Faire un don", 'support': "Support technique", 'free_wait': "Service gratuit une fois toutes les 24h. Abonnez-vous pour plus.", 'cv_prompt': "Envoyez vos infos (nom, spécialité, compétences):", 'cv_error': "Erreur lors de la création du CV. Réessayez.", 'cv_result': "Votre CV:", 'paid_link': "Payez via Binance:", 'thanks': "Merci pour votre soutien!", 'support_contact': "Contactez-nous: zahaffouad826@gmail.com", 'language_prompt': "Choisissez votre langue:" }, 'en': { 'welcome': "Welcome! Choose a service:", 'job_title': "Job Offers", 'scholarship': "Scholarships", 'courses': "Courses", 'cv': "CV by AI", 'subscribe': "Subscribe $3", 'donate': "Donate", 'support': "Support", 'free_wait': "Free service available once every 24h. Subscribe for more.", 'cv_prompt': "Send your info (name, major, skills):", 'cv_error': "Error generating CV. Try again later.", 'cv_result': "Your CV:", 'paid_link': "Pay via Binance:", 'thanks': "Thanks for your support!", 'support_contact': "Contact us: zahaffouad826@gmail.com", 'language_prompt': "Choose your language:" } }

user_lang = {}

@bot.message_handler(commands=['start']) def send_language(message): markup = types.ReplyKeyboardMarkup(resize_keyboard=True) for code, name in languages.items(): markup.add(name) bot.send_message(message.chat.id, messages['ar']['language_prompt'], reply_markup=markup)

@bot.message_handler(func=lambda m: m.text in languages.values()) def set_language(message): user_id = str(message.from_user.id) for code, name in languages.items(): if message.text == name: user_lang[user_id] = code users[user_id] = users.get(user_id, {"paid": False}) save_users() send_main_menu(message, code) return

القائمة الرئيسية

def send_main_menu(message, lang): markup = types.ReplyKeyboardMarkup(resize_keyboard=True) texts = messages[lang] markup.add(texts['job_title'], texts['scholarship'], texts['courses']) markup.add(texts['cv'], texts['subscribe'], texts['donate'], texts['support']) bot.send_message(message.chat.id, texts['welcome'], reply_markup=markup)

@bot.message_handler(func=lambda m: True) def reply_user(message): user_id = str(message.from_user.id) lang = user_lang.get(user_id, 'ar') text = message.text texts = messages[lang]

if text == texts['job_title']:
    if user_is_paid(user_id) or can_use_free(user_id):
        bot.send_message(message.chat.id, get_emploitic_jobs())
        if not user_is_paid(user_id):
            update_usage(user_id)
    else:
        bot.send_message(message.chat.id, texts['free_wait'])

elif text == texts['scholarship']:
    bot.send_message(message.chat.id, "- Turkey Scholarship\n- Erasmus+\n- Local Grants")

elif text == texts['courses']:
    bot.send_message(message.chat.id, "- Python from Coursera\n- AI from Udemy\n- Web Dev from edX")

elif text == texts['cv']:
    bot.send_message(message.chat.id, texts['cv_prompt'])
    bot.register_next_step_handler(message, generate_cv)

elif text == texts['subscribe']:
    bot.send_message(message.chat.id, texts['paid_link'])
    bot.send_message(message.chat.id, "https://binance.com/paylink/THPof7vZTAevKxhhLDu4Sk93QoXKzM4FqJ")
    users[user_id]['paid'] = True
    save_users()

elif text == texts['donate']:
    bot.send_message(message.chat.id, texts['thanks'])
    bot.send_message(message.chat.id, "https://binance.com/paylink/THPof7vZTAevKxhhLDu4Sk93QoXKzM4FqJ")

elif text == texts['support']:
    bot.send_message(message.chat.id, texts['support_contact'])

else:
    bot.send_message(message.chat.id, "/start")

@bot.message_handler(content_types=['text']) def generate_cv(message): user_id = str(message.from_user.id) lang = user_lang.get(user_id, 'ar') try: response = openai.ChatCompletion.create( model="gpt-4", messages=[ {"role": "system", "content": "أنشئ سيرة ذاتية بناءً على المدخلات."}, {"role": "user", "content": message.text} ], max_tokens=500 ) cv = response.choices[0].message.content bot.send_message(message.chat.id, f"{messages[lang]['cv_result']}\n\n{cv}") except Exception as e: bot.send_message(message.chat.id, messages[lang]['cv_error'])

bot.polling()
