#!/usr/bin/env python3
"""
Weather Notification Bot
Отправляет уведомления о погоде в Telegram по расписанию.
"""

import os
import logging
import asyncio
from datetime import datetime
from typing import Optional, Dict, Any

import aiohttp
import pytz
from aiogram import Bot, Dispatcher, types
from aiogram.contrib.middlewares.logging import LoggingMiddleware
from aiogram.utils import executor
from dotenv import load_dotenv
from apscheduler.schedulers.asyncio import AsyncIOScheduler

# Загружаем переменные окружения
load_dotenv()

# Конфигурация
TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
OPENWEATHER_API_KEY = os.getenv("OPENWEATHER_API_KEY")
DEFAULT_CITY = os.getenv("DEFAULT_CITY", "Moscow")
CHAT_ID = os.getenv("CHAT_ID")  # ID чата для рассылки
TIMEZONE = os.getenv("TIMEZONE", "Europe/Moscow")

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
logger = logging.getLogger(__name__)

# Инициализация бота и диспетчера
bot = Bot(token=TELEGRAM_TOKEN)
dp = Dispatcher(bot)
dp.middleware.setup(LoggingMiddleware())

# Планировщик задач
scheduler = AsyncIOScheduler(timezone=pytz.timezone(TIMEZONE))


class WeatherService:
    """Сервис для работы с OpenWeatherMap API"""
    
    BASE_URL = "https://api.openweathermap.org/data/2.5/weather"
    
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    async def get_weather(self, city: str) -> Optional[Dict[str, Any]]:
        """
        Получает текущую погоду для города
        
        Args:
            city: Название города
            
        Returns:
            Словарь с данными о погоде или None при ошибке
        """
        async with aiohttp.ClientSession() as session:
            params = {
                "q": city,
                "appid": self.api_key,
                "units": "metric",  # Цельсии
                "lang": "ru"  # Русский язык
            }
            
            try:
                async with session.get(self.BASE_URL, params=params) as resp:
                    if resp.status == 200:
                        data = await resp.json()
                        return self._format_weather_data(data)
                    else:
                        error_text = await resp.text()
                        logger.error(f"API error {resp.status}: {error_text}")
                        return None
            except aiohttp.ClientError as e:
                logger.error(f"Connection error: {e}")
                return None
    
    def _format_weather_data(self, raw_data: Dict) -> Dict:
        """Форматирует сырые данные от API"""
        return {
            "city": raw_data["name"],
            "temperature": round(raw_data["main"]["temp"]),
            "feels_like": round(raw_data["main"]["feels_like"]),
            "humidity": raw_data["main"]["humidity"],
            "pressure": raw_data["main"]["pressure"],
            "wind_speed": raw_data["wind"]["speed"],
            "description": raw_data["weather"][0]["description"].capitalize(),
            "icon": raw_data["weather"][0]["icon"]
        }


def format_weather_message(weather: Dict[str, Any]) -> str:
    """
    Форматирует сообщение с погодой для отправки в Telegram
    
    Args:
        weather: Словарь с данными погоды
        
    Returns:
        Отформатированное сообщение
    """
    weather_icons = {
        "01d": "☀️", "01n": "🌙",
        "02d": "⛅", "02n": "☁️",
        "03d": "☁️", "03n": "☁️",
        "04d": "☁️", "04n": "☁️",
        "09d": "🌧️", "09n": "🌧️",
        "10d": "🌦️", "10n": "🌧️",
        "11d": "⛈️", "11n": "⛈️",
        "13d": "❄️", "13n": "❄️",
        "50d": "🌫️", "50n": "🌫️"
    }
    
    icon_emoji = weather_icons.get(weather["icon"], "🌡️")
    
    message = (
        f"{icon_emoji} *Погода в {weather['city']}*\n\n"
        f"🌡️ Температура: *{weather['temperature']}°C* (ощущается как {weather['feels_like']}°C)\n"
        f"🌬️ Ветер: {weather['wind_speed']} м/с\n"
        f"💧 Влажность: {weather['humidity']}%\n"
        f"📊 Давление: {weather['pressure']} гПа\n"
        f"📝 {weather['description']}\n\n"
        f"🕒 *{datetime.now().strftime('%H:%M')}*"
    )
    return message


async def send_weather_notification():
    """Отправляет уведомление о погоде в Telegram"""
    logger.info("Sending weather notification...")
    
    weather_service = WeatherService(OPENWEATHER_API_KEY)
    weather = await weather_service.get_weather(DEFAULT_CITY)
    
    if weather:
        message = format_weather_message(weather)
        try:
            await bot.send_message(
                chat_id=CHAT_ID,
                text=message,
                parse_mode="Markdown"
            )
            logger.info("Weather notification sent successfully")
        except Exception as e:
            logger.error(f"Failed to send message: {e}")
    else:
        logger.error("Failed to fetch weather data")
        await bot.send_message(
            chat_id=CHAT_ID,
            text="⚠️ Не удалось получить данные о погоде. Проверьте API ключ или название города."
        )


# Команды бота
@dp.message_handler(commands=['start'])
async def cmd_start(message: types.Message):
    """Обработчик команды /start"""
    welcome_text = (
        "🌤️ *Привет! Я погодный бот*\n\n"
        "Я буду присылать тебе сводку о погоде каждый день в 9:00 и 18:00.\n\n"
        "Доступные команды:\n"
        "/weather - получить погоду сейчас\n"
        "/help - помощь\n"
        "/city - изменить город (пока в разработке)"
    )
    await message.reply(welcome_text, parse_mode="Markdown")


@dp.message_handler(commands=['weather'])
async def cmd_weather(message: types.Message):
    """Обработчик команды /weather - показывает текущую погоду"""
    await message.reply("🔍 Запрашиваю погоду...")
    
    weather_service = WeatherService(OPENWEATHER_API_KEY)
    weather = await weather_service.get_weather(DEFAULT_CITY)
    
    if weather:
        weather_text = format_weather_message(weather)
        await message.reply(weather_text, parse_mode="Markdown")
    else:
        await message.reply("❌ Не удалось получить данные о погоде")


@dp.message_handler(commands=['help'])
async def cmd_help(message: types.Message):
    """Обработчик команды /help"""
    help_text = (
        "📚 *Справка по командам*\n\n"
        "/start - Запустить бота\n"
        "/weather - Узнать погоду сейчас\n"
        "/help - Показать эту справку\n\n"
        "Бот автоматически присылает уведомления в 9:00 и 18:00"
    )
    await message.reply(help_text, parse_mode="Markdown")


@dp.message_handler()
async def echo(message: types.Message):
    """Обработчик всех остальных сообщений"""
    await message.reply(
        "🤔 Я не понимаю эту команду. Используйте /help для списка доступных команд."
    )


async def on_startup(dp):
    """Действия при запуске бота"""
    logger.info("Starting bot...")
    
    # Настройка расписания
    scheduler.add_job(
        send_weather_notification,
        trigger="cron",
        hour="9,18",
        minute="0",
        id="weather_notification"
    )
    scheduler.start()
    
    logger.info("Scheduler started. Notifications will be sent at 9:00 and 18:00")
    
    # Отправляем тестовое сообщение при запуске
    await bot.send_message(
        chat_id=CHAT_ID,
        text="✅ Бот запущен и готов к работе! Уведомления будут приходить по расписанию."
    )


async def on_shutdown(dp):
    """Действия при остановке бота"""
    logger.info("Shutting down bot...")
    scheduler.shutdown()
    await bot.close()


if __name__ == "__main__":
    # Проверка обязательных переменных окружения
    required_vars = ["TELEGRAM_TOKEN", "OPENWEATHER_API_KEY", "CHAT_ID"]
    missing_vars = [var for var in required_vars if not os.getenv(var)]
    
    if missing_vars:
        logger.error(f"Missing required environment variables: {', '.join(missing_vars)}")
        logger.error("Please create .env file with these variables")
        exit(1)
    
    executor.start_polling(
        dp,
        on_startup=on_startup,
        on_shutdown=on_shutdown,
        skip_updates=True
    )
