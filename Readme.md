from __future__ import annotations  #allows forward references in type hints

import datetime     # for time/date operations
import logging      # for logging messages
import time         #for sleep/delays
import webbrowser   # to open URLs in brower
from abc import ABC, abstractmethod     #for creating abstract base classes
import pyttsx3      # text to speech lib
import speech_recognition as sr  # speech to text library
import smtplib
from secrets import senderEmail, epwd, to  
from email.message import EmailMessage

# -----------------------------------------------
#   CONFIG
# ----------------------------------------------
class Config:
    voice_index:    int = 1          # which system voice to use
    listen_timeout: int = 5          # max sec to wait for speech
    phrase_limit:   int = 8          # max sec per phrase
    language:       str = "en-IN"    # indian english

# --------------------------------------------------
#   LOGGING
# --------------------------------------------------
logging.basicConfig(
    level=logging.INFO,     # show INFO level and above
    format="%(asctime)s  [%(levelname)s]  %(message)s",     # time+ level+ msg
    datefmt="%H:%M:%S",     # show only hours:minutes:seconds
)
log = logging.getLogger("Nexa")   # create logger named "Nexa"

# -----------------------------------------------
#   VOICE input
# -----------------------------------------------
class Speaker:
    def __init__(self, voice_index: int = 1) -> None:
        self._voice_index = voice_index     # store which voice to use

    def _make_engine(self) -> pyttsx3.Engine:
        engine = pyttsx3.init()     #initialize TTS engine
        voices = engine.getProperty("voices")   # get available system voices
        if voices:
            idx = min(self._voice_index, len(voices) - 1)   # prevent index out of range
            engine.setProperty("voice", voices[idx].id)     #set the voice
        return engine

    def say(self, text: str) -> None:
        log.info("Nexa %s", text) # log what will be spoken
        engine = self._make_engine()   # fresh engine every time
        engine.say(text)    # Queue the text
        engine.runAndWait() # speak it (blocking)
        engine.stop()       # clean shutdown

# ----------------------------------------
#   VOICE output
# -----------------------------------------
class Listener:
    def __init__(self, cfg: Config) -> None:
        self._cfg = cfg     # store config
        self._recognizer = sr.Recognizer()     # create speech recognizer

    def listen(self) -> str:
        r = self._recognizer
        try:
            with sr.Microphone() as source: # open microphone
                log.info("Listening…")
                r.adjust_for_ambient_noise(source, duration=0.8)    #calibrate for noise
                r.pause_threshold = 1.0     # 1 sec silence = end
                audio = r.listen(
                    source,
                    timeout=self._cfg.listen_timeout,   # wait 5sec for speech start
                    phrase_time_limit=self._cfg.phrase_limit,   # max 8 sec recording
                )

            log.info("Recognising…")
            text = r.recognize_google(audio, language=self._cfg.language)   # Google API
            log.info("You said: %s", text)
            return text.lower() # return lowercase text

        except sr.WaitTimeoutError:     # no speech detection
            log.debug("No speech detected (timeout).")
        except sr.UnknownValueError:    # couldn't understand audio
            log.debug("Audio not understood.")
        except sr.RequestError as exc:  # API/network error
            log.error("Speech service error: %s", exc)
        return "" # return empty string on any error

# -----------------------------------------------
#   SKILL BASE CLASS
# ------------------------------------------------
class Skill(ABC):
    @property
    @abstractmethod
    def triggers(self) -> list[str]: ...    # must define trigger keywords

    @abstractmethod
    def run(self, query: str, speaker: Speaker) -> None: ...    # must define action

    def matches(self, query: str) -> bool:
        return any(t in query for t in self.triggers)   # check if any trigger in query

# ------------------------------------------------
#   SKILLS
# ------------------------------------------------
class TimeSkill(Skill):
    triggers = ["time"] # activate on "time" keyword

    def run(self, query: str, speaker: Speaker) -> None:
        now = datetime.datetime.now().strftime("%I:%M %p")
        speaker.say(f"The time is {now}")


class DateSkill(Skill):
    triggers = ["date"] # activate on "date" keyword

    def run(self, query: str, speaker: Speaker) -> None:
        now = datetime.datetime.now()
        speaker.say(f"Today is {now.day} {now.strftime('%B')} {now.year}")


class WebSkill(Skill):
    SITES: dict[str, str] = {
        "youtube": "https://youtube.com",
        "google":  "https://google.com",
        "github":  "https://github.com",
        "maps":    "https://maps.google.com",
        "youtube": "https://youtube.com",
        "google":  "https://google.com",
        "github":  "https://github.com",
        "maps":    "https://maps.google.com",
        "netflix": "https://netflix.com",
        "spotify": "https://spotify.com",
        "gitlab": "https://gitlab.com",
        "replit": "https://replit.com",
        "uber": "https://uber.com",
        "ola": "https://ola.com",
        "instagram": "https://instagram.com",
        "linkedin": "https://linkedin.com",
        "twitter": "https://twitter.com",
        "amazon": "https://amazon.in",
        "flipkart": "https://flipkart.com",
        "zomato": "https://zomato.com",
        "swiggy": "https://swiggy.com",
        "gmail": "https://mail.google.com",
        "drive": "https://drive.google.com",
        "calendar": "https://calendar.google.com",
        "notion": "https://notion.so",
        "snapchat": "https://snapchat.com",
        "tiktok": "https://tiktok.com",
        "telegram": "https://web.telegram.org",
        "whatsapp": "https://web.whatsapp.com",
        "threads": "https://threads.net",
    }
    triggers = [f"open {k}" for k in SITES]

    def run(self, query: str, speaker: Speaker) -> None:
        for keyword, url in self.SITES.items():
            if keyword in query:    # check which site mentioned
                webbrowser.open(url)    # open in defualt browser
                speaker.say(f"Opening {keyword}")
                return


# ---------------------------------------------
#   COMMAND BUS
# ---------------------------------------------
class CommandBus:
    def __init__(self, skills: list[Skill], speaker: Speaker) -> None:
        self._skills  = skills  # list of all available skills
        self._speaker = speaker

    def dispatch(self, query: str) -> bool:
        for skill in self._skills:      # check each skills
            if skill.matches(query):    # does query contain skill's trigger?
                skill.run(query, self._speaker)     # execute skill
                return True     # command handled
        return False        # no skill matched

# ------------------------------------------------
#   GREETING
# --------------------------------------------------
def _greeting() -> str:
    hour = datetime.datetime.now().hour
    if   6  <= hour < 12: salutation = "Good morning"
    elif 12 <= hour < 18: salutation = "Good afternoon"
    elif 18 <= hour < 24: salutation = "Good evening"
    else:                 salutation = "Good night"
    return f"{salutation}. Nexa at your service. How may I help you?"

# -------------------------------------------------
#  Nexa
# ------------------------------------------------
class Nexa:
    def __init__(
        self,
        speaker:    Speaker,
        listener:   Listener,
        bus:        CommandBus,
        max_misses: int = 3,    # max  failed listens before check-in
        
    ) -> None:
        self._speaker    = speaker
        self._listener   = listener
        self._bus        = bus
        self._max_misses = max_misses

    @classmethod
    def build(cls) -> "Nexa":
        cfg      = Config()     #create config
        speaker  = Speaker(cfg.voice_index)     # create speaker
        listener = Listener(cfg)    # create listener

        skills: list[Skill] = [     # create all skills
            TimeSkill(),
            DateSkill(),
            WebSkill(),
            ReminderSkill(listener),
            Email(listener),
        ]

        return cls(speaker, listener, CommandBus(skills, speaker))  # assemble

    def run(self) -> None:
        self._speaker.say(_greeting())  # intial greeting
        consecutive_misses = 0

        while True:
            query = self._listener.listen()    # listen for command

            if not query:   # no input detection
                consecutive_misses += 1
                if consecutive_misses >= self._max_misses:  # 3 failures
                    self._speaker.say("Still there,?")
                    consecutive_misses = 0
                continue

            consecutive_misses = 0  # reset counter on successful input
            # check for exit commands

            if any(w in query for w in ("exit", "stop", "goodbye", "shut down")):
                self._speaker.say("Goodbye. Have a great day.")
                break
            
            # try to handle command
            if not self._bus.dispatch(query):   # create and start Nexa
                self._speaker.say("I'm not sure how to help with that yet,.")

# ──────────────────────────────────────────────
#   ENTRY POINT
# ──────────────────────────────────────────────
if __name__ == "__main__":
    Nexa.build().run()
