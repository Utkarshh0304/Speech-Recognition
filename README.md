import tkinter as tk
from tkinter import ttk, messagebox
from googletrans import Translator, LANGUAGES
import speech_recognition as sr
import threading
import pyttsx3
import webbrowser
from functools import partial

# --- THEME AND STYLE CONSTANTS ---
class DarkTheme:
    BACKGROUND = "#121212"
    SURFACE = "#1E1E1E"
    PRIMARY = "#BB86FC"
    PRIMARY_LIGHT = "#D0A9F9"
    SECONDARY = "#03DAC6"
    SECONDARY_LIGHT = "#36F4E2"
    TEXT = "#FFFFFF"
    TEXT_SECONDARY = "#B3B3B3"
    ERROR = "#CF6679"

class LightTheme:
    BACKGROUND = "#F5F5F5"
    SURFACE = "#FFFFFF"
    PRIMARY = "#6200EE"
    PRIMARY_LIGHT = "#7F39FB"
    SECONDARY = "#018786" # Darker teal for better contrast on white
    SECONDARY_LIGHT = "#03DAC6"
    TEXT = "#000000"
    TEXT_SECONDARY = "#666666"
    ERROR ="#B00020"

class Fonts:
    TITLE = ("Segoe UI", 24, "bold")
    BODY_BOLD = ("Segoe UI", 12, "bold")
    BODY = ("Segoe UI", 11)
    ITALIC = ("Segoe UI", 10, "italic")

# --- CUSTOM WIDGETS ---
class HoverButton(tk.Label):
    """A custom button-like label that changes color on hover."""
    def __init__(self, master, text, command, **kwargs):
        super().__init__(master, text=text, **kwargs)
        self.command = command
        self.default_bg = kwargs.get("bg")
        self.hover_bg = kwargs.get("activebackground")
        
        self.bind("<Enter>", self.on_enter)
        self.bind("<Leave>", self.on_leave)
        self.bind("<Button-1>", self.on_click)
        self.configure(cursor="hand2")

    def set_colors(self, default_bg, hover_bg, text_fg):
        """Allows direct setting of button colors for flexible theming."""
        self.default_bg = default_bg
        self.hover_bg = hover_bg
        self.config(bg=self.default_bg, fg=text_fg)

    def on_enter(self, event):
        self.config(bg=self.hover_bg)

    def on_leave(self, event):
        self.config(bg=self.default_bg)

    def on_click(self, event):
        self.command()

# --- MAIN APPLICATION ---
class VoiceGlobeApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.translator = Translator()
        self.theme = DarkTheme  # Start with Dark Theme
        self.placeholder_text = "Type or use voice input to translate..."
        self.name_to_code = {v: k for k, v in LANGUAGES.items()}
        self.buttons = [] # To store buttons for theme updates

        # --- ATTRACTION DATABASE ---
        self.ATTRACTIONS_DB = {
            'spanish': [
                {'name': 'Alhambra (Granada, Spain)', 'wiki': 'https://en.wikipedia.org/wiki/Alhambra'},
                {'name': 'Sagrada Família (Barcelona, Spain)', 'wiki': 'https://en.wikipedia.org/wiki/Sagrada_Fam%C3%ADlia'},
                {'name': 'Caminito del Rey (Hidden Spot, Spain)', 'wiki': 'https://en.wikipedia.org/wiki/Caminito_del_Rey'},
            ],
            'french': [
                {'name': 'Eiffel Tower (Paris, France)', 'wiki': 'https://en.wikipedia.org/wiki/Eiffel_Tower'},
                {'name': 'Louvre Museum (Paris, France)', 'wiki': 'https://en.wikipedia.org/wiki/Louvre'},
                {'name': 'Palais des Papes (Avignon, France)', 'wiki': 'https://en.wikipedia.org/wiki/Palais_des_Papes'},
                {'name': 'Canal du Midi (Hidden Spot, France)', 'wiki': 'https://en.wikipedia.org/wiki/Canal_du_Midi'},
            ],
            'japanese': [
                {'name': 'Kinkaku-ji (Kyoto, Japan)', 'wiki': 'https://en.wikipedia.org/wiki/Kinkaku-ji'},
                {'name': 'Fushimi Inari-taisha (Kyoto, Japan)', 'wiki': 'https://en.wikipedia.org/wiki/Fushimi_Inari-taisha'},
                {'name': 'Omoide Yokocho (Hidden Spot, Tokyo)', 'wiki': 'https://en.wikipedia.org/wiki/Omoide_Yokocho'},
            ],
            'italian': [
                {'name': 'Colosseum (Rome, Italy)', 'wiki': 'https://en.wikipedia.org/wiki/Colosseum'},
                {'name': 'Uffizi Gallery (Florence, Italy)', 'wiki': 'https://en.wikipedia.org/wiki/Uffizi'},
                {'name': 'Sassi di Matera (Hidden Spot, Italy)', 'wiki': 'https://en.wikipedia.org/wiki/Sassi_di_Matera'},
            ],
             'german': [
                {'name': 'Neuschwanstein Castle (Germany)', 'wiki': 'https://en.wikipedia.org/wiki/Neuschwanstein_Castle'},
                {'name': 'Brandenburg Gate (Berlin, Germany)', 'wiki': 'https://en.wikipedia.org/wiki/Brandenburg_Gate'},
                {'name': 'Spreewald (Hidden Spot, Germany)', 'wiki': 'https://en.wikipedia.org/wiki/Spreewald'},
            ],
            'hindi': [
                {'name': 'Taj Mahal (Agra, India)', 'wiki': 'https://en.wikipedia.org/wiki/Taj_Mahal'},
                {'name': 'Ghats in Varanasi (India)', 'wiki': 'https://en.wikipedia.org/wiki/Ghats_in_Varanasi'},
                {'name': 'Chand Baori (Hidden Spot, Rajasthan)', 'wiki': 'https://en.wikipedia.org/wiki/Chand_Baori'},
            ],
            'english': [
                {'name': 'Statue of Liberty (New York, USA)', 'wiki': 'https://en.wikipedia.org/wiki/Statue_of_Liberty'},
                {'name': 'Grand Canyon (Arizona, USA)', 'wiki': 'https://en.wikipedia.org/wiki/Grand_Canyon'},
                {'name': 'The Cotswolds (Scenic Area, UK)', 'wiki': 'https://en.wikipedia.org/wiki/Cotswolds'},
                {'name': 'Salvation Mountain (Hidden Spot, USA)', 'wiki': 'https://en.wikipedia.org/wiki/Salvation_Mountain'},
            ],
            'chinese': [
                {'name': 'Great Wall of China (China)', 'wiki': 'https://en.wikipedia.org/wiki/Great_Wall_of_China'},
                {'name': 'Terracotta Army (Xi\'an, China)', 'wiki': 'https://en.wikipedia.org/wiki/Terracotta_Army'},
                {'name': 'Shaxi Ancient Town (Hidden Spot, Yunnan)', 'wiki': 'https://en.wikipedia.org/wiki/Shaxi,_Yunnan'},
            ],
            'portuguese': [
                {'name': 'Christ the Redeemer (Rio, Brazil)', 'wiki': 'https://en.wikipedia.org/wiki/Christ_the_Redeemer_(statue)'},
                {'name': 'Pena Palace (Sintra, Portugal)', 'wiki': 'https://en.wikipedia.org/wiki/Pena_Palace'},
                {'name': 'Benagil Cave (Hidden Spot, Portugal)', 'wiki': 'https://en.wikipedia.org/wiki/Benagil'},
                {'name': 'Lençóis Maranhenses (Hidden Spot, Brazil)', 'wiki': 'https://en.wikipedia.org/wiki/Len%C3%A7%C3%B3is_Maranhenses_National_Park'},
            ]
        }
        # --- END DATABASE ---

        self.configure_window()
        self.create_widgets()
        self.update_theme() # Apply initial theme

    def configure_window(self):
        self.title("VOICE GLOBE")
        self.geometry("900x750")
        self.minsize(850, 700)

    def create_widgets(self):
        # Header - using grid to place toggle button easily
        self.header = tk.Frame(self)
        self.header.pack(fill=tk.X)
        self.header.grid_columnconfigure(0, weight=1)
        self.header.grid_columnconfigure(2, weight=1)
        
        self.header_title = tk.Label(self.header, text="VOICE GLOBE 🌐", font=Fonts.TITLE)
        self.header_title.grid(row=0, column=1, pady=20)
        
        self.theme_toggle_button = HoverButton(self.header, text="", command=self.toggle_theme,
                                               font=("Segoe UI", 16), relief=tk.FLAT, padx=10, pady=5)
        self.theme_toggle_button.grid(row=0, column=2, sticky='e', padx=20)

        # Main content frame
        self.content_frame = tk.Frame(self, padx=30, pady=20)
        self.content_frame.pack(fill=tk.BOTH, expand=True)

        # Language Selection
        self.lang_frame = tk.Frame(self.content_frame)
        self.lang_frame.pack(fill=tk.X, pady=(0, 20))
        self.lang_label = tk.Label(self.lang_frame, text="Translate To:", font=Fonts.BODY_BOLD)
        self.lang_label.pack(side=tk.LEFT, padx=(0, 10))
        
        languages = list(LANGUAGES.values())
        self.language_to = ttk.Combobox(self.lang_frame, values=languages, font=Fonts.BODY, width=25)
        self.language_to.set("english")
        self.language_to.pack(side=tk.LEFT)

        # Input Area
        self.input_label = tk.Label(self.content_frame, text="Input", font=Fonts.BODY_BOLD)
        self.input_label.pack(anchor=tk.W)
        
        # Removed 'bd=5' from here, 'update_theme' will handle styling
        self.text_input = tk.Text(self.content_frame, height=6, wrap=tk.WORD, font=Fonts.BODY, relief=tk.FLAT) 
        self.text_input.pack(fill=tk.X, pady=5)
        self.add_placeholder()
        self.text_input.bind("<FocusIn>", self.clear_placeholder)
        self.text_input.bind("<FocusOut>", self.add_placeholder)
        
        self.voice_label = tk.Label(self.content_frame, text="", font=Fonts.ITALIC)
        self.voice_label.pack(pady=5)
        
        # Buttons
        self.button_frame = tk.Frame(self.content_frame)
        self.button_frame.pack(pady=20)
        
        # --- Using the "CLEAR" button version ---
        buttons_info = [
            ("TRANSLATE", self.translate_text, 'primary'),
            (" VOICE INPUT", self.voice_input, 'secondary'),
            (" SPEAK", self.threaded_speak_output, 'secondary'),
            (" CLEAR", self.clear_text, 'secondary'),
            (" EXPLORE ✈️", self.show_recommendations, 'primary'),
        ]

        self.buttons = [] # Clear the list first
        for i, (text, cmd, style_key) in enumerate(buttons_info):
            btn = HoverButton(self.button_frame, text=text, command=cmd, font=Fonts.BODY_BOLD, relief=tk.FLAT, padx=20, pady=10)
            
            # --- FIX: Using self.theme to access theme data ---
            btn.set_colors(
                default_bg=self.theme.PRIMARY if style_key == 'primary' else self.theme.SECONDARY,
                hover_bg=self.theme.PRIMARY_LIGHT if style_key == 'primary' else self.theme.SECONDARY_LIGHT,
                text_fg=DarkTheme.TEXT if self.theme == DarkTheme else LightTheme.TEXT
            )
            # --- END FIX ---
            
            btn.grid(row=0, column=i, padx=10)
            self.buttons.append((btn, style_key))

        # Detected Language Label
        self.detected_label = tk.Label(self.content_frame, text="", font=Fonts.ITALIC)
        self.detected_label.pack(pady=5)
        
        # Output Area
        self.output_label = tk.Label(self.content_frame, text="Output", font=Fonts.BODY_BOLD)
        self.output_label.pack(anchor=tk.W)
        
        # Removed 'bd=5' from here, 'update_theme' will handle styling
        self.text_output = tk.Text(self.content_frame, height=6, wrap=tk.WORD, font=Fonts.BODY, relief=tk.FLAT)
        self.text_output.pack(fill=tk.X, pady=5)
        
        # Voice Settings
        self.settings_outer_frame = tk.Frame(self.content_frame)
        self.settings_outer_frame.pack(pady=20)
        self.settings_title = tk.Label(self.settings_outer_frame, text="Voice Settings", font=Fonts.BODY_BOLD)
        self.settings_title.pack()
        self.settings_frame = tk.Frame(self.settings_outer_frame, padx=15, pady=15)
        self.settings_frame.pack(pady=10)
        
        self.voice_var = tk.StringVar(value="Male")
        self.radio_male = tk.Radiobutton(self.settings_frame, text="Male", variable=self.voice_var, value="Male", font=Fonts.BODY)
        self.radio_male.grid(row=0, column=0, padx=10)
        self.radio_female = tk.Radiobutton(self.settings_frame, text="Female", variable=self.voice_var, value="Female", font=Fonts.BODY)
        self.radio_female.grid(row=0, column=1, padx=10)

        self.speed_label = tk.Label(self.settings_frame, text="Speed:", font=Fonts.BODY)
        self.speed_label.grid(row=0, column=2, padx=(20, 5))
        self.speed_scale = tk.Scale(self.settings_frame, from_=100, to=250, orient=tk.HORIZONTAL, length=120, highlightthickness=0)
        self.speed_scale.set(175)
        self.speed_scale.grid(row=0, column=3)

        self.volume_label = tk.Label(self.settings_frame, text="Volume:", font=Fonts.BODY)
        self.volume_label.grid(row=0, column=4, padx=(20, 5))
        self.volume_scale = tk.Scale(self.settings_frame, from_=0, to=100, orient=tk.HORIZONTAL, length=120, highlightthickness=0)
        self.volume_scale.set(100)
        self.volume_scale.grid(row=0, column=5)
    
    # --- THEME LOGIC (MODERNIZED) ---
    def toggle_theme(self):
        self.theme = LightTheme if self.theme == DarkTheme else DarkTheme
        self.update_theme()

    def update_theme(self):
        theme = self.theme
        
        # 1. MAIN WINDOW
        self.configure(bg=theme.BACKGROUND)
        
        # 2. HEADER
        self.header.config(bg=theme.SURFACE) # Header is a "card"
        self.header_title.config(bg=theme.SURFACE, fg=theme.SECONDARY)
        
        toggle_text = "☀️" if theme == DarkTheme else "🌙"
        self.theme_toggle_button.config(text=toggle_text)
        self.theme_toggle_button.set_colors(
            default_bg=theme.SURFACE,
            hover_bg=theme.BACKGROUND,
            text_fg=theme.SECONDARY
        )

        # 3. CONTENT FRAME (This is the "main background")
        for widget in [self.content_frame, self.lang_frame, self.button_frame, self.settings_outer_frame]:
            widget.config(bg=theme.BACKGROUND)

        # 4. LABELS (on the main background)
        for widget in [self.lang_label, self.input_label, self.output_label, self.settings_title]:
            widget.config(bg=theme.BACKGROUND, fg=theme.TEXT)
        self.voice_label.config(bg=theme.BACKGROUND)
        self.detected_label.config(bg=theme.BACKGROUND, fg=theme.PRIMARY_LIGHT)
        
        # 5. TEXT WIDGETS (These are "cards")
        # Add padding INSIDE the text box and a highlight border
        for text_widget in [self.text_input, self.text_output]:
            text_widget.config(
                bg=theme.SURFACE,                   # "Card" color
                fg=theme.TEXT,                      
                insertbackground=theme.TEXT,        # Cursor color
                relief=tk.FLAT,                     # No 3D border
                bd=0,                               # No standard border
                highlightthickness=1,               # Use the highlight ring as the border
                highlightbackground=theme.PRIMARY,  # Border color
                padx=10,                            # 10px padding on left/right
                pady=10                             # 10px padding on top/bottom
            )
        
        # Re-apply placeholder color
        if self.text_input.get("1.0", "end-1c") == self.placeholder_text:
            self.text_input.config(fg=theme.TEXT_SECONDARY)
        else:
            self.text_input.config(fg=theme.TEXT)

        # 6. ACTION BUTTONS
        for btn, style_key in self.buttons:
            btn.set_colors(
                default_bg=theme.PRIMARY if style_key == 'primary' else theme.SECONDARY,
                hover_bg=theme.PRIMARY_LIGHT if style_key == 'primary' else theme.SECONDARY_LIGHT,
                text_fg=DarkTheme.TEXT if theme == DarkTheme else LightTheme.TEXT
            )
            
        # 7. SETTINGS PANEL (This is a "card")
        self.settings_frame.config(
            bg=theme.SURFACE,    # "Card" color
            bd=1,                # 1px border
            relief=tk.SOLID,     # Solid border style
            highlightbackground=theme.PRIMARY,
            highlightthickness=0
        )
        
        # Contents of the settings card
        for widget in [self.radio_male, self.radio_female]:
            widget.config(bg=theme.SURFACE, fg=theme.TEXT, selectcolor=theme.BACKGROUND, activebackground=theme.SURFACE, activeforeground=theme.TEXT)
        for widget in [self.speed_label, self.volume_label]:
            widget.config(bg=theme.SURFACE, fg=theme.TEXT)
        for scale in [self.speed_scale, self.volume_scale]:
            scale.config(bg=theme.SURFACE, fg=theme.SECONDARY, troughcolor=theme.BACKGROUND)

        # 8. TTK COMBOBOX STYLE (Keep this)
        style = ttk.Style(self)
        style.configure("TCombobox", 
                        fieldbackground=theme.SURFACE, 
                        foreground=theme.TEXT, 
                        arrowcolor=theme.TEXT,
                        bordercolor=theme.PRIMARY,
                        borderwidth=1,
                        lightcolor=theme.SURFACE,
                        darkcolor=theme.SURFACE)
        
        self.option_add('*TCombobox*Listbox.background', theme.SURFACE)
        self.option_add('*TCombobox*Listbox.foreground', theme.TEXT) 
        self.option_add('*TCombobox*Listbox.selectBackground', theme.PRIMARY)
        self.option_add('*TCombobox*Listbox.selectForeground', theme.TEXT)

    # --- PLACEHOLDER LOGIC ---
    def add_placeholder(self, event=None):
        # --- FIX: Changed "1.to" to "1.0" ---
        if not self.text_input.get("1.0", "end-1c"):
            self.text_input.insert("1.0", self.placeholder_text)
            self.text_input.config(fg=self.theme.TEXT_SECONDARY)

    def clear_placeholder(self, event=None):
        if self.text_input.get("1.0", "end-1c") == self.placeholder_text:
            self.text_input.delete("1.0", tk.END)
            self.text_input.config(fg=self.theme.TEXT)

    # --- CORE FUNCTIONS ---
    
    # --- MODIFIED: This now starts the thread ---
    def translate_text(self):
        """Starts the translation in a new thread to prevent UI freezing."""
        threading.Thread(target=self._translate_thread, daemon=True).start()

    # --- MODIFIED: This is the original function, renamed to run in a thread ---
    def _translate_thread(self):
        """The actual translation logic that runs in a thread."""
        try:
            text = self.text_input.get("1.0", tk.END).strip()
            if not text or text == self.placeholder_text:
                messagebox.showwarning(" Warning", "Please enter text to translate.")
                return
            dest_lang = self.language_to.get()
            dest_code = self.name_to_code.get(dest_lang)
            
            # This is the blocking call that can time out
            translated = self.translator.translate(text, dest=dest_code)
            
            # Update UI
            self.text_output.delete("1.0", tk.END)
            self.text_output.insert(tk.END, translated.text)
            src_lang_name = LANGUAGES.get(translated.src, translated.src).capitalize()
            self.detected_label.config(text=f"Detected: {src_lang_name} → {dest_lang.capitalize()}")
        except Exception as e:
            # This will now catch timeouts without freezing the app
            messagebox.showerror("Error", f"Translation failed (network error or timeout):\n{e}")

    # --- ADDED: clear_text function ---
    def clear_text(self):
        """Clears all text fields and resets labels."""
        self.text_input.delete("1.0", tk.END)
        self.text_output.delete("1.0", tk.END)
        self.detected_label.config(text="")
        self.voice_label.config(text="")
        # Re-apply the placeholder to the now-empty input box
        self.add_placeholder()

    def threaded_speak_output(self):
        threading.Thread(target=self.speak_output, daemon=True).start()

    def speak_output(self):
        text = self.text_output.get("1.0", tk.END).strip()
        if not text:
            messagebox.showwarning(" Warning", "No translated text to speak.")
            return
        try:
            engine = pyttsx3.init()
            voices = engine.getProperty('voices')
            engine.setProperty('voice', voices[0].id if self.voice_var.get() == "Male" else voices[min(1, len(voices)-1)].id)
            engine.setProperty('rate', self.speed_scale.get())
            engine.setProperty('volume', self.volume_scale.get() / 100.0)
            engine.say(text)
            engine.runAndWait()
            engine.stop()
        except Exception as e:
            messagebox.showerror("Error", f"Voice playback failed:\n{e}")

    # --- DELETED: copy_text function ---

    def voice_input(self):
        threading.Thread(target=self._voice_input_thread, daemon=True).start()

    def _voice_input_thread(self):
        recognizer = sr.Recognizer()
        with sr.Microphone() as source:
            try:
                self.voice_label.config(text=" Listening...", fg=self.theme.SECONDARY)
                recognizer.adjust_for_ambient_noise(source, duration=0.5)
                audio = recognizer.listen(source, timeout=6)
                self.voice_label.config(text=" Processing...", fg=self.theme.PRIMARY_LIGHT)
                text = recognizer.recognize_google(audio)
                self.text_input.delete("1.0", tk.END)
                self.text_input.insert(tk.END, text)
                self.text_input.config(fg=self.theme.TEXT)
                self.voice_label.config(text=" Voice input received!", fg=self.theme.SECONDARY)
            except sr.UnknownValueError:
                self.voice_label.config(text=" Could not understand, try again.", fg=self.theme.ERROR)
            except sr.WaitTimeoutError:
                self.voice_label.config(text=" Listening timed out.", fg=self.theme.ERROR)
            except Exception as e:
                self.voice_label.config(text=f"Mic Error: {e}", fg=self.theme.ERROR)

    # --- RECOMMENDATION FUNCTIONS ---

    def show_recommendations(self):
        """Creates a new themed Toplevel window to show recommendations."""
        target_lang = self.language_to.get().lower()
        
        if 'chinese' in target_lang:
            target_lang = 'chinese'
            
        if target_lang not in self.ATTRACTIONS_DB:
            messagebox.showinfo("No Recommendations", f"Sorry, we don't have specific recommendations for {self.language_to.get().capitalize()} yet.")
            return
        
        recs = self.ATTRACTIONS_DB[target_lang]
        
        # Create a new Toplevel window
        rec_window = tk.Toplevel(self)
        rec_window.title(f"Explore {self.language_to.get().capitalize()}")
        rec_window.geometry("550x400")
        rec_window.configure(bg=self.theme.BACKGROUND)
        rec_window.transient(self)
        rec_window.grab_set()

        title_label = tk.Label(rec_window, text=f"Attractions for {self.language_to.get().capitalize()}", 
                               font=Fonts.BODY_BOLD, bg=self.theme.BACKGROUND, fg=self.theme.PRIMARY)
        title_label.pack(pady=(15, 10))

        # This frame is now a "card"
        main_frame = tk.Frame(rec_window, bg=self.theme.SURFACE, bd=1, relief=tk.SOLID)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=10)
        main_frame.grid_columnconfigure(0, weight=1)
        main_frame.grid_columnconfigure(1, weight=0)

        for i, place in enumerate(recs):
            name_label = tk.Label(main_frame, text=place['name'], 
                                   font=Fonts.BODY, bg=self.theme.SURFACE, 
                                   fg=self.theme.TEXT, anchor='w')
            name_label.grid(row=i, column=0, sticky='w', padx=10, pady=5)
            
            link_label = tk.Label(main_frame, text="[Wikipedia Link]", 
                                  font=(Fonts.BODY[0], Fonts.BODY[1], "underline"), 
                                  bg=self.theme.SURFACE, fg=self.theme.SECONDARY_LIGHT, 
                                  cursor="hand2", anchor='e')
            link_label.grid(row=i, column=1, sticky='e', padx=10, pady=5)
            
            link_label.bind("<Button-1>", partial(self.open_link, place['wiki']))
        
        close_btn = HoverButton(rec_window, text="CLOSE", command=rec_window.destroy, 
                                font=Fonts.BODY_BOLD, relief=tk.FLAT, padx=20, pady=10)
        
        btn_text_fg = DarkTheme.TEXT if self.theme == DarkTheme else LightTheme.TEXT
        close_btn.set_colors(self.theme.PRIMARY, self.theme.PRIMARY_LIGHT, btn_text_fg)
        close_btn.pack(pady=10)

    def open_link(self, url, event=None):
        """Opens the given URL in a new browser tab."""
        try:
            webbrowser.open_new(url)
        except Exception as e:
            messagebox.showerror("Error", f"Could not open link:\n{e}", parent=self)
    
if __name__ == "__main__":
    app = VoiceGlobeApp()
    app.mainloop()
