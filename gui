# gui.py

import tkinter as tk
from tkinter import messagebox, simpledialog, scrolledtext, ttk
import threading
import time
import json
from datetime import datetime

from drive_service import (
    extract_file_id, extract_folder_id, list_files_in_folder,
    copy_file, move_file, delete_file, set_file_permission,
    get_file_hierarchy
)
from config import GOOGLE_ROOT_ID

# Файлы для хранения мониторинговых задач и журнала изменений
MONITOR_TASKS_FILE = "monitor_tasks.json"
CHANGES_LOG_FILE = "changes_log.json"

def load_json(filepath: str):
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return []

def save_json(filepath: str, data):
    try:
        with open(filepath, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"Ошибка записи файла {filepath}: {e}")

def add_change_record(operation, file_name, file_id, source_id="", dest_id="", comment=""):
    log_data = load_json(CHANGES_LOG_FILE)
    record = {
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "operation": operation,
        "file_name": file_name,
        "file_id": file_id,
        "source_folder_id": source_id,
        "dest_folder_id": dest_id,
        "comment": comment
    }
    log_data.append(record)
    save_json(CHANGES_LOG_FILE, log_data)

def add_monitor_task(source_id, dest_id, copied_files=None):
    if copied_files is None:
        copied_files = []
    tasks = load_json(MONITOR_TASKS_FILE)
    # Проверяем, существует ли такая задача
    for t in tasks:
        if t["source_folder_id"] == source_id and t["dest_folder_id"] == dest_id:
            return False
    tasks.append({
        "source_folder_id": source_id,
        "dest_folder_id": dest_id,
        "copied_files": copied_files
    })
    save_json(MONITOR_TASKS_FILE, tasks)
    return True

def get_monitor_tasks():
    return load_json(MONITOR_TASKS_FILE)

def check_monitor_tasks(log_callback):
    tasks = load_json(MONITOR_TASKS_FILE)
    updated = False
    for task in tasks:
        source_id = task["source_folder_id"]
        dest_id = task["dest_folder_id"]
        copied_files = task.get("copied_files", [])

        files = list_files_in_folder(source_id)
        for f in files:
            if f["name"] not in copied_files:
                try:
                    new_file = copy_file(f["id"], f["name"], dest_id)
                    copied_files.append(f["name"])
                    updated = True
                    log_callback(f"[Мониторинг] Скопирован новый файл: {f['name']}")
                    add_change_record("monitor_copy", new_file["name"], new_file["id"], source_id, dest_id,
                                      "Автоматическое копирование при мониторинге")
                except Exception as e:
                    log_callback(f"[Мониторинг] Ошибка копирования {f['name']}: {e}")

        task["copied_files"] = copied_files

    if updated:
        save_json(MONITOR_TASKS_FILE, tasks)

def monitor_worker(log_callback):
    """
    Фоновый поток мониторинга.
    Запускается при инициализации приложения и раз в 10 секунд проверяет задачи мониторинга.
    """
    while True:
        check_monitor_tasks(log_callback)
        time.sleep(10)  # Интервал проверки (10 секунд)

class DriveApp:
    def __init__(self, master):
        self.master = master
        master.title("Google Drive Manager")
        master.geometry("800x600")

        # Панель команд
        self.frame_commands = tk.Frame(master)
        self.frame_commands.pack(side=tk.TOP, fill=tk.X, padx=10, pady=10)

        # Набор кнопок
        commands = [
            ("Copy", self.copy_files),
            ("Monitor", self.show_monitor_tasks),
            ("AddMonitor", self.add_monitor_task_cmd),  # <-- Кнопка для добавления новой задачи
            ("Report", self.show_report),
            ("FolderReport", self.folder_report),
            ("Move", self.move_file),
            ("Delete", self.delete_file),
            ("SetPermissions", self.set_permissions),
            ("Cancel", self.cancel_operation)
        ]
        for (text, cmd) in commands:
            btn = tk.Button(self.frame_commands, text=text, width=12, command=cmd)
            btn.pack(side=tk.LEFT, padx=5)

        # Область вывода логов
        self.log_area = scrolledtext.ScrolledText(master, width=100, height=25)
        self.log_area.pack(padx=10, pady=10)

        # Запускаем фоновый поток мониторинга
        self.monitor_thread = threading.Thread(target=monitor_worker, args=(self.log,), daemon=True)
        self.monitor_thread.start()

    def log(self, message):
        self.log_area.insert(tk.END, message + "\n")
        self.log_area.see(tk.END)

    def cancel_operation(self):
        self.log("Операция отменена.")

    def copy_files(self):
        source_link = simpledialog.askstring("Copy", "Введите ссылку на исходную папку:")
        if source_link is None:
            return
        dest_link = simpledialog.askstring("Copy", "Введите ссылку на целевую папку (оставьте пустым для корневой):")
        if dest_link is None:
            return

        source_id = extract_folder_id(source_link)
        if not source_id:
            messagebox.showerror("Ошибка", "Не удалось извлечь ID исходной папки.")
            return

        if dest_link.strip() == "":
            dest_id = GOOGLE_ROOT_ID
        else:
            dest_id = extract_folder_id(dest_link)
            if not dest_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID целевой папки.")
                return

        files = list_files_in_folder(source_id)
        if not files:
            messagebox.showinfo("Copy", "В исходной папке нет файлов для копирования.")
            return

        total = len(files)
        copied_count = 0
        self.log(f"Начинается копирование {total} файлов...")
        for idx, f in enumerate(files, start=1):
            try:
                new_file = copy_file(f["id"], f["name"], dest_id)
                copied_count += 1
                add_change_record("copy", new_file["name"], new_file["id"], source_id, dest_id,
                                  "Ручное копирование через Copy")
                self.log(f"Копирован: {f['name']} ({idx}/{total})")
            except Exception as e:
                self.log(f"Ошибка копирования {f['name']}: {e}")

        # Создаём задачу мониторинга, если такой ещё нет
        tasks = get_monitor_tasks()
        exists = any(t["source_folder_id"] == source_id and t["dest_folder_id"] == dest_id for t in tasks)
        if not exists:
            add_monitor_task(source_id, dest_id, [f["name"] for f in files])
            self.log("Задача мониторинга создана.")

        self.log(f"Копирование завершено. Скопировано {copied_count} файлов.")

    def show_monitor_tasks(self):
        tasks = get_monitor_tasks()
        if not tasks:
            messagebox.showinfo("Monitor", "Нет мониторинговых задач.")
            return

        # Выводим задачи в новом окне с использованием Treeview
        win = tk.Toplevel(self.master)
        win.title("Мониторинговые задачи")
        tree = ttk.Treeview(win, columns=("Source", "Destination", "Copied"), show="headings")
        tree.heading("Source", text="Исходная папка")
        tree.heading("Destination", text="Целевая папка")
        tree.heading("Copied", text="Скопировано файлов")
        tree.pack(fill=tk.BOTH, expand=True)

        for task in tasks:
            tree.insert("", tk.END,
                        values=(
                            task["source_folder_id"],
                            task["dest_folder_id"],
                            len(task.get("copied_files", []))
                        ))

    def add_monitor_task_cmd(self):
        """
        Метод для ручного добавления новой задачи мониторинга.
        """
        source_link = simpledialog.askstring("AddMonitor", "Введите ссылку на исходную папку:")
        if source_link is None:
            return
        dest_link = simpledialog.askstring("AddMonitor",
                                           "Введите ссылку на целевую папку (оставьте пустым для корневой):")
        if dest_link is None:
            return

        source_id = extract_folder_id(source_link)
        if not source_id:
            messagebox.showerror("Ошибка", "Не удалось извлечь ID исходной папки.")
            return

        if dest_link.strip() == "":
            dest_id = GOOGLE_ROOT_ID
        else:
            dest_id = extract_folder_id(dest_link)
            if not dest_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID целевой папки.")
                return

        # Вызываем глобальную функцию add_monitor_task
        if add_monitor_task(source_id, dest_id):
            self.log(f"Новая задача мониторинга добавлена:\nИсходная: {source_id}\nЦелевая: {dest_id}")
        else:
            self.log("Такая задача уже существует!")

    def show_report(self):
        changes = load_json(CHANGES_LOG_FILE)
        report_lines = ["==== Полный отчёт по изменениям ===="]
        report_lines.append(f"Google Root ID: {GOOGLE_ROOT_ID}")
        report_lines.append("")

        if changes:
            for rec in sorted(changes, key=lambda x: x["timestamp"]):
                line = f"{rec['timestamp']} | {rec['operation']} | Файл: {rec['file_name']} (ID: {rec['file_id']})"
                if rec['source_folder_id']:
                    line += f" | От: {rec['source_folder_id']}"
                if rec['dest_folder_id']:
                    line += f" | В: {rec['dest_folder_id']}"
                if rec['comment']:
                    line += f" | {rec['comment']}"
                report_lines.append(line)
        else:
            report_lines.append("Журнал изменений пуст.")

        report = "\n".join(report_lines)

        # Выводим отчёт в новом окне
        win = tk.Toplevel(self.master)
        win.title("Отчёт изменений")
        txt = scrolledtext.ScrolledText(win, width=100, height=30)
        txt.insert(tk.END, report)
        txt.pack()

    def folder_report(self):
        link = simpledialog.askstring("FolderReport", "Введите ссылку на папку (или подпапку):")
        if not link:
            return
        folder_id = extract_folder_id(link)
        if not folder_id:
            messagebox.showerror("Ошибка", "Не удалось извлечь ID папки.")
            return

        report = get_file_hierarchy(folder_id)
        win = tk.Toplevel(self.master)
        win.title("Отчёт по папке")
        txt = scrolledtext.ScrolledText(win, width=100, height=30)
        txt.insert(tk.END, report)
        txt.pack()

    def move_file(self):
        file_link = simpledialog.askstring("Move", "Введите ссылку на файл (или папку):")
        if not file_link:
            return
        file_id = extract_file_id(file_link)
        if not file_id:
            file_id = extract_folder_id(file_link)
            if not file_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID файла/папки.")
                return

        dest_link = simpledialog.askstring("Move",
                                           "Введите ссылку на целевую папку (или оставьте пустым для корневой):")
        if dest_link is None:
            return

        if dest_link.strip() == "":
            dest_id = GOOGLE_ROOT_ID
        else:
            dest_id = extract_folder_id(dest_link)
            if not dest_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID целевой папки.")
                return

        try:
            updated = move_file(file_id, dest_id)
            self.log(f"Файл/папка успешно перемещён. Новые родительские папки: {updated.get('parents', [])}")
            messagebox.showinfo("Move", "Операция перемещения выполнена.")
            add_change_record("move", updated["name"], updated["id"], "(неизвестно)", dest_id, "Перемещение через Move")
        except Exception as e:
            self.log(f"Ошибка при перемещении: {e}")
            messagebox.showerror("Ошибка", "Ошибка при перемещении.")

    def delete_file(self):
        file_link = simpledialog.askstring("Delete", "Введите ссылку на файл (или папку) для удаления:")
        if not file_link:
            return
        file_id = extract_file_id(file_link)
        if not file_id:
            file_id = extract_folder_id(file_link)
            if not file_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID файла/папки.")
                return

        answer = messagebox.askyesno("Подтверждение", "Вы действительно хотите удалить этот файл/папку?")
        if not answer:
            self.log("Удаление отменено.")
            return

        try:
            delete_file(file_id)
            self.log("Файл/папка успешно удалены.")
            add_change_record("delete", "(unknown)", file_id, comment="Удаление через Delete")
            messagebox.showinfo("Delete", "Файл/папка удалены.")
        except Exception as e:
            self.log(f"Ошибка при удалении: {e}")
            messagebox.showerror("Ошибка", "Ошибка при удалении.")

    def set_permissions(self):
        file_link = simpledialog.askstring("SetPermissions", "Введите ссылку на файл (или папку):")
        if not file_link:
            return
        file_id = extract_file_id(file_link)
        if not file_id:
            file_id = extract_folder_id(file_link)
            if not file_id:
                messagebox.showerror("Ошибка", "Не удалось извлечь ID файла/папки.")
                return

        email = simpledialog.askstring("SetPermissions", "Введите email пользователя:")
        if not email:
            return

        role = simpledialog.askstring("SetPermissions", "Введите роль (reader, writer, owner):")
        if not role or role.lower() not in ['reader', 'writer', 'owner']:
            messagebox.showerror("Ошибка", "Некорректная роль.")
            return

        try:
            perm_id = set_file_permission(file_id, email, role.lower())
            self.log(f"Права успешно изменены. ID разрешения: {perm_id}")
            add_change_record("setpermissions", "(unknown)", file_id, comment=f"Права {role} для {email}")
            messagebox.showinfo("SetPermissions", "Права успешно изменены.")
        except Exception as e:
            self.log(f"Ошибка при изменении прав: {e}")
            messagebox.showerror("Ошибка", "Ошибка при изменении прав.")


if __name__ == '__main__':
    root = tk.Tk()
    app = DriveApp(root)
    root.mainloop()
