# lds
мой проект телефон проект давно заброшен <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/dc0f4eff-1c17-4f38-966b-5c265b85ec6f" />

[телефон.py](https://github.com/user-attachments/files/32239807/default.py)
import tkinter as tk
from PIL import ImageTk, Image
from datetime import datetime
import cv2
import webbrowser
import os
import random
import time
import subprocess
from tkinter import messagebox
import requests
import logging
import sys
from pathlib import Path

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('app.log'),
        logging.StreamHandler()
    ]
)

# Функция для получения пути к ресурсам
def get_resource_path(relative_path):
    """Получить путь к ресурсу в режиме разработки или собранном приложении"""
    try:
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.path.abspath(".")

    return os.path.join(base_path, relative_path)

# Функция безопасной загрузки изображений
def load_image_safe(path, size=None):
    """Безопасная загрузка изображения с проверкой существования"""
    try:
        if os.path.exists(path):
            img = Image.open(path)
            if size:
                img = img.resize(size, Image.LANCZOS)
            return ImageTk.PhotoImage(img)
        else:
            logging.warning(f"Файл не найден: {path}")
            return None
    except Exception as e:
        logging.error(f"Ошибка загрузки изображения {path}: {e}")
        return None

# Общедоступный родительский класс для всех окон
class BaseWindow(tk.Toplevel):
    def __init__(self, parent):
        super().__init__(parent)
        self.parent = parent
        self.protocol("WM_DELETE_WINDOW", self.on_closing)

    def create_back_button(self):
        """Создание кнопки 'Назад' в каждом окне."""
        back_button = tk.Button(self, text="◯", font=("Helvetica", 14), command=self.return_to_desktop)
        back_button.pack(side="bottom", pady=10)

    def return_to_desktop(self):
        """Метод для возврата на рабочий стол."""
        self.destroy()

    def on_closing(self):
        """Обработка закрытия окна"""
        self.destroy()

class BatteryIndicator:
    def __init__(self, parent):
        self.parent = parent
        self.charge_level = 100
        self.images = {}
        self.label = None
        self.timer_id = None
        self.load_images()
        self.show_charge()
        self.start_charge_timer()

    def load_images(self):
        """Загрузка изображений батареи"""
        levels = [100, 60, 20, 0]
        for level in levels:
            # Используем относительные пути
            path = os.path.join("D:", f"battery_{level}.png")
            photo = load_image_safe(path)
            if photo:
                self.images[level] = photo

    def show_charge(self):
        """Отображение текущего заряда батареи"""
        if self.charge_level in self.images:
            charge_img = self.images[self.charge_level]
            if self.label:
                self.label.destroy()
            self.label = tk.Label(self.parent, image=charge_img)
            self.label.image = charge_img
            # Обновляем позицию после отрисовки
            self.parent.update_idletasks()
            self.label.place(x=(self.parent.winfo_width() - charge_img.width()), y=0)

    def start_charge_timer(self):
        """Запуск таймера разряда батареи"""
        if self.timer_id:
            self.parent.after_cancel(self.timer_id)
        self.timer_id = self.parent.after(60 * 1000, self.change_charge)

    def change_charge(self):
        """Изменение уровня заряда"""
        next_levels = {100: 60, 60: 20, 20: 0}
        if self.charge_level in next_levels:
            self.charge_level = next_levels[self.charge_level]
            self.show_charge()
            # Перезапускаем таймер если не 0%
            if self.charge_level > 0:
                self.timer_id = self.parent.after(60 * 1000, self.change_charge)
            else:
                logging.info("Батарея разряжена")

class WeatherWidget(BaseWindow):
    def __init__(self, parent):
        super().__init__(parent)
        self.title("Погода")
        self.geometry("300x200")
        # ЗАМЕНИТЕ НА ВАШ РЕАЛЬНЫЙ API КЛЮЧ
        self.api_key = "ваш_реальный_api_ключ"  # Получите на openweathermap.org
        self.city = "Москва"

        self.create_widgets()
        self.get_weather()

    def create_widgets(self):
        """Создание элементов интерфейса"""
        self.city_label = tk.Label(self, text="Город:")
        self.city_label.pack(pady=5)

        self.city_entry = tk.Entry(self, width=20)
        self.city_entry.insert(0, self.city)
        self.city_entry.pack(pady=5)

        self.get_weather_button = tk.Button(
            self,
            text="Обновить",
            command=self.get_weather,
            font=("Helvetica", 12)
        )
        self.get_weather_button.pack(pady=5)

        self.weather_label = tk.Label(
            self,
            font=("Helvetica", 14),
            justify="left",
            wraplength=250
        )
        self.weather_label.pack(pady=10)

        self.create_back_button()

    def get_weather(self):
        """Получение данных о погоде"""
        try:
            city = self.city_entry.get().strip()
            if not city:
                raise ValueError("Город не указан")

            base_url = "http://api.openweathermap.org/data/2.5/weather?"
            complete_url = (
                f"{base_url}q={city}&"
                f"appid={self.api_key}&"
                f"units=metric&"
                f"lang=ru"
            )

            response = requests.get(complete_url, timeout=10)
            response.raise_for_status()
            data = response.json()

            if data["cod"] == 200:
                main = data["main"]
                weather_desc = data["weather"][0]["description"]
                temperature = main["temp"]
                humidity = main["humidity"]
                wind_speed = data["wind"]["speed"]

                weather_info = (
                    f"Температура: {temperature:.1f}°C\n"
                    f"Описание: {weather_desc.capitalize()}\n"
                    f"Влажность: {humidity}%\n"
                    f"Скорость ветра: {wind_speed:.1f} м/с"
                )
                self.weather_label.config(text=weather_info)
                logging.info(f"Погода для города {city} успешно получена")
            else:
                raise ValueError("Город не найден")

        except requests.exceptions.HTTPError as http_err:
            error_msg = f"Ошибка сервера: {http_err}"
            logging.error(error_msg)
            messagebox.showerror("HTTP ошибка", error_msg)
        except requests.exceptions.ConnectionError:
            error_msg = "Проверьте подключение к интернету"
            logging.error(error_msg)
            messagebox.showerror("Ошибка сети", error_msg)
        except requests.exceptions.Timeout:
            error_msg = "Запрос занял слишком много времени"
            logging.error(error_msg)
            messagebox.showerror("Таймаут", error_msg)
        except Exception as e:
            error_msg = f"Произошла ошибка: {str(e)}"
            logging.error(error_msg)
            messagebox.showerror("Ошибка", error_msg)

class PhoneDialer(BaseWindow):
    def __init__(self, master=None):
        super().__init__(master)
        self.geometry("300x400")
        self.current_number = ""
        self.create_widgets()

    def create_widgets(self):
        """Создание интерфейса телефона"""
        # Верхняя область для отображения номера
        display_frame = tk.Frame(self)
        display_frame.pack(fill="x")
        self.number_display = tk.Label(display_frame, text="", font=("Helvetica", 24),
                                      justify="right", anchor="e", bg="white", relief="sunken")
        self.number_display.pack(fill="x", ipadx=10, ipady=10)

        # Цифровая клавиатура
        keyboard_frame = tk.Frame(self)
        keyboard_frame.pack(expand=True, fill="both")

        buttons = [
            ('1', '2', '3'),
            ('4', '5', '6'),
            ('7', '8', '9'),
            ('*', '0', '#')
        ]

        for row_idx, row in enumerate(buttons):
            for col_idx, digit in enumerate(row):
                btn = tk.Button(
                    keyboard_frame,
                    text=digit,
                    font=("Helvetica", 18),
                    width=5,
                    height=2,
                    command=lambda d=digit: self.add_digit(d)
                )
                btn.grid(row=row_idx, column=col_idx, sticky="nsew", padx=2, pady=2)

            # Настройка веса строк и столбцов
            keyboard_frame.grid_rowconfigure(row_idx, weight=1)
            for col_idx in range(3):
                keyboard_frame.grid_columnconfigure(col_idx, weight=1)

        # Кнопка вызова
        call_button = tk.Button(self, text="📞", font=("Helvetica", 14), command=self.make_call)
        call_button.pack(pady=10)

        # Кнопка возврата
        self.create_back_button()

    def add_digit(self, digit):
        if digit == '*':
            self.clear_number()
        elif digit == '#':
            self.delete_last_digit()
        else:
            if len(self.current_number) < 15:  # Ограничение длины номера
                self.current_number += digit
                self.update_display()

    def make_call(self):
        if not self.current_number.strip():
            messagebox.showwarning("Предупреждение", "Введите номер телефона")
            return
        CallWindow(self, self.current_number)

    def clear_number(self):
        self.current_number = ""
        self.update_display()

    def delete_last_digit(self):
        self.current_number = self.current_number[:-1]
        self.update_display()

    def update_display(self):
        self.number_display.config(text=self.current_number)

class CallWindow(BaseWindow):
    def __init__(self, parent, number):
        super().__init__(parent)
        self.title("Звонок")
        self.geometry("300x200")

        label = tk.Label(self, text=f"Звонок: {number}\n\nИдет звонок...",
                        font=("Helvetica", 18))
        label.pack(pady=20)

        hangup_button = tk.Button(self, text="❌", font=("Helvetica", 14),
                                 command=self.hang_up)
        hangup_button.pack(pady=10)

        # Автоматическое завершение звонка через 30 секунд
        self.after(30000, self.hang_up)

    def hang_up(self):
        self.destroy()

class CameraWindow(BaseWindow):
    def __init__(self, parent):
        super().__init__(parent)
        self.title("📷 Камера")
        self.geometry("640x600")
        self.cap = None
        self.is_running = True

        try:
            self.cap = cv2.VideoCapture(0)
            if not self.cap.isOpened():
                raise ValueError("Не удаётся захватить видео с камеры.")

            # Холст для отображения видео
            self.canvas = tk.Canvas(self, width=640, height=480)
            self.canvas.pack()

            # Кнопка для фотосъёмки
            capture_button = tk.Button(self, text="📷 Сделать фото", font=("Helvetica", 14),
                                      command=self.capture_photo)
            capture_button.pack(pady=10)

            # Кнопка для видео
            video_button = tk.Button(self, text="🎥 Запись видео", font=("Helvetica", 14),
                                    command=self.start_video)
            video_button.pack(pady=5)

            # Периодическое обновление кадра
            self.update_camera_feed()
            self.create_back_button()

        except Exception as e:
            logging.error(f"Ошибка инициализации камеры: {e}")
            messagebox.showerror("Ошибка", f"Не удалось открыть камеру: {e}")
            self.destroy()

    def update_camera_feed(self):
        """Обновление видеопотока с камеры"""
        if self.is_running and self.cap and self.cap.isOpened():
            ret, frame = self.cap.read()
            if ret:
                frame_rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                img = Image.fromarray(frame_rgb)
                img = img.resize((640, 480), Image.LANCZOS)
                img_tk = ImageTk.PhotoImage(image=img)
                self.canvas.img_tk = img_tk
                self.canvas.create_image(0, 0, anchor="nw", image=img_tk)
            self.after(10, self.update_camera_feed)

    def capture_photo(self):
        """Сохранение фотографии"""
        if self.cap and self.cap.isOpened():
            ret, frame = self.cap.read()
            if ret:
                timestamp = datetime.now().strftime('%Y-%m-%d_%H-%M-%S')
                # Создаем папку для фотографий
                photo_dir = os.path.join("D:", "фотки")
                os.makedirs(photo_dir, exist_ok=True)
                filename = os.path.join(photo_dir, f'photo_{timestamp}.png')
                cv2.imwrite(filename, frame)
                logging.info(f"Сохранено изображение: {filename}")
                messagebox.showinfo("Успех", f"Фото сохранено: {filename}")

    def start_video(self):
        """Запись видео"""
        messagebox.showinfo("Инфо", "Функция записи видео будет добавлена в следующей версии")

    def on_closing(self):
        """Освобождение ресурсов при закрытии"""
        self.is_running = False
        if self.cap:
            self.cap.release()
        self.destroy()

class CalculatorWindow(BaseWindow):
    def __init__(self, parent):
        super().__init__(parent)
        self.title("Калькулятор")
        self.geometry("300x500")
        self.expression = ""
        self.create_widgets()

    def create_widgets(self):
        """Создание интерфейса калькулятора"""
        # Поле ввода
        self.result_display = tk.Entry(self, font=("Helvetica", 24), bd=5,
                                       insertwidth=4, justify='right', state='readonly')
        self.result_display.pack(fill="x", padx=10, pady=10)
        self.update_display()

        # Кнопки калькулятора
        buttons_frame = tk.Frame(self)
        buttons_frame.pack(expand=True, fill="both", padx=10, pady=10)

        buttons = [
            ('7', '8', '9', '/'),
            ('4', '5', '6', '*'),
            ('1', '2', '3', '-'),
            ('0', '.', '=', '+'),
            ('C', '(', ')', '←')
        ]

        for i, row in enumerate(buttons):
            for j, char in enumerate(row):
                btn = tk.Button(
                    buttons_frame,
                    text=char,
                    font=("Helvetica", 18),
                    width=5,
                    height=2,
                    command=lambda ch=char: self.on_click(ch)
                )
                btn.grid(row=i, column=j, sticky="nsew", padx=2, pady=2)

            buttons_frame.grid_rowconfigure(i, weight=1)
            for j in range(4):
                buttons_frame.grid_columnconfigure(j, weight=1)

        self.create_back_button()

    def on_click(self, key):
        """Обработка нажатия кнопок"""
        try:
            if key == '=':
                result = self.safe_calculate(self.expression)
                self.expression = str(result)
            elif key == 'C':
                self.expression = ''
            elif key == '←':
                self.expression = self.expression[:-1]
            else:
                self.expression += key
            self.update_display()
        except Exception as e:
            logging.error(f"Ошибка вычисления: {e}")
            self.expression = "Ошибка"
            self.update_display()
            self.after(1000, self.clear_error)

    def safe_calculate(self, expression):
        """Безопасное вычисление математического выражения"""
        # Проверка на пустое выражение
        if not expression:
            return "0"

        # Очищаем выражение от опасных символов
        allowed_chars = set('0123456789+-*/(). ')
        if not all(c in allowed_chars for c in expression):
            raise ValueError("Недопустимые символы")

        # Проверка на деление на ноль
        if '/0' in expression:
            raise ZeroDivisionError("Деление на ноль")

        # Безопасное вычисление
        try:
            result = eval(expression, {"__builtins__": {}}, {})
            # Округляем до 10 знаков после запятой
            if isinstance(result, float):
                result = round(result, 10)
            return result
        except ZeroDivisionError:
            return "Ошибка: деление на 0"
        except:
            return "Ошибка"

    def update_display(self):
        """Обновление дисплея"""
        self.result_display.config(state='normal')
        self.result_display.delete(0, tk.END)
        self.result_display.insert(0, self.expression)
        self.result_display.config(state='readonly')

    def clear_error(self):
        """Очистка сообщения об ошибке"""
        if self.expression == "Ошибка":
            self.expression = ""
            self.update_display()

class ChatWindow(BaseWindow):
    def __init__(self, parent):
        super().__init__(parent)
        self.title("Чат GPT")
        self.geometry("500x400")
        self.create_widgets()

    def create_widgets(self):
        """Создание интерфейса чата"""
        # Область для отображения сообщений
        self.chat_area = tk.Text(self, font=("Helvetica", 12), wrap=tk.WORD,
                                height=15, state='disabled')
        self.chat_area.pack(fill="both", expand=True, padx=10, pady=10)

        # Скроллбар
        scrollbar = tk.Scrollbar(self.chat_area)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        self.chat_area.config(yscrollcommand=scrollbar.set)
        scrollbar.config(command=self.chat_area.yview)

        # Поле ввода
        input_frame = tk.Frame(self)
        input_frame.pack(fill="x", padx=10, pady=5)

        self.input_field = tk.Entry(input_frame, font=("Helvetica", 14))
        self.input_field.pack(side="left", fill="x", expand=True, padx=(0, 5))
        self.input_field.bind("<Return>", lambda event: self.send_message())

        send_button = tk.Button(input_frame, text="Отправить", font=("Helvetica", 12),
                               command=self.send_message)
        send_button.pack(side="right")

        self.create_back_button()

    def send_message(self):
        """Отправка сообщения"""
        message = self.input_field.get().strip()
        if not message:
            return

        # Добавляем сообщение пользователя в чат
        self.add_to_chat(f"Вы: {message}")
        self.input_field.delete(0, tk.END)

        # Получаем ответ
        response = self.get_response(message)
        self.add_to_chat(f"Бот: {response}")

    def add_to_chat(self, text):
        """Добавление текста в область чата"""
        self.chat_area.config(state='normal')
        self.chat_area.insert(tk.END, text + "\n\n")
        self.chat_area.see(tk.END)
        self.chat_area.config(state='disabled')

    def get_response(self, message):
        """Генерация ответа бота"""
        responses = {
            "привет": "Привет! Я ваш виртуальный помощник. Чем могу помочь?",
            "пока": "До свидания! Буду рад помочь снова!",
            "как дела": "У меня всё отлично! А как ваши дела?",
            "что ты умеешь": "Я могу общаться, отвечать на вопросы, помогать с задачами и многое другое!",
            "спасибо": "Пожалуйста! Всегда рад помочь!",
            "помощь": "Я могу ответить на ваши вопросы, поддержать беседу или помочь с простыми задачами.",
            "кто тебя создал": "Меня создал талантливый разработчик как часть этого приложения!",
            "бот": "Да, я бот-помощник. Чем могу быть полезен?",
            "имя": "Меня зовут Помощник. Приятно познакомиться!",
        }

        # Поиск ответа
        message_lower = message.lower()
        for key, response in responses.items():
            if key in message_lower:
                return response

        # Если нет готового ответа
        return f"Интересный вопрос! Я ещё учусь, но обязательно найду ответ на '{message}'."

class GamesWindow(BaseWindow):
    def __init__(self, parent):
        super().__init__(parent)
        self.title("Игры")
        self.geometry("400x300")

        self.games = {
            "Hamster Kombat": r"D:\прогром\питончеееек\хамсет комбат.py",
            "Змейка": r"D:\игры\snake.py",
            "Крестики-нолики": r"D:\игры\tic_tac_toe.py"
        }

        self.create_widgets()

    def create_widgets(self):
        """Создание интерфейса игр"""
        title = tk.Label(self, text="Выберите игру", font=("Helvetica", 18, "bold"))
        title.pack(pady=10)

        for game_name in self.games.keys():
            game_btn = tk.Button(
                self,
                text=game_name,
                font=("Helvetica", 14),
                width=20,
                height=2,
                command=lambda name=game_name: self.open_game(name)
            )
            game_btn.pack(pady=5)

        self.create_back_button()

    def open_game(self, game_title):
        """Открытие выбранной игры"""
        game_path = self.games[game_title]
        if os.path.exists(game_path):
            try:
                subprocess.Popen(['python', game_path])
                logging.info(f"Запущена игра: {game_title}")
            except Exception as e:
                logging.error(f"Ошибка запуска игры {game_title}: {e}")
                messagebox.showerror("Ошибка", f"Не удалось запустить игру: {e}")
        else:
            messagebox.showerror("Ошибка", f"Игра '{game_title}' не найдена по пути:\n{game_path}")

class BrowserWindow(BaseWindow):
    def __init__(self, parent, url="https://www.google.com"):
        super().__init__(parent)
        self.title("Браузер")
        self.geometry("500x400")

        self.create_widgets(url)

    def create_widgets(self, url):
        """Создание интерфейса браузера"""
        label = tk.Label(self, text="Веб-браузер", font=("Helvetica", 18, "bold"))
        label.pack(pady=20)

        url_label = tk.Label(self, text=f"Откроется: {url}", font=("Helvetica", 12))
        url_label.pack(pady=10)

        open_button = tk.Button(
            self,
            text="Открыть в браузере",
            font=("Helvetica", 14),
            width=20,
            height=2,
            command=lambda: webbrowser.open(url)
        )
        open_button.pack(pady=20)

        # Поле для ввода URL
        url_frame = tk.Frame(self)
        url_frame.pack(pady=10, padx=20, fill="x")

        tk.Label(url_frame, text="URL:").pack(side="left", padx=5)
        self.url_entry = tk.Entry(url_frame)
        self.url_entry.pack(side="left", fill="x", expand=True, padx=5)
        self.url_entry.insert(0, url)

        go_button = tk.Button(url_frame, text="Перейти", command=self.go_to_url)
        go_button.pack(side="left", padx=5)

        self.create_back_button()

    def go_to_url(self):
        """Переход по введенному URL"""
        url = self.url_entry.get().strip()
        if not url.startswith(('http://', 'https://')):
            url = 'https://' + url
        webbrowser.open(url)
        logging.info(f"Открыт URL: {url}")

# Главный класс приложения
class MainApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Рабочий стол")
        self.geometry("574x634")

        # Инициализация индикатора батареи
        self.battery_indicator = BatteryIndicator(self)

        # Настройка фона
        self.setup_background()

        # Создание виджетов
        self.create_widgets()

        # Запуск часов
        self.update_clock()

        logging.info("Приложение успешно запущено")

    def setup_background(self):
        """Настройка фонового изображения"""
        bg_image = load_image_safe(r"D:\основапроекта.png")
        if bg_image:
            background_label = tk.Label(self, image=bg_image)
            background_label.place(relwidth=1, relheight=1)
            background_label.image = bg_image

    def create_widgets(self):
        """Создание всех виджетов на рабочем столе"""
        # Часы
        self.clock_label = tk.Label(self, font=("Helvetica", 100), fg="black", bg="blue")
        self.clock_label.pack(side="top", anchor="n", pady=10)

        # Создание иконок приложений
        apps = [
            ("GPT", r"D:\гпт.png", self.open_gpt_chat, 0.3, 0.6),
            ("Мессенджер", r"D:\приложение1.png", self.open_messenger, 0.5, 1.0, "s"),
            ("Звонки", r"D:\звонок.png", self.open_phone_dialer, 0.1, 1.0, "sw"),
            ("Погода", r"D:\иконке_погоды.png", self.open_weather, 0.5, 0.8),
            ("Игры", r"D:\игры.png", self.open_games, 0.9, 1.0, "se"),
            ("Камера", r"D:\камера.png", self.open_camera, 0.9, 0.5, "e"),
            ("Калькулятор", r"D:\калькулятор.png", self.open_calculator, 0.5, 0.4),
            ("Браузер", r"D:\браузер.png", self.open_browser, 0.7, 0.5, "e")
        ]

        for app in apps:
            self.create_app_icon(*app)

        # Логотип Google
        google_logo = load_image_safe(r"D:\гугл.png", (550, 50))
        if google_logo:
            logo_button = tk.Button(self, image=google_logo, borderwidth=0,
                                   highlightthickness=0, activebackground="#ffffff",
                                   relief="flat", command=self.open_google)
            logo_button.place(relx=0.5, rely=0.3, anchor="center")
            logo_button.image = google_logo

    def create_app_icon(self, name, icon_path, command, relx, rely, anchor="center"):
        """Создание иконки приложения"""
        icon = load_image_safe(icon_path, (50, 50))
        if icon:
            button = tk.Button(self, image=icon, command=command)
            button.place(relx=relx, rely=rely, anchor=anchor)
            button.image = icon
        else:
            # Если иконка не найдена, создаем текстовую кнопку
            button = tk.Button(self, text=name, font=("Helvetica", 10), command=command)
            button.place(relx=relx, rely=rely, anchor=anchor)

    def update_clock(self):
        """Обновление часов"""
        current_time = datetime.now().strftime("%H:%M:%S")
        self.clock_label.config(text=current_time)
        self.after(1000, self.update_clock)

    def open_messenger(self):
        """Открытие мессенджера"""
        script_path = r"D:\прогром\питончеееек\месенджер.py"
        if os.path.exists(script_path):
            subprocess.Popen(["python", script_path])
            logging.info("Мессенджер запущен")
        else:
            messagebox.showerror("Ошибка", "Мессенджер не найден")

    def open_games(self):
        """Открытие окна с играми"""
        games_window = GamesWindow(self)
        games_window.focus_force()

    def open_google(self):
        """Открытие Google"""
        webbrowser.open("https://www.google.com")

    def open_phone_dialer(self):
        """Открытие телефона"""
        phone_dialer = PhoneDialer(master=self)
        phone_dialer.focus_force()

    def open_camera(self):
        """Открытие камеры"""
        try:
            cam_window = CameraWindow(self)
            cam_window.focus_force()
        except Exception as e:
            logging.error(f"Ошибка открытия камеры: {e}")
            messagebox.showerror("Ошибка", "Не удалось открыть камеру")

    def open_calculator(self):
        """Открытие калькулятора"""
        calculator = CalculatorWindow(self)
        calculator.focus_force()

    def open_gpt_chat(self):
        """Открытие чата с GPT"""
        chat_window = ChatWindow(self)
        chat_window.focus_force()

    def open_browser(self):
        """Открытие браузера"""
        browser_win = BrowserWindow(self)
        browser_win.focus_force()

    def open_weather(self):
        """Открытие погоды"""
        weather_window = WeatherWidget(self)
        weather_window.focus_force()

# Запуск главного приложения
if __name__ == "__main__":
    try:
        app = MainApp()
        app.mainloop()
    except Exception as e:
        logging.critical(f"Критическая ошибка при запуске приложения: {e}")
        messagebox.showerror("Критическая ошибка", f"Приложение не может быть запущено:\n{e}")
                for row_idx, row in enumerate(buttons):
            for col_idx, digit in enumerate(row):
                btn = tk.Button(
                    keyboard_frame, 
                    text=digit, 
                    font=("Helvetica", 14),
                    command=lambda d=digit: self.press_digit(d)
                )
                btn.grid(row=row_idx, column=col_idx, sticky="nsew", padx=2, pady=2)

        self.create_back_button()

    def press_digit(self, digit):
        self.current_number += digit
        self.number_display.config(text=self.current_number)

