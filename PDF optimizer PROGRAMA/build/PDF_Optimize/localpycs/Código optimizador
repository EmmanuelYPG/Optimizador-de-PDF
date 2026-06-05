#!/usr/bin/env python3
"""
pdf_optimize_gui.py
───────────────────
Ejecutable que optimiza PDFs:
    1. Abre un diálogo para que el usuario seleccione uno o varios PDFs.
    2. Upscale de imágenes a 300 DPI con LANCZOS (sin modificar posición ni tamaño visual).
    3. Conversión a escala de grises.
    4. Control de tamaño máximo (3 MB por defecto).
    5. Abre un diálogo para que el usuario elija dónde guardar y con qué nombre.

Para generar el .exe:
    pip install pyinstaller PyMuPDF Pillow
    pyinstaller --onefile --windowed --name="PDF_Optimize" pdf_optimize_gui.py
"""

import sys
import os
import io
import tkinter as tk
from tkinter import filedialog, messagebox
import fitz  # PyMuPDF
from PIL import Image

# ──────────────────────────── CONFIGURACIÓN ────────────────────────────
TARGET_DPI = 300
MAX_FILE_SIZE_MB = 3.0
MAX_FILE_SIZE_BYTES = int(MAX_FILE_SIZE_MB * 1024 * 1024)
INITIAL_JPEG_QUALITY = 90
MIN_JPEG_QUALITY = 40
QUALITY_STEP = 5
# ───────────────────────────────────────────────────────────────────────


def analyze_image(doc, xref, page):
    """Extrae metadata de una imagen: dimensiones, DPI efectivo, colorspace."""
    base = doc.extract_image(xref)
    rects = page.get_image_rects(xref)
    rect = rects[0] if rects else None

    info = {
        "xref": xref,
        "ext": base["ext"],
        "width": base["width"],
        "height": base["height"],
        "cs_name": base.get("cs-name", "unknown"),
        "image_bytes": base["image"],
        "rect": rect,
        "dpi_x": None,
        "dpi_y": None,
    }

    if rect and rect.width > 0 and rect.height > 0:
        info["dpi_x"] = round(base["width"] / (rect.width / 72), 1)
        info["dpi_y"] = round(base["height"] / (rect.height / 72), 1)

    return info


def process_image(img_info, jpeg_quality):
    """
    Abre imagen con Pillow, upscale a 300 DPI si es necesario,
    convierte a escala de grises, retorna bytes JPEG.
    """
    pil_img = Image.open(io.BytesIO(img_info["image_bytes"]))

    rect = img_info["rect"]
    if rect and rect.width > 0 and rect.height > 0:
        target_w = int(round((rect.width / 72) * TARGET_DPI))
        target_h = int(round((rect.height / 72) * TARGET_DPI))
    else:
        current_dpi = img_info["dpi_x"] or 72
        scale = TARGET_DPI / current_dpi
        target_w = int(round(pil_img.width * scale))
        target_h = int(round(pil_img.height * scale))

    # Solo upscale; si ya supera 300 DPI, conservar dimensiones originales
    if pil_img.width >= target_w and pil_img.height >= target_h:
        target_w = pil_img.width
        target_h = pil_img.height

    if (target_w, target_h) != (pil_img.width, pil_img.height):
        pil_img = pil_img.resize((target_w, target_h), Image.LANCZOS)

    if pil_img.mode != "L":
        pil_img = pil_img.convert("L")

    buf = io.BytesIO()
    pil_img.save(buf, format="JPEG", quality=jpeg_quality, optimize=True)
    buf.seek(0)
    return buf.read(), target_w, target_h


def optimize_pdf(input_path, output_path, progress_callback=None):
    """Procesa un PDF completo: upscale + grayscale + límite de tamaño."""

    def log(msg):
        if progress_callback:
            progress_callback(msg)

    doc = fitz.open(input_path)
    file_name = os.path.basename(input_path)
    log(f"Procesando: {file_name}")
    log(f"Páginas: {len(doc)}  |  Original: {os.path.getsize(input_path):,} bytes")

    # ── Fase 1: inventario de imágenes únicas ──
    xref_map = {}
    for page_num in range(len(doc)):
        page = doc[page_num]
        for img_tuple in page.get_images(full=True):
            xref = img_tuple[0]
            if xref not in xref_map:
                try:
                    info = analyze_image(doc, xref, page)
                    info["first_page"] = page_num + 1
                    xref_map[xref] = info
                except Exception as e:
                    log(f"⚠ xref {xref} pág {page_num+1}: {e}")

    log(f"Imágenes únicas: {len(xref_map)}")

    # ── Fase 2: reemplazar imágenes, ajustar calidad hasta cumplir tamaño ──
    jpeg_quality = INITIAL_JPEG_QUALITY
    attempt = 0

    while True:
        attempt += 1
        doc_work = fitz.open(input_path)
        log(f"Intento {attempt}: JPEG quality={jpeg_quality}")

        for xref, info in xref_map.items():
            try:
                new_bytes, new_w, new_h = process_image(info, jpeg_quality)

                for page_num in range(len(doc_work)):
                    page = doc_work[page_num]
                    page_images = [img[0] for img in page.get_images(full=True)]
                    if xref in page_images:
                        page.replace_image(xref, filename=None, pixmap=None,
                                        stream=new_bytes)
                        break

                action = "upscaled" if (new_w > info["width"] or new_h > info["height"]) else "recompressed"
                dpi_before = info["dpi_x"] or "?"
                log(f"  xref={xref}: {info['width']}x{info['height']} → "
                    f"{new_w}x{new_h}  ({dpi_before}→{TARGET_DPI} DPI) [{action}]")

            except Exception as e:
                log(f"  xref={xref}: ERROR - {e}")

        doc_work.save(output_path, garbage=4, deflate=True, clean=True)
        doc_work.close()

        result_size = os.path.getsize(output_path)
        log(f"Resultado: {result_size:,} bytes ({result_size/1024:.1f} KB)")

        if result_size <= MAX_FILE_SIZE_BYTES:
            log(f"✓ Dentro del límite de {MAX_FILE_SIZE_MB} MB")
            break
        elif jpeg_quality <= MIN_JPEG_QUALITY:
            log(f"⚠ Calidad mínima alcanzada ({MIN_JPEG_QUALITY}), "
                f"archivo en {result_size/1024/1024:.2f} MB")
            break
        else:
            jpeg_quality -= QUALITY_STEP
            log(f"✗ Excede {MAX_FILE_SIZE_MB} MB → reduciendo quality a {jpeg_quality}")
            os.remove(output_path)

    doc.close()
    log(f"✓ Guardado: {output_path}")
    return output_path


# ═══════════════════════════════════════════════════════════════════════
#  GUI con Tkinter
# ═══════════════════════════════════════════════════════════════════════

class PDFOptimizerApp:
    def __init__(self, root):
        self.root = root
        self.root.title("PDF Optimize — 300 DPI + Grayscale")
        self.root.geometry("720x520")
        self.root.resizable(True, True)
        self.root.configure(bg="#1e1e2e")

        # ── Estilo ──
        self.BG = "#1e1e2e"
        self.FG = "#cdd6f4"
        self.ACCENT = "#89b4fa"
        self.BTN_BG = "#313244"
        self.BTN_ACTIVE = "#45475a"
        self.SUCCESS = "#a6e3a1"
        self.WARNING = "#f9e2af"
        self.FONT = ("Segoe UI", 10)
        self.FONT_BOLD = ("Segoe UI", 10, "bold")
        self.FONT_MONO = ("Consolas", 9)

        self._build_ui()

    def _build_ui(self):
        # ── Header ──
        header = tk.Frame(self.root, bg=self.BG)
        header.pack(fill="x", padx=20, pady=(18, 6))

        tk.Label(header, text="PDF Optimize", font=("Segoe UI", 16, "bold"),
                bg=self.BG, fg=self.ACCENT).pack(anchor="w")
        tk.Label(header, text="Upscale a 300 DPI  •  Escala de grises  •  Máx 3 MB",
                font=self.FONT, bg=self.BG, fg="#6c7086").pack(anchor="w")

        # ── Archivo seleccionado ──
        file_frame = tk.Frame(self.root, bg=self.BG)
        file_frame.pack(fill="x", padx=20, pady=(12, 4))

        tk.Label(file_frame, text="Archivo:", font=self.FONT_BOLD,
                bg=self.BG, fg=self.FG).pack(side="left")

        self.file_label = tk.Label(file_frame, text="(ninguno)",
                                font=self.FONT, bg=self.BG, fg="#6c7086",
                                anchor="w")
        self.file_label.pack(side="left", fill="x", expand=True, padx=(8, 0))

        # ── Botones ──
        btn_frame = tk.Frame(self.root, bg=self.BG)
        btn_frame.pack(fill="x", padx=20, pady=(8, 4))

        self.btn_select = tk.Button(
            btn_frame, text="1 · Seleccionar PDF",
            font=self.FONT_BOLD, bg=self.ACCENT, fg="#1e1e2e",
            activebackground="#b4d0fb", activeforeground="#1e1e2e",
            relief="flat", cursor="hand2", padx=16, pady=6,
            command=self._select_file
        )
        self.btn_select.pack(side="left", padx=(0, 10))

        self.btn_process = tk.Button(
            btn_frame, text="2 · Procesar y Guardar",
            font=self.FONT_BOLD, bg=self.BTN_BG, fg=self.FG,
            activebackground=self.BTN_ACTIVE, activeforeground=self.FG,
            relief="flat", cursor="hand2", padx=16, pady=6,
            state="disabled", command=self._process_file
        )
        self.btn_process.pack(side="left")

        # ── Log ──
        log_frame = tk.Frame(self.root, bg=self.BG)
        log_frame.pack(fill="both", expand=True, padx=20, pady=(12, 18))

        tk.Label(log_frame, text="Registro:", font=self.FONT_BOLD,
                bg=self.BG, fg=self.FG).pack(anchor="w", pady=(0, 4))

        self.log_text = tk.Text(
            log_frame, font=self.FONT_MONO, bg="#181825", fg=self.FG,
            insertbackground=self.FG, relief="flat", wrap="word",
            highlightthickness=1, highlightbackground="#313244",
            state="disabled"
        )
        self.log_text.pack(fill="both", expand=True)

        scrollbar = tk.Scrollbar(self.log_text, command=self.log_text.yview)
        scrollbar.pack(side="right", fill="y")
        self.log_text.configure(yscrollcommand=scrollbar.set)

        # ── Estado ──
        self.input_path = None

    def _log(self, msg):
        self.log_text.configure(state="normal")
        self.log_text.insert("end", msg + "\n")
        self.log_text.see("end")
        self.log_text.configure(state="disabled")
        self.root.update_idletasks()

    def _select_file(self):
        path = filedialog.askopenfilename(
            title="Selecciona el PDF a optimizar",
            filetypes=[("Archivos PDF", "*.pdf"), ("Todos los archivos", "*.*")]
        )
        if not path:
            return

        self.input_path = path
        display = path if len(path) < 70 else "..." + path[-67:]
        self.file_label.configure(text=display, fg=self.ACCENT)
        self.btn_process.configure(state="normal", bg=self.ACCENT, fg="#1e1e2e",
                                activebackground="#b4d0fb")
        self._log(f"Seleccionado: {os.path.basename(path)}")
        self._log(f"  Ruta: {path}")
        self._log(f"  Tamaño: {os.path.getsize(path):,} bytes")

    def _process_file(self):
        if not self.input_path:
            return

        # ── Sugerir nombre de salida ──
        base_name = os.path.basename(self.input_path)
        name_no_ext, ext = os.path.splitext(base_name)
        suggested_name = f"{name_no_ext}_optimized{ext}"
        initial_dir = os.path.dirname(self.input_path)

        output_path = filedialog.asksaveasfilename(
            title="Guardar PDF optimizado como...",
            initialdir=initial_dir,
            initialfile=suggested_name,
            defaultextension=".pdf",
            filetypes=[("Archivos PDF", "*.pdf")]
        )
        if not output_path:
            self._log("Operación cancelada por el usuario.")
            return

        # ── Deshabilitar botones durante el proceso ──
        self.btn_select.configure(state="disabled")
        self.btn_process.configure(state="disabled")
        self._log("")
        self._log("═" * 56)

        try:
            optimize_pdf(self.input_path, output_path, progress_callback=self._log)

            original_size = os.path.getsize(self.input_path)
            final_size = os.path.getsize(output_path)
            ratio = final_size / original_size if original_size > 0 else 0

            self._log("═" * 56)
            self._log(f"Original:   {original_size:>10,} bytes")
            self._log(f"Optimizado: {final_size:>10,} bytes  (×{ratio:.2f})")
            self._log("═" * 56)

            messagebox.showinfo(
                "Proceso completado",
                f"PDF guardado exitosamente.\n\n"
                f"Archivo: {os.path.basename(output_path)}\n"
                f"Tamaño: {final_size:,} bytes ({final_size/1024:.1f} KB)\n"
                f"Ubicación: {os.path.dirname(output_path)}"
            )

        except Exception as e:
            self._log(f"✗ ERROR: {e}")
            messagebox.showerror("Error", f"Ocurrió un error:\n\n{e}")

        finally:
            self.btn_select.configure(state="normal")
            self.btn_process.configure(state="normal")


def main():
    root = tk.Tk()

    # Icono y DPI awareness en Windows
    try:
        import ctypes
        ctypes.windll.shcore.SetProcessDpiAwareness(1)
    except Exception:
        pass

    app = PDFOptimizerApp(root)
    root.mainloop()


if __name__ == "__main__":
    main()
