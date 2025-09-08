[pcm_v1.py.txt](https://github.com/user-attachments/files/22212420/pcm_v1.py.txt)
# -*- coding: utf-8 -*-
# GUI mínima para generar y reproducir tonos por PCM5102 (o HDMI/jack) usando sounddevice.
# Compatible con Python 3.5 (Raspbian Stretch). No usa 'soundfile'.

from __future__ import print_function
import math
import numpy as np
import sounddevice as sd

try:
    import tkinter as tk
    from tkinter import ttk
except:
    # En algunos sistemas el paquete es 'Tkinter' (pero en Py3 es 'tkinter')
    import Tkinter as tk
    import ttk

APP_TITLE = "PCM5102 Test GUI (sin soundfile)"

def list_output_devices():
    """Devuelve lista [(label, device_selector)] de dispositivos de salida."""
    items = []
    try:
        devs = sd.query_devices()
        for i, d in enumerate(devs):
            if d.get('max_output_channels', 0) >= 2:
                label = "[#{:d}] {} (ch={}, sr≈{})".format(
                    i, d.get('name', 'unknown'), d.get('max_output_channels', 0),
                    int(d.get('default_samplerate', 0) or 0)
                )
                items.append((label, i))
    except Exception as e:
        print("No se pudieron listar dispositivos:", e)
    # Añadimos opción manual para 'hw:X,Y'
    items.append(("Manual: hw:X,Y (ingresá abajo)", "manual"))
    return items

class App(tk.Tk):
    def __init__(self):
        tk.Tk.__init__(self)
        self.title(APP_TITLE)
        self.resizable(False, False)

        # ---- Controles ----
        frm = ttk.Frame(self, padding=10)
        frm.grid(row=0, column=0, sticky="nsew")

        # Dispositivo
        ttk.Label(frm, text="Dispositivo de salida:").grid(row=0, column=0, sticky="w")
        self.device_opts = list_output_devices()
        self.dev_var = tk.StringVar(value=str(self.device_opts[0][1]))
        self.dev_cb = ttk.Combobox(frm, state="readonly", width=45,
                                   values=[x[0] for x in self.device_opts])
        self.dev_cb.current(0)
        self.dev_cb.grid(row=0, column=1, columnspan=3, pady=2, sticky="w")

        ttk.Label(frm, text="Manual (p.ej. hw:0,0):").grid(row=1, column=0, sticky="w")
        self.manual_dev = tk.Entry(frm, width=15)
        self.manual_dev.grid(row=1, column=1, sticky="w")

        # Samplerate y duración
        ttk.Label(frm, text="Samplerate (Hz):").grid(row=2, column=0, sticky="w")
        self.fs_var = tk.StringVar(value="48000")
        ttk.Combobox(frm, textvariable=self.fs_var, width=8,
                     values=["48000", "44100"]).grid(row=2, column=1, sticky="w")

        ttk.Label(frm, text="Duración (s):").grid(row=2, column=2, sticky="e")
        self.dur_var = tk.StringVar(value="2.0")
        ttk.Entry(frm, textvariable=self.dur_var, width=6).grid(row=2, column=3, sticky="w")

        # Tono 1 / 2 / 3
        def tone_row(r, label, fdef, adef, phdef="0"):
            ttk.Label(frm, text=label).grid(row=r, column=0, sticky="w")
            fvar = tk.StringVar(value=fdef); avar = tk.StringVar(value=adef); phvar = tk.StringVar(value=phdef)
            ttk.Label(frm, text="f (Hz)").grid(row=r, column=1, sticky="e"); ttk.Entry(frm, textvariable=fvar, width=8).grid(row=r, column=1, sticky="w", padx=(48,0))
            ttk.Label(frm, text="amp (0-1)").grid(row=r, column=2, sticky="e"); ttk.Entry(frm, textvariable=avar, width=6).grid(row=r, column=3, sticky="w")
            ttk.Label(frm, text="fase (°)").grid(row=r, column=4, sticky="e"); 
            return fvar, avar, phvar

        ttk.Label(frm, text="Tonos (se suman y se normalizan)").grid(row=3, column=0, columnspan=4, pady=(6,0), sticky="w")

        ttk.Label(frm, text="f1 (Hz):").grid(row=4, column=0, sticky="e")
        self.f1 = tk.Entry(frm, width=8); self.f1.insert(0,"440"); self.f1.grid(row=4, column=1, sticky="w")
        ttk.Label(frm, text="a1 (0-1):").grid(row=4, column=2, sticky="e")
        self.a1 = tk.Entry(frm, width=6); self.a1.insert(0,"1.0"); self.a1.grid(row=4, column=3, sticky="w")
        ttk.Label(frm, text="fase1 (°):").grid(row=4, column=4, sticky="e")
        self.p1 = tk.Entry(frm, width=6); self.p1.insert(0,"0"); self.p1.grid(row=4, column=5, sticky="w")

        ttk.Label(frm, text="f2 (Hz):").grid(row=5, column=0, sticky="e")
        self.f2 = tk.Entry(frm, width=8); self.f2.insert(0,"880"); self.f2.grid(row=5, column=1, sticky="w")
        ttk.Label(frm, text="a2 (0-1):").grid(row=5, column=2, sticky="e")
        self.a2 = tk.Entry(frm, width=6); self.a2.insert(0,"0.5"); self.a2.grid(row=5, column=3, sticky="w")
        ttk.Label(frm, text="fase2 (°):").grid(row=5, column=4, sticky="e")
        self.p2 = tk.Entry(frm, width=6); self.p2.insert(0,"45"); self.p2.grid(row=5, column=5, sticky="w")

        ttk.Label(frm, text="f3 (Hz):").grid(row=6, column=0, sticky="e")
        self.f3 = tk.Entry(frm, width=8); self.f3.insert(0,"1760"); self.f3.grid(row=6, column=1, sticky="w")
        ttk.Label(frm, text="a3 (0-1):").grid(row=6, column=2, sticky="e")
        self.a3 = tk.Entry(frm, width=6); self.a3.insert(0,"0.3"); self.a3.grid(row=6, column=3, sticky="w")
        ttk.Label(frm, text="fase3 (°):").grid(row=6, column=4, sticky="e")
        self.p3 = tk.Entry(frm, width=6); self.p3.insert(0,"0"); self.p3.grid(row=6, column=5, sticky="w")

        ttk.Label(frm, text="Volumen (0–1):").grid(row=7, column=0, sticky="e", pady=(6,0))
        self.vol = tk.Entry(frm, width=6); self.vol.insert(0,"1.0"); self.vol.grid(row=7, column=1, sticky="w", pady=(6,0))

        # Botones
        btns = ttk.Frame(frm); btns.grid(row=8, column=0, columnspan=6, pady=10)
        ttk.Button(btns, text="Reproducir", command=self.play).grid(row=0, column=0, padx=5)
        ttk.Button(btns, text="Stop", command=lambda: sd.stop()).grid(row=0, column=1, padx=5)
        ttk.Button(btns, text="Salir", command=self.destroy).grid(row=0, column=2, padx=5)

        # Estado
        self.status = tk.StringVar(value="Listo.")
        ttk.Label(self, textvariable=self.status, padding=(10,0)).grid(row=1, column=0, sticky="w")

    def _get_device(self):
        idx = self.dev_cb.current()
        sel = self.device_opts[idx][1]
        if sel == "manual":
            manual = self.manual_dev.get().strip()
            return manual if manual else None
        return sel

    def play(self):
        try:
            # Selección de dispositivo
            dev = self._get_device()
            if dev not in (None, ""):
                sd.default.device = dev

            fs = int(float(self.fs_var.get()))
            dur = float(self.dur_var.get())
            f1, a1, p1 = float(self.f1.get()), float(self.a1.get()), math.radians(float(self.p1.get()))
            f2, a2, p2 = float(self.f2.get()), float(self.a2.get()), math.radians(float(self.p2.get()))
            f3, a3, p3 = float(self.f3.get()), float(self.a3.get()), math.radians(float(self.p3.get()))
            vol = max(0.0, min(1.0, float(self.vol.get())))

            t = np.linspace(0, dur, int(fs*dur), endpoint=False)
            s = (a1*np.sin(2*np.pi*f1*t + p1) +
                 a2*np.sin(2*np.pi*f2*t + p2) +
                 a3*np.sin(2*np.pi*f3*t + p3))

            # Normalizar y aplicar volumen
            peak = np.max(np.abs(s)) or 1.0
            s = (s / peak) * vol

            # Estéreo float32
            stereo = np.column_stack([s, s]).astype(np.float32)

            sd.default.samplerate = fs
            sd.default.channels = 2
            sd.play(stereo)  # no bloquea
            self.status.set("Reproduciendo… (dispositivo: {})".format(sd.default.device))
        except Exception as e:
            self.status.set("Error: {}".format(e))
            print("ERROR:", e)

if __name__ == "__main__":
    App().mainloop()
