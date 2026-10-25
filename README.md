# PNG Sequence Optimizer

PNG Sequence Optimizer is a desktop graphical tool built with Python to batch-compress sequential image sequences (such as broadcast lower-thirds, screen overlays, HUDs, and 2D sprite sheets) as well as standalone PNG assets.

The application leverages **Pngquant** (lossy palette quantization with alpha transparency) and **OxiPNG** (lossless stream optimization and chunk stripping) to significantly reduce storage footprint without sacrificing visual quality or transparency.

---

## Features

- **Dual Compression Pipelines:**
  - **Pngquant:** Quantizes 24/32-bit RGBA imagery down to 8-bit indexed palettes while maintaining an alpha channel.
  - **OxiPNG:** Performs lossless compression, restructuring IDAT chunks and stripping redundant metadata.
- **Engine Selection:**
  - `Both (Pngquant + OxiPNG)`: Maximum size reduction.
  - `Pngquant Only`: Fast palette quantization.
  - `OxiPNG Only`: Lossless size optimization without touching pixel colors.
- **Output Targets:**
  - **Overwrite Original Files:** Replaces source frames directly in place.
  - **Save to New Destination:** Copies and processes files into an isolated directory, leaving original renders intact.
- **Non-blocking Multi-threaded Architecture:** Runs intensive operations in background threads with thread-safe UI progress tracking.
- **Single-file Executable Deployment:** Compatible with PyInstaller bundling for standalone `.exe` distributions.

---

## Requirements

- Python 3.10 or higher
- `pngquant` binary
- `oxipng` binary

### Tool Installation (Windows)

Using Windows Package Manager:

```cmd
winget install oxipng
```

To install `pngquant` manually into Windows Apps:

```cmd
curl -Lo pngquant.zip https://pngquant.org/pngquant-windows.zip
tar -xf pngquant.zip
move pngquant\pngquant.exe %LOCALAPPDATA%\Microsoft\WindowsApps\
rd /s /q pngquant
del pngquant.zip
```

---

## Python Application Source (`compress_gui.py`)

Save the following code as `compress_gui.py`:

```python
import os
import shutil
import subprocess
import sys
import threading
import tkinter as tk
from tkinter import filedialog, messagebox, ttk
import webbrowser


def get_tool_path(tool_name):
  """Locates required binary tools from PyInstaller bundle, local folder, or PATH."""
  if hasattr(sys, "_MEIPASS"):
    bundled_path = os.path.join(sys._MEIPASS, tool_name)
    if os.path.exists(bundled_path):
      return bundled_path

  local_path = os.path.join(
      os.path.dirname(os.path.abspath(__file__)), tool_name
  )
  if os.path.exists(local_path):
    return local_path

  return tool_name


class PNGCompressorGUI:

  def __init__(self, root):
    self.root = root
    self.root.title("PNG Sequence Optimizer")
    self.root.geometry("560x620")
    self.root.resizable(False, False)

    icon_path = get_tool_path("icon.ico")
    if os.path.exists(icon_path):
      try:
        self.root.iconbitmap(icon_path)
      except Exception:
        pass

    self.source_folder = tk.StringVar()
    self.dest_folder = tk.StringVar()
    self.save_mode = tk.StringVar(value="overwrite")
    self.tool_mode = tk.StringVar(value="Both (Pngquant + OxiPNG)")

    self.min_q = tk.IntVar(value=65)
    self.max_q = tk.IntVar(value=85)
    self.oxi_level = tk.IntVar(value=4)

    self._build_ui()

  def _build_ui(self):
    # Folder Settings Frame
    f_frame = ttk.LabelFrame(self.root, text=" Folder Settings ", padding=10)
    f_frame.pack(fill="x", padx=15, pady=8)

    ttk.Label(
        f_frame, text="Source Folder:", font=("Segoe UI", 9, "bold")
    ).pack(anchor="w")
    s_box = ttk.Frame(f_frame)
    s_box.pack(fill="x", pady=(2, 8))
    ttk.Entry(s_box, textvariable=self.source_folder, width=48).pack(
        side="left", padx=(0, 5)
    )
    ttk.Button(s_box, text="Browse", command=self.browse_source).pack(
        side="left"
    )

    ttk.Label(
        f_frame, text="Save Mode / Destination:", font=("Segoe UI", 9, "bold")
    ).pack(anchor="w")
    radio_box = ttk.Frame(f_frame)
    radio_box.pack(fill="x", pady=2)
    ttk.Radiobutton(
        radio_box,
        text="Overwrite Original Files",
        variable=self.save_mode,
        value="overwrite",
        command=self.toggle_dest_state,
    ).pack(side="left", padx=(0, 15))
    ttk.Radiobutton(
        radio_box,
        text="Save to New Destination",
        variable=self.save_mode,
        value="destination",
        command=self.toggle_dest_state,
    ).pack(side="left")

    self.d_box = ttk.Frame(f_frame)
    self.d_box.pack(fill="x", pady=(5, 0))
    self.dest_entry = ttk.Entry(
        self.d_box, textvariable=self.dest_folder, width=48, state="disabled"
    )
    self.dest_entry.pack(side="left", padx=(0, 5))
    self.dest_btn = ttk.Button(
        self.d_box, text="Browse", command=self.browse_dest, state="disabled"
    )
    self.dest_btn.pack(side="left")

    # Compression Settings Frame
    s_frame = ttk.LabelFrame(
        self.root, text=" Compression Settings ", padding=10
    )
    s_frame.pack(fill="x", padx=15, pady=6)

    ttk.Label(
        s_frame, text="Compression Engine:", font=("Segoe UI", 9, "bold")
    ).pack(anchor="w")
    self.engine_combo = ttk.Combobox(
        s_frame,
        textvariable=self.tool_mode,
        state="readonly",
        values=[
            "Both (Pngquant + OxiPNG)",
            "Pngquant Only",
            "OxiPNG Only",
        ],
    )
    self.engine_combo.pack(fill="x", pady=(2, 10))
    self.engine_combo.bind("<<ComboboxSelected>>", self.toggle_slider_states)

    self.lbl_pq_title = ttk.Label(
        s_frame,
        text="Pngquant Quality (Min - Max):",
        font=("Segoe UI", 9, "bold"),
    )
    self.lbl_pq_title.pack(anchor="w")

    self.q_box = ttk.Frame(s_frame)
    self.q_box.pack(fill="x", pady=2)
    self.lbl_min_tag = ttk.Label(self.q_box, text="Min:")
    self.lbl_min_tag.pack(side="left")
    self.scale_min = ttk.Scale(
        self.q_box,
        from_=10,
        to=100,
        variable=self.min_q,
        command=lambda v: self.lbl_min.config(text=f"{int(float(v))}%"),
    )
    self.scale_min.pack(side="left", fill="x", expand=True, padx=5)
    self.lbl_min = ttk.Label(self.q_box, text="65%", width=5)
    self.lbl_min.pack(side="left")

    self.lbl_max_tag = ttk.Label(self.q_box, text="Max:")
    self.lbl_max_tag.pack(side="left", padx=(10, 0))
    self.scale_max = ttk.Scale(
        self.q_box,
        from_=10,
        to=100,
        variable=self.max_q,
        command=lambda v: self.lbl_max.config(text=f"{int(float(v))}%"),
    )
    self.scale_max.pack(side="left", fill="x", expand=True, padx=5)
    self.lbl_max = ttk.Label(self.q_box, text="85%", width=5)
    self.lbl_max.pack(side="left")

    self.lbl_oxi_title = ttk.Label(
        s_frame, text="OxiPNG Level (1 to 6):", font=("Segoe UI", 9, "bold")
    )
    self.lbl_oxi_title.pack(anchor="w", pady=(8, 0))

    self.oxi_box = ttk.Frame(s_frame)
    self.oxi_box.pack(fill="x", pady=2)
    self.scale_oxi = ttk.Scale(
        self.oxi_box,
        from_=1,
        to=6,
        variable=self.oxi_level,
        command=lambda v: self.lbl_oxi.config(text=str(int(float(v)))),
    )
    self.scale_oxi.pack(side="left", fill="x", expand=True, padx=5)
    self.lbl_oxi = ttk.Label(self.oxi_box, text="4", width=5)
    self.lbl_oxi.pack(side="left")

    # Progress Bar & Status
    self.prog_bar = ttk.Progressbar(self.root, mode="determinate")
    self.prog_bar.pack(fill="x", padx=15, pady=(10, 4))

    self.status_lbl = ttk.Label(
        self.root, text="Ready", font=("Segoe UI", 9), foreground="gray"
    )
    self.status_lbl.pack(anchor="w", padx=15)

    # Action Buttons
    btn_frame = ttk.Frame(self.root)
    btn_frame.pack(fill="x", padx=15, pady=10)

    self.btn_run = ttk.Button(
        btn_frame, text="Start Optimization", command=self.start_process_thread
    )
    self.btn_run.pack(side="left", fill="x", expand=True, padx=(0, 8))

    self.btn_about = ttk.Button(
        btn_frame, text="About", width=10, command=self.show_about
    )
    self.btn_about.pack(side="right")

    # Footer
    footer_frame = ttk.Frame(self.root)
    footer_frame.pack(side="bottom", pady=8)

    dev_label = ttk.Label(
        footer_frame,
        text="Developer: Shams Sohag | ",
        font=("Segoe UI", 8),
        foreground="#555",
    )
    dev_label.pack(side="left")

    github_label = ttk.Label(
        footer_frame,
        text="GitHub",
        font=("Segoe UI", 8, "underline"),
        foreground="#0066cc",
        cursor="hand2",
    )
    github_label.pack(side="left")
    github_label.bind(
        "<Button-1>", lambda e: webbrowser.open("https://github.com/shamssohag")
    )

  def show_about(self):
    about_text = (
        "PNG Sequence Optimizer\n\n"
        "A lightweight desktop GUI utility to compress PNG sequences and images "
        "with alpha channel preservation.\n\n"
        "Developer: Shams Sohag\n"
        "GitHub: https://github.com/shamssohag\n\n"
        "Powered by:\n"
        "- Pngquant (Quantization Engine)\n"
        "- OxiPNG (Lossless Optimizer)\n"
        "- Icon by Magnific (Flaticon)"
    )
    messagebox.showinfo("About PNG Optimizer", about_text)

  def toggle_dest_state(self):
    if self.save_mode.get() == "destination":
      self.dest_entry.config(state="normal")
      self.dest_btn.config(state="normal")
    else:
      self.dest_entry.config(state="disabled")
      self.dest_btn.config(state="disabled")

  def toggle_slider_states(self, event=None):
    mode = self.tool_mode.get()
    pq_state = (
        "normal"
        if mode in ["Both (Pngquant + OxiPNG)", "Pngquant Only"]
        else "disabled"
    )
    oxi_state = (
        "normal"
        if mode in ["Both (Pngquant + OxiPNG)", "OxiPNG Only"]
        else "disabled"
    )

    self.scale_min.config(state=pq_state)
    self.scale_max.config(state=pq_state)
    self.scale_oxi.config(state=oxi_state)

  def browse_source(self):
    p = filedialog.askdirectory()
    if p:
      self.source_folder.set(p)

  def browse_dest(self):
    p = filedialog.askdirectory()
    if p:
      self.dest_folder.set(p)

  def update_ui_progress(self, current, total, filename):
    self.prog_bar["value"] = current
    self.status_lbl.config(
        text=f"Processing ({current}/{total}): {filename}", foreground="black"
    )

  def finish_process(self, message, is_error=False):
    self.btn_run.config(state="normal")
    if is_error:
      self.status_lbl.config(text="Failed!", foreground="red")
      messagebox.showerror("Error", message)
    else:
      self.status_lbl.config(text="Done!", foreground="green")
      messagebox.showinfo("Success", message)

  def start_process_thread(self):
    src = self.source_folder.get().strip()
    if not src or not os.path.isdir(src):
      messagebox.showerror("Error", "Please select a valid source directory.")
      return

    is_overwrite = self.save_mode.get() == "overwrite"
    dst = src
    if not is_overwrite:
      dst = self.dest_folder.get().strip()
      if not dst:
        messagebox.showerror(
            "Error", "Please select a valid destination directory."
        )
        return
      os.makedirs(dst, exist_ok=True)

    min_val = int(self.min_q.get())
    max_val = int(self.max_q.get())
    if min_val > max_val:
      min_val, max_val = max_val, min_val

    oxi_lvl = int(self.oxi_level.get())
    engine = self.tool_mode.get()

    self.btn_run.config(state="disabled")
    thread = threading.Thread(
        target=self.run_compression,
        args=(src, dst, is_overwrite, engine, min_val, max_val, oxi_lvl),
        daemon=True,
    )
    thread.start()

  def run_compression(
      self, src, dst, is_overwrite, engine, min_val, max_val, oxi_lvl
  ):
    try:
      files = sorted(
          [f for f in os.listdir(src) if f.lower().endswith(".png")]
      )
    except Exception as e:
      self.root.after(
          0,
          self.finish_process,
          f"Failed to read folder: {str(e)}",
          True,
      )
      return

    total = len(files)
    if total == 0:
      self.root.after(
          0,
          self.finish_process,
          "No PNG images found in source folder.",
          True,
      )
      return

    self.root.after(0, lambda: self.prog_bar.config(maximum=total, value=0))

    pngquant_bin = get_tool_path("pngquant.exe")
    oxipng_bin = get_tool_path("oxipng.exe")
    flags = subprocess.CREATE_NO_WINDOW if os.name == "nt" else 0

    use_pq = engine in ["Both (Pngquant + OxiPNG)", "Pngquant Only"]
    use_oxi = engine in ["Both (Pngquant + OxiPNG)", "OxiPNG Only"]

    for idx, f in enumerate(files, start=1):
      src_filepath = os.path.join(src, f)
      target_filepath = os.path.join(dst, f)

      if not is_overwrite:
        try:
          shutil.copy2(src_filepath, target_filepath)
        except Exception as e:
          self.root.after(
              0,
              self.finish_process,
              f"Failed copying {f}: {str(e)}",
              True,
          )
          return
      else:
        target_filepath = src_filepath

      self.root.after(0, self.update_ui_progress, idx, total, f)

      if use_pq:
        cmd_pngquant = [
            pngquant_bin,
            f"--quality={min_val}-{max_val}",
            "--ext",
            ".png",
            "--force",
            target_filepath,
        ]
        try:
          subprocess.run(
              cmd_pngquant,
              stdout=subprocess.DEVNULL,
              stderr=subprocess.DEVNULL,
              creationflags=flags,
          )
        except FileNotFoundError:
          self.root.after(
              0,
              self.finish_process,
              "pngquant executable not found.",
              True,
          )
          return

      if use_oxi:
        cmd_oxipng = [
            oxipng_bin,
            "-o",
            str(oxi_lvl),
            "--strip",
            "all",
            target_filepath,
        ]
        try:
          subprocess.run(
              cmd_oxipng,
              stdout=subprocess.DEVNULL,
              stderr=subprocess.DEVNULL,
              creationflags=flags,
          )
        except FileNotFoundError:
          self.root.after(
              0,
              self.finish_process,
              "oxipng executable not found.",
              True,
          )
          return

    self.root.after(
        0,
        self.finish_process,
        f"Completed: {total} PNG files processed.",
        False,
    )


if __name__ == "__main__":
  root = tk.Tk()
  app = PNGCompressorGUI(root)
  root.mainloop()
```

---

## Standalone Executable Packaging

To bundle the application into a single `.exe` with the custom icon:

1. Install PyInstaller:
   ```cmd
   python -m pip install pyinstaller
   ```
2. Put `icon.ico` in the directory alongside `compress_gui.py`.
3. Build the binary:
   ```cmd
   python -m PyInstaller --noconsole --onefile --icon="icon.ico" --add-data "icon.ico;." compress_gui.py
   ```
4. The generated output file will be located at `dist\compress_gui.exe`.

---

## Developer & Attribution

- **Developer:** Shams Sohag
- **GitHub:** [https://github.com/shamssohag](https://github.com/shamssohag)
- **Quantization Engine:** [pngquant](https://pngquant.org/) by Kornel Lesiński
- **Optimizer Engine:** [oxipng](https://github.com/oxipng/oxipng) by Joshua Holmer
- **Icon:** [Gallery icons created by Magnific - Flaticon](https://www.flaticon.com/free-icons/gallery)

## License

This project is distributed under the MIT License.
