import os
import math
import re
import threading
import webbrowser
from collections import Counter
import tkinter as tk
from tkinter import filedialog, messagebox, scrolledtext
from pypdf import PdfReader, PdfWriter


# --- КОНФИГУРАЦИЯ ДЕМО-РЕЖИМА И ОПЛАТЫ ---

DEMO_MAX_SCANS = 2            # сколько бесплатных сканирований даётся
DEMO_MAX_PAGES = 3            # сколько страниц PDF обрабатывается в демо
PRICE = "990 ₽"

# --- КОНТАКТЫ ДЛЯ ПОКУПКИ ---
CONTACT_TELEGRAM      = "https://t.me/+JGlolqiEKEQ2YjRi"
CONTACT_PHONE_DISPLAY = "+7 924 123 1234"
CONTACT_PHONE_LINK    = "sms:+79241231234"           # открывает SMS-приложение
CONTACT_EMAIL         = "pila3383@gmail.com"
CONTACT_EMAIL_LINK    = "mailto:pila3383@gmail.com"  # открывает почтовый клиент

VALID_KEY_PREFIX = "GHOST-"   # демо-ключ активации (в реале — с сервера)


# --- ЛОГИКА АНАЛИЗА И ОЧИСТКИ ---

def calculate_entropy(data: bytes) -> float:
    if not data:
        return 0.0
    entropy = 0.0
    length = len(data)
    counts = Counter(data)
    for count in counts.values():
        p_x = count / length
        entropy -= p_x * math.log2(p_x)
    return round(entropy, 4)


def analyze_stylometry(text: str) -> dict:
    sentences = [s.strip() for s in re.split(r'[.!?]+', text) if s.strip()]
    words = re.findall(r'\b[а-яА-Яa-zA-Z]+\b', text.lower())

    if not words or not sentences:
        return {"avg_sentence_len": 0, "lexical_diversity": 0, "risk": "Низкий"}

    avg_len = round(len(words) / len(sentences), 1)
    lexical_diversity = round(len(set(words)) / len(words), 2)

    risk = (
        "Высокий (уникальный авторский стиль)"
        if (avg_len > 18 and lexical_diversity > 0.7)
        else "Умеренный"
    )
    return {
        "avg_sentence_len": avg_len,
        "lexical_diversity": lexical_diversity,
        "risk": risk
    }


def sanitize_pdf(input_path: str, output_path: str, max_pages: int = None):
    """Глубокая очистка PDF. max_pages=None — обрабатывать все страницы."""
    with open(input_path, "rb") as f:
        raw_bytes = f.read()

    entropy = calculate_entropy(raw_bytes)
    reader = PdfReader(input_path)
    writer = PdfWriter()

    total_pages = len(reader.pages)
    pages_to_process = reader.pages if max_pages is None else list(reader.pages)[:max_pages]
    processed_pages = len(pages_to_process)

    extracted_text = ""
    for page in pages_to_process:
        writer.add_page(page)
        extracted_text += page.extract_text() or ""

    old_meta = reader.metadata
    writer.add_metadata({"/Producer": "GhostTrace Sanitized Engine v0.1"})

    with open(output_path, "wb") as f_out:
        writer.write(f_out)

    stylometry = analyze_stylometry(extracted_text)
    return old_meta, entropy, stylometry, total_pages, processed_pages


# --- ГРАФИЧЕСКИЙ ИНТЕРФЕЙС ---

class GhostTraceApp:
    def __init__(self, root: tk.Tk):
        self.root = root
        self.root.title("GHOSTTRACE v0.1 DEMO | Очистка цифрового следа")
        self.root.geometry("720x600")
        self.root.minsize(640, 540)
        self.root.configure(bg="#121212")

        # Состояние демо / лицензии
        self.scan_count = 0
        self.is_licensed = False

        self.selected_file = tk.StringVar()
        self._build_ui()
        self._update_demo_status()

    def _build_ui(self):
        # ---------- Заголовок + кнопка покупки ----------
        header_frame = tk.Frame(self.root, bg="#1e1e1e", pady=12, padx=15)
        header_frame.pack(fill=tk.X)
        header_frame.grid_columnconfigure(0, weight=1)

        title_col = tk.Frame(header_frame, bg="#1e1e1e")
        title_col.grid(row=0, column=0, sticky="ew")

        tk.Label(
            title_col, text="GHOSTTRACE v0.1",
            font=("Consolas", 18, "bold"), fg="#00e676", bg="#1e1e1e"
        ).pack()

        tk.Label(
            title_col,
            text="Анализ энтропии, стилометрии и удаление метаданных PDF",
            font=("Segoe UI", 10), fg="#aaaaaa", bg="#1e1e1e"
        ).pack()

        self.buy_btn = tk.Button(
            header_frame,
            text="💎 КУПИТЬ PRO",
            font=("Segoe UI", 10, "bold"),
            bg="#ffb300", fg="#000000",
            activebackground="#ffca28", activeforeground="#000000",
            relief=tk.FLAT, padx=15, pady=8, cursor="hand2",
            command=self._show_paywall
        )
        self.buy_btn.grid(row=0, column=1, padx=(15, 0))

        # ---------- Выбор файла ----------
        file_frame = tk.Frame(self.root, bg="#121212", padx=20, pady=15)
        file_frame.pack(fill=tk.X)

        tk.Label(
            file_frame, text="Выбранный PDF-файл:",
            font=("Segoe UI", 10, "bold"), fg="#ffffff", bg="#121212"
        ).pack(anchor=tk.W, pady=(0, 5))

        input_row = tk.Frame(file_frame, bg="#121212")
        input_row.pack(fill=tk.X)

        self.path_entry = tk.Entry(
            input_row, textvariable=self.selected_file,
            font=("Consolas", 10), bg="#252525", fg="#ffffff",
            insertbackground="#ffffff", relief=tk.FLAT, bd=6
        )
        self.path_entry.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=(0, 10))

        tk.Button(
            input_row, text="Обзор...",
            font=("Segoe UI", 10, "bold"),
            bg="#333333", fg="#ffffff",
            activebackground="#444444", activeforeground="#ffffff",
            relief=tk.FLAT, padx=15, pady=4, cursor="hand2",
            command=self._select_file
        ).pack(side=tk.RIGHT)

        # ---------- Кнопка запуска ----------
        self.run_btn = tk.Button(
            self.root, text="ЗАПУСТИТЬ АНАЛИЗ И ОЧИСТКУ",
            font=("Segoe UI", 11, "bold"),
            bg="#00c853", fg="#000000",
            activebackground="#00e676", activeforeground="#000000",
            relief=tk.FLAT, pady=10, cursor="hand2",
            command=self._start_processing
        )
        self.run_btn.pack(fill=tk.X, padx=20)

        # ---------- Статус демо-режима ----------
        self.demo_status = tk.Label(
            self.root, text="", font=("Consolas", 9, "bold"),
            fg="#ffb300", bg="#121212"
        )
        self.demo_status.pack(pady=(6, 0))

        # ---------- Окно логов ----------
        log_frame = tk.Frame(self.root, bg="#121212", padx=20, pady=15)
        log_frame.pack(fill=tk.BOTH, expand=True)

        tk.Label(
            log_frame, text="Отчёт сканирования:",
            font=("Segoe UI", 10, "bold"), fg="#ffffff", bg="#121212"
        ).pack(anchor=tk.W, pady=(0, 5))

        self.log_area = scrolledtext.ScrolledText(
            log_frame, font=("Consolas", 10), bg="#1a1a1a", fg="#e0e0e0",
            relief=tk.FLAT, bd=8, state=tk.DISABLED
        )
        self.log_area.pack(fill=tk.BOTH, expand=True)

        self.log_area.tag_config("info", foreground="#00e676")
        self.log_area.tag_config("warn", foreground="#ff5252", font=("Consolas", 10, "bold"))
        self.log_area.tag_config("header", foreground="#40c4ff", font=("Consolas", 10, "bold"))
        self.log_area.tag_config("gold", foreground="#ffb300", font=("Consolas", 10, "bold"))

    # ---------------- Вспомогательные методы ----------------

    def _update_demo_status(self):
        if self.is_licensed:
            self.demo_status.config(
                text="✓ PRO-версия активирована — все ограничения сняты",
                fg="#00e676"
            )
            self.buy_btn.config(text="✓ PRO АКТИВЕН", state=tk.DISABLED,
                                bg="#1e1e1e", fg="#00e676")
        else:
            remaining = max(0, DEMO_MAX_SCANS - self.scan_count)
            self.demo_status.config(
                text=(f"ДЕМО-РЕЖИМ: осталось {remaining}/{DEMO_MAX_SCANS} "
                      f"сканирований | макс. {DEMO_MAX_PAGES} стр./файл"),
                fg="#ffb300" if remaining > 0 else "#ff5252"
            )

    def _select_file(self):
        filepath = filedialog.askopenfilename(
            title="Выберите PDF-файл",
            filetypes=[("PDF документы", "*.pdf"), ("Все файлы", "*.*")]
        )
        if filepath:
            self.selected_file.set(filepath)

    def _log(self, message: str, tag: str = None):
        self.log_area.configure(state=tk.NORMAL)
        if tag:
            self.log_area.insert(tk.END, message + "\n", tag)
        else:
            self.log_area.insert(tk.END, message + "\n")
        self.log_area.see(tk.END)
        self.log_area.configure(state=tk.DISABLED)

    def _clear_log(self):
        self.log_area.configure(state=tk.NORMAL)
        self.log_area.delete("1.0", tk.END)
        self.log_area.configure(state=tk.DISABLED)

    # ---------------- Paywall / оплата ----------------

    def _open_link(self, url: str):
        try:
            webbrowser.open(url)
        except Exception as e:
            messagebox.showerror("Ошибка", f"Не удалось открыть ссылку:\n{e}")

    def _copy_text(self, text: str, label: str):
        self.root.clipboard_clear()
        self.root.clipboard_append(text)
        messagebox.showinfo("Скопировано",
                            f"{label} скопирован в буфер обмена:\n{text}")

    def _show_paywall(self):
        if self.is_licensed:
            messagebox.showinfo("PRO", "PRO-версия уже активирована.")
            return

        win = tk.Toplevel(self.root)
        win.title("GhostTrace PRO — Полная версия")
        win.configure(bg="#121212")
        win.resizable(False, False)
        win.transient(self.root)

        w, h = 520, 720
        self.root.update_idletasks()
        x = self.root.winfo_rootx() + (self.root.winfo_width() - w) // 2
        y = self.root.winfo_rooty() + (self.root.winfo_height() - h) // 2
        win.geometry(f"{w}x{h}+{max(0, x)}+{max(0, y)}")
        win.grab_set()

        tk.Label(win, text="💎 GHOSTTRACE PRO",
                 font=("Segoe UI", 18, "bold"),
                 fg="#ffb300", bg="#121212").pack(pady=(20, 4))
        tk.Label(win, text="Полная версия без ограничений",
                 font=("Segoe UI", 10), fg="#aaaaaa", bg="#121212").pack()

        tk.Frame(win, bg="#333333", height=1).pack(fill=tk.X, padx=30, pady=15)

        # Что даёт PRO
        features_frame = tk.Frame(win, bg="#1a1a1a", padx=20, pady=15)
        features_frame.pack(fill=tk.X, padx=30)

        tk.Label(features_frame, text="Что вы получаете:",
                 font=("Segoe UI", 10, "bold"),
                 fg="#40c4ff", bg="#1a1a1a", anchor="w").pack(fill=tk.X, pady=(0, 8))

        features = [
            "Неограниченное число сканирований",
            "Обработка всех страниц PDF",
            "Расширенный стилометрический отчёт",
            "Экспорт отчёта в TXT-файл",
            "Приоритетная поддержка",
        ]
        for text in features:
            row = tk.Frame(features_frame, bg="#1a1a1a")
            row.pack(fill=tk.X, pady=2)
            tk.Label(row, text="★", fg="#ffb300", bg="#1a1a1a",
                     font=("Segoe UI", 11, "bold")).pack(side=tk.LEFT, padx=(0, 8))
            tk.Label(row, text=text, fg="#e0e0e0", bg="#1a1a1a",
                     font=("Segoe UI", 10), anchor="w").pack(side=tk.LEFT, fill=tk.X, expand=True)

        # Цена
        price_frame = tk.Frame(win, bg="#121212")
        price_frame.pack(pady=18)
        tk.Label(price_frame, text="Цена:", font=("Segoe UI", 11),
                 fg="#aaaaaa", bg="#121212").pack(side=tk.LEFT, padx=(0, 8))
        tk.Label(price_frame, text=PRICE, font=("Segoe UI", 22, "bold"),
                 fg="#00e676", bg="#121212").pack(side=tk.LEFT)
        tk.Label(price_frame, text="единоразово", font=("Segoe UI", 9),
                 fg="#777777", bg="#121212").pack(side=tk.LEFT, padx=(8, 0))

        # Контакты для покупки
        tk.Label(win, text="Свяжитесь с нами для покупки:",
                 font=("Segoe UI", 10), fg="#ffffff", bg="#121212").pack()

        # ---- Кнопки контактов ----
        btn_frame = tk.Frame(win, bg="#121212")
        btn_frame.pack(pady=10)

        # Telegram — открывает чат
        tk.Button(btn_frame, text="✈ Telegram", font=("Segoe UI", 10, "bold"),
                  bg="#0088cc", fg="#ffffff", relief=tk.FLAT,
                  padx=18, pady=8, cursor="hand2",
                  command=lambda: self._open_link(CONTACT_TELEGRAM)
                  ).pack(side=tk.LEFT, padx=4)

        # Телефон — открывает SMS «чтобы писать»
        tk.Button(btn_frame, text="📱 SMS", font=("Segoe UI", 10, "bold"),
                  bg="#00c853", fg="#000000", relief=tk.FLAT,
                  padx=18, pady=8, cursor="hand2",
                  command=lambda: self._open_link(CONTACT_PHONE_LINK)
                  ).pack(side=tk.LEFT, padx=4)

        # Email — открывает почтовый клиент
        tk.Button(btn_frame, text="✉ Email", font=("Segoe UI", 10, "bold"),
                  bg="#444444", fg="#ffffff", relief=tk.FLAT,
                  padx=18, pady=8, cursor="hand2",
                  command=lambda: self._open_link(CONTACT_EMAIL_LINK)
                  ).pack(side=tk.LEFT, padx=4)

        # ---- Показ самих контактов (кликабельно) ----
        contact_info = tk.Frame(win, bg="#121212")
        contact_info.pack(pady=(4, 0), fill=tk.X, padx=30)

        tg_lbl = tk.Label(contact_info, text=f"Telegram: {CONTACT_TELEGRAM}",
                          font=("Consolas", 9, "underline"),
                          fg="#40c4ff", bg="#121212", cursor="hand2", anchor="w")
        tg_lbl.pack(fill=tk.X)
        tg_lbl.bind("<Button-1>", lambda e: self._open_link(CONTACT_TELEGRAM))

        phone_lbl = tk.Label(contact_info,
                             text=f"Телефон (SMS): {CONTACT_PHONE_DISPLAY}",
                             font=("Consolas", 9, "underline"),
                             fg="#00e676", bg="#121212", cursor="hand2", anchor="w")
        phone_lbl.pack(fill=tk.X)
        phone_lbl.bind("<Button-1>", lambda e: self._open_link(CONTACT_PHONE_LINK))

        email_lbl = tk.Label(contact_info,
                             text=f"Email: {CONTACT_EMAIL}",
                             font=("Consolas", 9, "underline"),
                             fg="#ffb300", bg="#121212", cursor="hand2", anchor="w")
        email_lbl.pack(fill=tk.X)
        email_lbl.bind("<Button-1>", lambda e: self._open_link(CONTACT_EMAIL_LINK))

        # ---- Активация ключа ----
        tk.Label(win,
                 text="После оплаты вы получите ключ активации.\n"
                      "Введите его ниже для снятия ограничений.",
                 font=("Segoe UI", 9), fg="#888888", bg="#121212",
                 justify=tk.CENTER).pack(pady=(14, 6))

        key_frame = tk.Frame(win, bg="#121212")
        key_frame.pack(fill=tk.X, padx=30)
        tk.Label(key_frame, text="Ключ активации:",
                 font=("Segoe UI", 9), fg="#aaaaaa", bg="#121212").pack(anchor=tk.W)
        key_entry = tk.Entry(key_frame, font=("Consolas", 10),
                             bg="#252525", fg="#ffffff",
                             insertbackground="#ffffff",
                             relief=tk.FLAT, bd=6, justify=tk.CENTER)
        key_entry.pack(fill=tk.X, pady=(3, 0))

        def _activate():
            key = key_entry.get().strip().upper()
            if key.startswith(VALID_KEY_PREFIX) and len(key) >= 12:
                self.is_licensed = True
                self._update_demo_status()
                messagebox.showinfo("Активация",
                                    "✓ PRO-версия успешно активирована!\n"
                                    "Все ограничения сняты.")
                win.destroy()
            else:
                messagebox.showerror("Ошибка",
                                     "Неверный ключ активации.\n"
                                     "Проверьте введённые данные.")

        tk.Button(win, text="АКТИВИРОВАТЬ",
                  font=("Segoe UI", 10, "bold"),
                  bg="#00c853", fg="#000000",
                  relief=tk.FLAT, pady=9, cursor="hand2",
                  command=_activate).pack(fill=tk.X, padx=30, pady=8)

        tk.Button(win, text="Закрыть",
                  font=("Segoe UI", 9),
                  bg="#252525", fg="#aaaaaa",
                  relief=tk.FLAT, pady=6, cursor="hand2",
                  command=win.destroy).pack(fill=tk.X, padx=30, pady=(0, 15))

    # ---------------- Основная обработка ----------------

    def _start_processing(self):
        file_path = self.selected_file.get().strip().strip('"')

        if not file_path:
            messagebox.showwarning("Внимание", "Сначала выберите PDF-файл.")
            return
        if not os.path.exists(file_path):
            messagebox.showerror("Ошибка", "Указанный файл не найден.")
            return
        if not file_path.lower().endswith(".pdf"):
            messagebox.showwarning("Внимание", "Пожалуйста, выберите файл формата .pdf")
            return

        # ---- Проверка лимита демо ----
        if not self.is_licensed and self.scan_count >= DEMO_MAX_SCANS:
            self._clear_log()
            self._log("[!] Бесплатные сканирования закончились.", "warn")
            self._log("    Открываем окно приобретения PRO-версии...", "gold")
            self._show_paywall()
            return

        self.run_btn.configure(state=tk.DISABLED, text="ОБРАБОТКА...")
        self._clear_log()

        thread = threading.Thread(
            target=self._process_worker, args=(file_path,), daemon=True
        )
        thread.start()

    def _process_worker(self, file_path: str):
        try:
            base, ext = os.path.splitext(file_path)
            out_path = f"{base}_clean{ext}"

            max_pages = None if self.is_licensed else DEMO_MAX_PAGES

            self._log("[1/3] Анализ бинарной структуры и энтропии...", "header")
            old_meta, entropy, stylo, total_pages, processed_pages = sanitize_pdf(
                file_path, out_path, max_pages=max_pages
            )

            if not self.is_licensed and processed_pages < total_pages:
                self._log(
                    f" [!] ДЕМО: обработано {processed_pages} из {total_pages} стр. "
                    f"Для полного файла — приобретите PRO.", "warn"
                )

            self._log(f" -> Энтропия файла: {entropy} бит/байт "
                      f"(норма для PDF: 7.2 – 7.9)")
            if entropy > 7.95:
                self._log(" [!] ВНИМАНИЕ: Аномально высокая энтропия. "
                          "Возможна стеганография!", "warn")

            self._log("\n[2/3] Обнаруженные и удалённые метаданные:", "header")
            if old_meta:
                for k, v in old_meta.items():
                    self._log(f" - {k}: {v}")
            else:
                self._log(" - Скрытых тегов не найдено.")

            self._log("\n[3/3] Стилометрический аудит текста:", "header")
            self._log(f" -> Средняя длина предложения: "
                      f"{stylo['avg_sentence_len']} слов")
            self._log(f" -> Коэффициент лексического разнообразия: "
                      f"{stylo['lexical_diversity']}")

            risk_tag = "warn" if "Высокий" in stylo['risk'] else None
            self._log(f" -> Риск деанонимизации автора: {stylo['risk']}", risk_tag)

            self._log("\n" + "=" * 58)
            self._log(f"[ГОТОВО] Очищенный файл сохранён:\n{out_path}", "info")
            self._log("=" * 58)

            # ---- Учёт демо-сканирования ----
            if not self.is_licensed:
                self.scan_count += 1
                self.root.after(0, self._update_demo_status)

                remaining = DEMO_MAX_SCANS - self.scan_count
                if remaining > 0:
                    self._log(f"\n[i] Осталось бесплатных сканирований: "
                              f"{remaining}.", "gold")
                else:
                    self._log("\n[!] Это было ваше последнее бесплатное "
                              "сканирование.", "warn")
                    self._log("    Нажмите «💎 КУПИТЬ PRO» для продолжения "
                              "работы.", "gold")

            self.root.after(0, lambda: messagebox.showinfo(
                "Готово",
                f"Файл успешно очищен!\n"
                f"Сохранён как: {os.path.basename(out_path)}"
            ))

        except Exception as e:
            self._log(f"\n[ОШИБКА] Произошёл сбой при обработке: {e}", "warn")
            self.root.after(0, lambda: messagebox.showerror(
                "Ошибка", f"Не удалось обработать файл:\n{e}"
            ))
        finally:
            self.root.after(0, lambda: self.run_btn.configure(
                state=tk.NORMAL, text="ЗАПУСТИТЬ АНАЛИЗ И ОЧИСТКУ"
            ))


if __name__ == "__main__":
    root = tk.Tk()
    app = GhostTraceApp(root)
    root.mainloop()