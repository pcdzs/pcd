import tkinter as tk
from tkinter import ttk
import threading
import time
import pyautogui
import mss
import numpy as np
import cv2
import pytesseract
from PIL import Image, ImageTk, ImageDraw
import re

# ============== CONFIGURAÇÕES ==============
INTERVALO_VERIFICACAO = 1.5          # segundos entre cada captura
TEMPO_APOS_DETECTAR = 2.0            # espera antes de marcar
COR_MARCACAO = (0, 255, 0)           # verde (BGR)
ESPESSURA = 4
REGIAO_MONITORAMENTO = None         # None = tela inteira. Ex: (x, y, largura, altura)

# Caminho do Tesseract (ajuste se necessário)
# pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

class OverlayApp:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Quiz Helper")
        self.root.attributes("-topmost", True)
        self.root.attributes("-alpha", 0.92)
        self.root.overrideredirect(True)          # remove barra de título
        self.root.geometry("220x90+100+100")

        self.running = False
        self.thread = None
        self.last_question = ""

        # Frame principal (arrastável)
        self.frame = tk.Frame(self.root, bg="#1e1e2e", bd=2, relief="raised")
        self.frame.pack(fill="both", expand=True)

        self.label = tk.Label(self.frame, text="Parado", bg="#1e1e2e", fg="#cdd6f4",
                              font=("Segoe UI", 10, "bold"))
        self.label.pack(pady=(8, 4))

        self.btn = tk.Button(self.frame, text="▶ Iniciar", command=self.toggle,
                             bg="#89b4fa", fg="#1e1e2e", font=("Segoe UI", 11, "bold"),
                             relief="flat", padx=12, pady=4, cursor="hand2")
        self.btn.pack(pady=4)

        # Arrastar a janela
        self.frame.bind("<Button-1>", self.start_move)
        self.frame.bind("<B1-Motion>", self.do_move)
        self.label.bind("<Button-1>", self.start_move)
        self.label.bind("<B1-Motion>", self.do_move)

        # Overlay transparente para desenhar o retângulo
        self.overlay = None

    def start_move(self, event):
        self.x = event.x
        self.y = event.y

    def do_move(self, event):
        x = self.root.winfo_x() + event.x - self.x
        y = self.root.winfo_y() + event.y - self.y
        self.root.geometry(f"+{x}+{y}")

    def toggle(self):
        if self.running:
            self.running = False
            self.btn.config(text="▶ Iniciar", bg="#89b4fa")
            self.label.config(text="Parado")
            if self.overlay:
                self.overlay.destroy()
                self.overlay = None
        else:
            self.running = True
            self.btn.config(text="⏹ Parar", bg="#f38ba8")
            self.label.config(text="Monitorando...")
            self.thread = threading.Thread(target=self.loop, daemon=True)
            self.thread.start()

    def capturar_tela(self):
        with mss.mss() as sct:
            if REGIAO_MONITORAMENTO:
                mon = {"left": REGIAO_MONITORAMENTO[0],
                       "top": REGIAO_MONITORAMENTO[1],
                       "width": REGIAO_MONITORAMENTO[2],
                       "height": REGIAO_MONITORAMENTO[3]}
            else:
                mon = sct.monitors[1]  # monitor principal
            img = np.array(sct.grab(mon))
            return cv2.cvtColor(img, cv2.COLOR_BGRA2BGR)

    def extrair_texto(self, img):
        # Pré-processamento simples
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        gray = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)[1]
        texto = pytesseract.image_to_string(gray, lang="por")
        return texto.strip()

    def parece_pergunta(self, texto):
        # Heurística simples: tem "?" e opções A) B) C) D) ou 1. 2. 3. 4.
        if "?" not in texto:
            return False
        opcoes = re.findall(r'[A-D][\).]|[1-4][\).]', texto, re.IGNORECASE)
        return len(opcoes) >= 2

    def escolher_resposta(self, texto):
        """
        AQUI é o ponto que você decide a lógica.
        Por enquanto ele só escolhe a primeira opção encontrada
        (só para demonstração).

        Substitua por:
        - chamada a uma API de IA
        - banco de dados de questões
        - ou qualquer outra lógica sua
        """
        linhas = [l.strip() for l in texto.split("\n") if l.strip()]
        for i, linha in enumerate(linhas):
            if re.match(r'^[A-D1-4][\).]', linha, re.IGNORECASE):
                return i, linha   # retorna índice aproximado e o texto
        return None, None

    def desenhar_marcacao(self, img, texto_completo):
        # Versão bem simplificada: desenha um retângulo no meio da tela
        # (você pode melhorar depois com detecção de posição real das opções)
        h, w = img.shape[:2]
        x1, y1 = int(w * 0.15), int(h * 0.45)
        x2, y2 = int(w * 0.85), int(h * 0.55)

        cv2.rectangle(img, (x1, y1), (x2, y2), COR_MARCACAO, ESPESSURA)
        return img

    def mostrar_overlay(self, img_marcada):
        if self.overlay:
            self.overlay.destroy()

        self.overlay = tk.Toplevel(self.root)
        self.overlay.attributes("-topmost", True)
        self.overlay.attributes("-transparentcolor", "black")
        self.overlay.overrideredirect(True)

        # Converte para PhotoImage
        img_rgb = cv2.cvtColor(img_marcada, cv2.COLOR_BGR2RGB)
        pil = Image.fromarray(img_rgb)
        # Redimensiona se for muito grande (opcional)
        screen_w = self.root.winfo_screenwidth()
        screen_h = self.root.winfo_screenheight()
        pil = pil.resize((screen_w, screen_h), Image.LANCZOS)

        self.photo = ImageTk.PhotoImage(pil)
        label = tk.Label(self.overlay, image=self.photo, bg="black")
        label.pack()
        self.overlay.geometry(f"{screen_w}x{screen_h}+0+0")

        # Some depois de 4 segundos
        self.overlay.after(4000, self.overlay.destroy)

    def loop(self):
        while self.running:
            try:
                img = self.capturar_tela()
                texto = self.extrair_texto(img)

                if self.parece_pergunta(texto) and texto != self.last_question:
                    self.last_question = texto
                    self.label.config(text="Pergunta detectada!")
                    time.sleep(TEMPO_APOS_DETECTAR)

                    if not self.running:
                        break

                    idx, opcao = self.escolher_resposta(texto)
                    img_marcada = self.desenhar_marcacao(img, texto)
                    self.mostrar_overlay(img_marcada)
                    self.label.config(text="Marcado!")

                time.sleep(INTERVALO_VERIFICACAO)
            except Exception as e:
                print("Erro:", e)
                time.sleep(2)

    def run(self):
        self.root.mainloop()

if __name__ == "__main__":
    app = OverlayApp()
    app.run()
