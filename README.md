# FT_code
#푸리에 변환을 이용한 노이즈 필터링 코드입니다. 노이즈 제거할 파일(wav)명을 입력하고 제거할 진동수를 입력하면 해당 진동수의 노이즈가 제거됩니다.

import numpy as np
import matplotlib.pyplot as plt
from scipy.fftpack import fft, ifft
import soundfile as sf
import os
import tkinter as tk
from tkinter import filedialog, messagebox
import ctypes

def set_dpi_awareness():
    try:
        ctypes.windll.shcore.SetProcessDpiAwareness(2)  # Enable DPI awareness on Windows
    except Exception as e:
        print(f"DPI awareness could not be set: {e}")

def read_audio_file(file_name):
    if not os.path.exists(file_name):
        raise FileNotFoundError(f"File '{file_name}' not found.")
    try:
        data, samplerate = sf.read(file_name)
    except Exception as e:
        raise RuntimeError(f"Failed to read audio file: {e}")
    return data, samplerate

def compute_fft(signal, samplerate):
    n = len(signal)
    T = 1.0 / samplerate
    yf = fft(signal)
    xf = np.fft.fftfreq(n, d=T)
    return xf, yf

def remove_frequency(signal_fft, samplerate, target_freq, bandwidth=10):
    n = len(signal_fft)
    freqs = np.fft.fftfreq(n, d=1/samplerate)
    target_indices = np.where((np.abs(freqs - target_freq) < bandwidth) | (np.abs(freqs + target_freq) < bandwidth))
    signal_fft[target_indices] = 0
    return signal_fft

def compute_ifft(signal_fft):
    return np.real(ifft(signal_fft))

def save_audio_file(file_name, data, samplerate):
    try:
        sf.write(file_name, data, samplerate)
    except Exception as e:
        raise RuntimeError(f"Failed to save audio file: {e}")

def process_audio(input_file, target_freq, output_file):
    try:
        audio_data, samplerate = read_audio_file(input_file)
        if audio_data.ndim > 1:
            audio_data = audio_data[:, 0]  # Convert to mono if stereo

        xf, yf = compute_fft(audio_data, samplerate)
        original_yf = yf.copy()  # Preserve the original FFT for plotting
        yf_filtered = remove_frequency(yf, samplerate, target_freq)
        filtered_audio = compute_ifft(yf_filtered)
        save_audio_file(output_file, filtered_audio, samplerate)

        # Plot original and filtered frequency domain
        plt.figure(figsize=(10, 6))
        plt.plot(xf[:len(xf)//2], np.abs(original_yf[:len(original_yf)//2]), label="Original")
        plt.plot(xf[:len(xf)//2], np.abs(yf_filtered[:len(yf_filtered)//2]), label="Filtered")
        plt.title("Frequency Domain")
        plt.xlabel("Frequency (Hz)")
        plt.ylabel("Amplitude")
        plt.legend()
        plt.grid()
        plt.show()

        return True
    except Exception as e:
        messagebox.showerror("Error", str(e))
        return False

def browse_file(entry):
    file_path = filedialog.askopenfilename(filetypes=[("WAV files", "*.wav")])
    entry.delete(0, tk.END)
    entry.insert(0, file_path)

def save_as(entry):
    file_path = filedialog.asksaveasfilename(defaultextension=".wav", filetypes=[("WAV files", "*.wav")])
    entry.delete(0, tk.END)
    entry.insert(0, file_path)

def run_process(input_entry, freq_entry, output_entry):
    input_file = input_entry.get()
    target_freq = freq_entry.get()
    output_file = output_entry.get()

    if not input_file or not target_freq or not output_file:
        messagebox.showwarning("Input Error", "All fields must be filled.")
        return

    try:
        target_freq = float(target_freq)
        success = process_audio(input_file, target_freq, output_file)
        if success:
            messagebox.showinfo("Success", f"Filtered audio saved as: {output_file}")
    except ValueError:
        messagebox.showerror("Input Error", "Frequency must be a numeric value.")

def main_gui():
    set_dpi_awareness()  # Enable DPI awareness
    root = tk.Tk()
    root.title("Audio Noise Filter")
    root.resizable(False, False)  # Disable window resizing

    tk.Label(root, text="Input File:").grid(row=0, column=0, padx=5, pady=5, sticky="e")
    input_entry = tk.Entry(root, width=50)
    input_entry.grid(row=0, column=1, padx=5, pady=5)
    tk.Button(root, text="Browse", command=lambda: browse_file(input_entry)).grid(row=0, column=2, padx=5, pady=5)

    tk.Label(root, text="Target Frequency (Hz):").grid(row=1, column=0, padx=5, pady=5, sticky="e")
    freq_entry = tk.Entry(root, width=20)
    freq_entry.grid(row=1, column=1, padx=5, pady=5, sticky="w")

    tk.Label(root, text="Output File:").grid(row=2, column=0, padx=5, pady=5, sticky="e")
    output_entry = tk.Entry(root, width=50)
    output_entry.grid(row=2, column=1, padx=5, pady=5)
    tk.Button(root, text="Save As", command=lambda: save_as(output_entry)).grid(row=2, column=2, padx=5, pady=5)

    tk.Button(root, text="Process", command=lambda: run_process(input_entry, freq_entry, output_entry)).grid(row=3, column=0, columnspan=3, pady=10)

    root.mainloop()

if __name__ == "__main__":
    main_gui()
