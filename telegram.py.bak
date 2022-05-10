# Ванюков Е.Е. 05_2022
# my tg: @Evan_N
# tg-bot: @Evan_location_bot

from pathlib import Path
import sqlite3
from bot_token import token
import telebot
from telebot import types
from collections import defaultdict
import time
from geopy.distance import geodesic # https://pythonpip.ru/osnovy/geopy-python - вычисление расстояние


# Build paths inside the project like this: BASE_DIR / 'subdir'.
BASE_DIR = Path(__file__).resolve().parent
DB_DIR = BASE_DIR / "dBase" 
DB_NAME= DB_DIR / "places.db"
START, ADD_PLACE, ADD_PHOTO, ADD_LOCATION, ADD_SAVE, LIST, RESET = range(7)
USER_STATE = defaultdict(lambda:START)
answer = [False, True]
BOT_DESCRIPTION = '''Я бот для сохранения ваших любимых мест! Введите нужную команду:
    /add - Добавить новое место
    /list - Вывести список 10 последних добавленных мест
    /reset - Очистить ваш личный список
    /help - Вывести описание команд
    Если вы пришлете мне локацию, я покажу сохраненные места из вашего списка рядом (500 м)'''

class DataConn:
 
    def __init__(self, db_name):
        """Конструктор"""
        self.db_name = db_name
    
    def __enter__(self):
        """
        Открываем подключение с базой данных.
        """
        self.conn = sqlite3.connect(self.db_name)
        return self.conn
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """
        Закрываем подключение.
        """
        self.conn.close()
        if exc_val:
            raise

def create_database():
    if not DB_DIR.is_dir():
        DB_DIR.mkdir()
    with DataConn(DB_NAME) as conn:
        #cursor = conn.cursor()    
        #conn = sqlite3.connect( DB_DIR)
        cur = conn.cursor()
        cur.execute("""CREATE TABLE IF NOT EXISTS places(
            id INTEGER PRIMARY KEY,
            chatid INT,
            title TEXT,
            location TEXT,
            filename TEXT);
            """)
    
class dataBase():
    def __init__(self):
        self.id=defaultdict(lambda:False)


class Data():
    def __init__(self):
        self.name=''
        self.location=tuple()
        self.photo_name=''
        self.photo_file=None


current_data = dict()

def save_name(message):
    current_data[message.chat.id].name = message.text

def save_location(message):
    current_data[message.chat.id].location = (message.location.latitude, message.location.longitude)

def save_photo(message):
    current_data[message.chat.id].photo_name = get_file_name(message)
    file_info = bot.get_file(message.photo[len(message.photo) - 1].file_id)
    current_data[message.chat.id].photo_file = bot.download_file(file_info.file_path)

def new_current_data(message):
    current_data[message.chat.id]=Data()

def clear_current_data(message):
    if (current_data.get(message.chat.id, False)):
        del current_data[message.chat.id]
    
def get_file_name(message):
    tl = time.localtime()
    fn = str (message.chat.id) + f"_{tl.tm_year}_{tl.tm_mon}_{tl.tm_mday}_{tl.tm_hour}_{tl.tm_min}_{tl.tm_sec}"
    return DB_DIR / fn

def get_location(data):
    return [float(i) for i in data[1:-1].split(',')]

def get_state(message):
    return USER_STATE[message.chat.id]

def update_state(message, state):
    USER_STATE[message.chat.id] = state

def reset_data_base(message):
    with DataConn(DB_NAME) as conn:
        cur = conn.cursor()
        cur.execute(f"SELECT filename FROM places WHERE chatid={message.chat.id};")
        #remove fotofiles connected with chatid
        photofiles = cur.fetchall()
        for filename in photofiles:
            if (filename[0]):
                Path(filename[0]).unlink(missing_ok=True)
        #remove data with chatid
        cur.execute(f"DELETE FROM places WHERE chatid={message.chat.id};")
        conn.commit()


def add_current_data(message):
    data = current_data[message.chat.id]
    entry = (message.chat.id, data.name, str(data.location), str(data.photo_name))
    print(entry)
    with DataConn(DB_NAME) as conn:
        cur = conn.cursor()
        cur.execute("INSERT INTO places (chatid, title, location, filename) VALUES(?, ?, ?, ?);",entry)
        conn.commit()

    if(data.photo_name):
        try:
            with open(data.photo_name, 'wb') as new_file:
                new_file.write(data.photo_file)
        except:
            bot.send_message(message.chat.id, text="Извините, что-то пошло не так, и я сохранил место без фото:-(")

def answer_keyboard():
    keybord = types.InlineKeyboardMarkup(row_width=2)
    buttons = [types.InlineKeyboardButton(text = "Да", callback_data= "true"),
                types.InlineKeyboardButton(text = "Нет", callback_data= "false")]
    keybord.add(*buttons)
    return keybord


bot = telebot.TeleBot(token.token)

@bot.callback_query_handler(func=lambda x:True)
def answer_handler(callback_query):
    message = callback_query.message
    answer = callback_query.data
    if (get_state(message)==RESET):
        if (answer == "true"):
            bot.send_message(message.chat.id, text="Список сохраненных мест очищен!")
            reset_data_base(message)
            clear_current_data(message)
        elif (answer == "false"):
            bot.send_message(message.chat.id, text="Очистка отменена")
        update_state(message, START)
    
    if (get_state(message)==ADD_SAVE):
        if (answer == "true"):
            add_current_data(message)
            bot.send_message(message.chat.id, text="Место сохранено!")
        elif (answer == "false"):
            bot.send_message(message.chat.id, text="Добавление места отменено")
        update_state(message, START)
        clear_current_data(message)

@bot.message_handler(commands=['start', 'help'])
def reset(message):
    bot.send_message(message.chat.id, text=BOT_DESCRIPTION)
    update_state(message, START)

@bot.message_handler(commands=['reset'])
def reset(message):
    keybord = answer_keyboard()
    bot.send_message(message.chat.id, text="Очистить весь список?", reply_markup= keybord)
    update_state(message, RESET)
    
@bot.message_handler(commands=['list'])
def list(message):
    if ( get_state(message)!= START):
        bot.send_message(message.chat.id, text="Ввод данных прерван!")
        clear_current_data(message)
    bot.send_message(message.chat.id, text="Последние сохраненные места:")
    
    with DataConn(DB_NAME) as conn:
        cur = conn.cursor()
        cur.execute(f"""SELECT title, location, filename FROM places WHERE chatid={message.chat.id}
                        ORDER BY id DESC LIMIT 10;""")
        entries = cur.fetchall()
        if (entries):
            for i, data in enumerate(entries):
                bot.send_message(message.chat.id, text=f"# {i+1}:")
                bot.send_message(message.chat.id, text = data[0])
                bot.send_location(message.chat.id, *get_location(data[1]))
                if (data[2]):
                    try:
                        bot.send_photo(message.chat.id, photo=open(Path(data[2]), 'rb'))
                    except:
                        pass
                    #photo = open('path', 'rb'))
        else:
            bot.send_message(message.chat.id, text="Ваш список пуст!")
        
    update_state(message, START)

@bot.message_handler(commands=['add'])
def add_place(message):
    if ( get_state(message)!= START):
        bot.send_message(message.chat.id, text="Ввод данных прерван!")
        clear_current_data(message)
    new_current_data(message)
    bot.send_message(message.chat.id, text="Введите название нового места:")
    update_state(message, ADD_PLACE)

@bot.message_handler(commands=['skip'])
def skip(message):
    #print(get_state(message))
    if (get_state(message) == ADD_PHOTO):
        update_state(message, ADD_SAVE)
        keybord = answer_keyboard()
        bot.send_message(message.chat.id, text="Сохранить место в базу?", reply_markup= keybord)
        
# Handles all sent text
@bot.message_handler(content_types=['text'])
def text(message):
    if (get_state(message) == ADD_PLACE):
        save_name(message)
        update_state(message, ADD_LOCATION)
        bot.send_message(message.chat.id, text="Отправьте геопозицию нового места следующим сообщением:")

@bot.message_handler(content_types=['location'])
def add_location(message):
  
    if (get_state(message) == ADD_LOCATION):
        save_location(message)
        update_state(message, ADD_PHOTO)
        bot.send_message(message.chat.id, text="Отправьте фото нового места следующим сообщением или /skip:")
    elif (get_state(message) == START):
        bot.send_message(message.chat.id, text="Сохраненные места в пределах 500 м от указанного:")
        # выводим из базы сохраенные места
        with DataConn(DB_NAME) as conn:
            cur = conn.cursor()
            cur.execute(f"SELECT title, location, filename FROM places WHERE chatid={message.chat.id};")
            entries = cur.fetchall()
            if (entries):
                count = 0
                for data in reversed(entries):
                    distance = geodesic(get_location(data[1]), (message.location.latitude, message.location.longitude)).m
                    if distance < 500:
                        bot.send_message(message.chat.id, text=f"# {count+1}:")
                        bot.send_message(message.chat.id, text = data[0])
                        bot.send_location(message.chat.id, *get_location(data[1]))
                        if (data[2]):
                            try:
                                bot.send_photo(message.chat.id, photo=open(Path(data[2]), 'rb'))
                            except:
                                pass
                        count +=1
                    else:
                        bot.send_message(message.chat.id, text="Поблизости нет любимых мест!")
            else:
                bot.send_message(message.chat.id, text="Ваш список пуст!")

@bot.message_handler(content_types=['photo'])
def add_photo(message):
    #print(get_state(message))
    if (get_state(message) == ADD_PHOTO):
        try:
            save_photo(message)
            bot.send_message(message.chat.id, text = "Фото добавлено")
            update_state(message, ADD_SAVE)
            keybord = answer_keyboard()
            bot.send_message(message.chat.id, text="Сохранить место в базу?", reply_markup= keybord)
        except Exception as e:
            bot.reply_to(message, e)
        
if __name__ == '__main__':
    # create dBase
    #try:
    create_database()
    bot.polling()
    #except Exception as e:
    #        print(e)

