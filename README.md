import json
import psycopg2
from datetime import datetime
import telebot
from transformers import pipeline

def load_video_stats(json_file):
    # Коннект с sql
    conn = psycopg2.connect(
        name="video_stats",
        user="postgres",
        passw="password",
        host="localhost"
    )
    cur = conn.cursor()
    
    with open(json_file, 'r', encoding='utf-8') as f:
        videos = json.load(f)
        
        # Видосы
        for video in videos:
            cur.execute("""
                INSERT INTO videos (id, creator_id, video_created_at, 
                                    views_count, likes_count, comments_count, reports_count)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
            """, (
                video['id'], video['creator_id'], video['video_created_at'],
                video['views_count'], video['likes_count'], 
                video['comments_count'], video['reports_count']
            ))
            
            # Снапшоты
            for snap in video['snapshots']:
                cur.execute("""
                    INSERT INTO video_snapshots 
                        (video_id, views_count, likes_count, comments_count, 
                         reports_count, delta_views_count, delta_likes_count, 
                         delta_comments_count, delta_reports_count, created_at)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                """, (
                    video['id'], snap['views_count'], snap['likes_count'],
                    snap['comments_count'], snap['reports_count'],
                    snap['delta_views_count'], snap['delta_likes_count'],
                    snap['delta_comments_count'], snap['delta_reports_count'],
                    snap['created_at']
                ))


    # API гигачада
    # Бот
    TOKEN = "8661732821:AAH2Xkrtq36rRKAicIKbWhNRNIu7Da0EiVE"
    bot = telebot.TeleBot(TOKEN)

    # NLP
    nlp_model = pipeline("question-answering", model="DeepPavlov/rubert-base-cased-squad2")

    @bot.message_handler(commands=['start'])
    def start(message):
        bot.reply_to(message, "Привет! Я могу показать тебе статистику по твоим видео.")

    @bot.message_handler(func=lambda message: True)
    def handle_message(message):
        try:
            # Анализ с NLP
            result = nlp_model({
                "context": "Покажи статистику по моим последним видео",
                "question": message.text
            })
        
            if result["score"] > 0.7:  
                # Формирование SQL-запросов исходя из контекста вопроса
                sql_query = """
                    SELECT * FROM videos WHERE creator_id=%s ORDER BY video_created_at DESC LIMIT 1
                """
                cur.execute(sql_query, (message.from_user.id,))
                row = cur.fetchone()
            
                if row is not None:
                    response_text = f"Ваше последнее видео:\nID: {row[0]}\nПросмотры: {row[3]} Лайки: {row[4]}"
                    bot.send_message(message.chat.id, response_text)
                else:
                    bot.send_message(message.chat.id, "Нет информации о ваших последних видео.")
                
            else:
                bot.send_message(message.chat.id, "Извините, ваш запрос непонятен. Попробуйте задать его иначе.")
        except Exception as e:
            print(e)
            bot.send_message(message.chat.id, "Ошибка при обработке вашего запроса.")

    if __name__ == "__main__":
        bot.polling(none_stop=True)

    conn.commit()
    cur.close()
    conn.close()
