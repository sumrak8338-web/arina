import json
import random
import tkinter as tk
from tkinter import ttk, messagebox, filedialog
from datetime import datetime

class RandomTaskGenerator:
    def __init__(self, root):
        self.root = root
        self.root.title("Random Task Generator")
        self.root.geometry("700x550")
        
        # Предопределённые задачи (текст, тип)
        self.tasks_pool = [
            {"text": "Прочитать статью", "type": "учёба"},
            {"text": "Сделать зарядку", "type": "спорт"},
            {"text": "Написать отчёт", "type": "работа"},
            {"text": "Выучить 10 новых слов", "type": "учёба"},
            {"text": "Пробежка 5 км", "type": "спорт"},
            {"text": "Провести встречу", "type": "работа"},
            {"text": "Почитать книгу", "type": "учёба"},
            {"text": "Йога 20 минут", "type": "спорт"}
        ]
        
        # История сгенерированных задач (сохраняем с датой/временем)
        self.history = []   # каждый элемент: {"task": str, "type": str, "timestamp": str}
        
        # Файл для сохранения истории
        self.history_file = "task_history.json"
        
        # Загружаем историю из файла
        self.load_history()
        
        # Текущий фильтр (None = все)
        self.current_filter = None
        
        self.create_widgets()
        self.update_history_display()
    
    def create_widgets(self):
        # --- Панель добавления новой задачи ---
        add_frame = ttk.LabelFrame(self.root, text="Добавить новую задачу", padding=10)
        add_frame.pack(fill="x", padx=10, pady=5)
        
        ttk.Label(add_frame, text="Текст задачи:").grid(row=0, column=0, sticky="e", padx=5, pady=5)
        self.new_task_var = tk.StringVar()
        self.new_task_entry = ttk.Entry(add_frame, textvariable=self.new_task_var, width=40)
        self.new_task_entry.grid(row=0, column=1, padx=5, pady=5)
        
        ttk.Label(add_frame, text="Тип:").grid(row=0, column=2, sticky="e", padx=5, pady=5)
        self.new_type_var = tk.StringVar()
        self.type_combobox = ttk.Combobox(add_frame, textvariable=self.new_type_var, values=["учёба", "спорт", "работа"], width=10)
        self.type_combobox.grid(row=0, column=3, padx=5, pady=5)
        self.type_combobox.set("учёба")
        
        self.add_btn = ttk.Button(add_frame, text="Добавить задачу", command=self.add_task)
        self.add_btn.grid(row=0, column=4, padx=10, pady=5)
        
        # --- Панель генерации ---
        generate_frame = ttk.Frame(self.root)
        generate_frame.pack(fill="x", padx=10, pady=5)
        
        self.generate_btn = ttk.Button(generate_frame, text="Сгенерировать задачу", command=self.generate_task, width=25)
        self.generate_btn.pack(pady=10)
        
        # Отображение текущей сгенерированной задачи
        self.current_task_label = ttk.Label(generate_frame, text="", font=("Arial", 12, "bold"), foreground="blue")
        self.current_task_label.pack(pady=5)
        
        # --- Фильтрация истории ---
        filter_frame = ttk.LabelFrame(self.root, text="Фильтрация истории по типу", padding=5)
        filter_frame.pack(fill="x", padx=10, pady=5)
        
        ttk.Label(filter_frame, text="Тип:").pack(side="left", padx=5)
        self.filter_var = tk.StringVar()
        self.filter_combobox = ttk.Combobox(filter_frame, textvariable=self.filter_var, values=["Все", "учёба", "спорт", "работа"], width=10)
        self.filter_combobox.pack(side="left", padx=5)
        self.filter_combobox.set("Все")
        self.filter_combobox.bind("<<ComboboxSelected>>", self.on_filter_change)
        
        self.reset_filter_btn = ttk.Button(filter_frame, text="Сбросить", command=self.reset_filter)
        self.reset_filter_btn.pack(side="left", padx=5)
        
        # --- История задач ---
        history_frame = ttk.LabelFrame(self.root, text="История сгенерированных задач", padding=5)
        history_frame.pack(fill="both", expand=True, padx=10, pady=5)
        
        # Создаём Treeview для отображения истории
        columns = ("timestamp", "task", "type")
        self.tree = ttk.Treeview(history_frame, columns=columns, show="headings", height=12)
        self.tree.heading("timestamp", text="Дата/время")
        self.tree.heading("task", text="Задача")
        self.tree.heading("type", text="Тип")
        self.tree.column("timestamp", width=140)
        self.tree.column("task", width=300)
        self.tree.column("type", width=80)
        
        scrollbar = ttk.Scrollbar(history_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)
        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
        
        # --- Кнопки сохранения/загрузки ---
        control_frame = ttk.Frame(self.root)
        control_frame.pack(fill="x", padx=10, pady=10)
        
        self.save_btn = ttk.Button(control_frame, text="Сохранить историю в JSON", command=self.save_history_dialog)
        self.save_btn.pack(side="left", padx=5)
        
        self.load_btn = ttk.Button(control_frame, text="Загрузить историю из JSON", command=self.load_history_dialog)
        self.load_btn.pack(side="left", padx=5)
        
        self.clear_btn = ttk.Button(control_frame, text="Очистить историю", command=self.clear_history)
        self.clear_btn.pack(side="right", padx=5)
        
        # Статистика
        self.stats_label = ttk.Label(control_frame, text="", font=("Arial", 9))
        self.stats_label.pack(side="right", padx=10)
        self.update_stats()
    
    # --- Работа с задачами ---
    def add_task(self):
        """Добавление пользовательской задачи в пул"""
        task_text = self.new_task_var.get().strip()
        task_type = self.new_type_var.get().strip()
        
        if not task_text:
            messagebox.showerror("Ошибка", "Текст задачи не может быть пустым")
            return
        if not task_type:
            messagebox.showerror("Ошибка", "Выберите тип задачи")
            return
        
        # Добавляем в пул
        self.tasks_pool.append({"text": task_text, "type": task_type})
        self.new_task_var.set("")  # очищаем поле
        messagebox.showinfo("Успех", f"Задача '{task_text}' добавлена в общий список")
    
    def generate_task(self):
        """Выбирает случайную задачу из пула и сохраняет в историю"""
        if not self.tasks_pool:
            messagebox.showwarning("Предупреждение", "Нет доступных задач. Добавьте хотя бы одну.")
            return
        
        chosen = random.choice(self.tasks_pool)
        task_text = chosen["text"]
        task_type = chosen["type"]
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        # Добавляем в историю
        self.history.append({
            "task": task_text,
            "type": task_type,
            "timestamp": timestamp
        })
        
        # Отображаем текущую задачу
        self.current_task_label.config(text=f"Сгенерировано: {task_text} [{task_type}]")
        
        # Обновляем интерфейс и сохраняем историю
        self.update_history_display()
        self.save_history_auto()
        self.update_stats()
    
    # --- Фильтрация ---
    def on_filter_change(self, event=None):
        filt = self.filter_var.get()
        if filt == "Все":
            self.current_filter = None
        else:
            self.current_filter = filt
        self.update_history_display()
    
    def reset_filter(self):
        self.filter_var.set("Все")
        self.current_filter = None
        self.update_history_display()
    
    # --- Отображение истории ---
    def update_history_display(self):
        """Обновляет Treeview с учётом фильтра"""
        # Очищаем текущие строки
        for row in self.tree.get_children():
            self.tree.delete(row)
        
        # Применяем фильтр
        filtered = self.history
        if self.current_filter:
            filtered = [item for item in self.history if item["type"] == self.current_filter]
        
        # Вставляем в обратном порядке (новые сверху)
        for item in reversed(filtered):
            self.tree.insert("", "end", values=(
                item["timestamp"],
                item["task"],
                item["type"]
            ))
    
    def update_stats(self):
        total = len(self.history)
        self.stats_label.config(text=f"Всего сгенерировано: {total}")
    
    # --- Работа с JSON ---
    def save_history_auto(self):
        """Автоматическое сохранение в файл по умолчанию"""
        try:
            with open(self.history_file, "w", encoding="utf-8") as f:
                json.dump(self.history, f, ensure_ascii=False, indent=2)
        except Exception as e:
            print(f"Ошибка автосохранения: {e}")
    
    def save_history_dialog(self):
        """Сохранение в выбранный файл"""
        filename = filedialog.asksaveasfilename(
            defaultextension=".json",
            filetypes=[("JSON files", "*.json"), ("All files", "*.*")]
        )
        if filename:
            try:
                with open(filename, "w", encoding="utf-8") as f:
                    json.dump(self.history, f, ensure_ascii=False, indent=2)
                messagebox.showinfo("Успех", f"История сохранена в {filename}")
            except Exception as e:
                messagebox.showerror("Ошибка", f"Не удалось сохранить: {e}")
    
    def load_history(self):
        """Загрузка истории из файла по умолчанию"""
        try:
            with open(self.history_file, "r", encoding="utf-8") as f:
                data = json.load(f)
            if isinstance(data, list):
                self.history = data
        except FileNotFoundError:
            pass
        except Exception as e:
            messagebox.showerror("Ошибка", f"Ошибка загрузки истории: {e}")
    
    def load_history_dialog(self):
        """Загрузка истории из выбранного файла"""
        filename = filedialog.askopenfilename(
            filetypes=[("JSON files", "*.json"), ("All files", "*.*")]
        )
        if filename:
            try:
                with open(filename, "r", encoding="utf-8") as f:
                    data = json.load(f)
                if isinstance(data, list):
                    self.history = data
                    self.update_history_display()
                    self.save_history_auto()  # синхронизация с файлом по умолчанию
                    self.update_stats()
                    messagebox.showinfo("Успех", f"История загружена из {filename}")
                else:
                    messagebox.showwarning("Предупреждение", "Файл не содержит список истории")
            except Exception as e:
                messagebox.showerror("Ошибка", f"Не удалось загрузить: {e}")
    
    def clear_history(self):
        """Очистка всей истории"""
        if messagebox.askyesno("Подтверждение", "Вы уверены, что хотите очистить всю историю?"):
            self.history = []
            self.update_history_display()
            self.save_history_auto()
            self.update_stats()
            self.current_task_label.config(text="")
            messagebox.showinfo("Информация", "История очищена")

if __name__ == "__main__":
    root = tk.Tk()
    app = RandomTaskGenerator(root)
    root.mainloop()
