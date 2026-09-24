import math
import random
import re
import time
from typing import Any, List

from android.content import Context
from android.graphics import Bitmap, Canvas, Paint
from android.graphics.drawable import GradientDrawable
from android.hardware import Sensor, SensorEventListener, SensorManager
from android.os import Handler, Looper
from android.util import TypedValue
from android.view import MotionEvent, Surface, SurfaceView, TextureView, View, ViewGroup
from android.view.animation import AccelerateInterpolator, OvershootInterpolator
from android.widget import EditText, FrameLayout, ImageView, TextView
from androidx.recyclerview.widget import RecyclerView
from java import dynamic_proxy, jarray, jclass, jint
from org.telegram.messenger import AndroidUtilities, MediaController, MessagesController, NotificationCenter, UserObject
from org.telegram.ui import ChatActivity, LaunchActivity
from org.telegram.ui.ActionBar import Theme

from android_utils import R, log, run_on_ui_thread
from base_plugin import AppEvent, BasePlugin, HookResult, MenuItemData, MenuItemType
from client_utils import get_last_fragment
from ui.settings import Divider, Header, Selector, Switch, Text

try:
    from org.telegram.ui.Cells import ChatMessageCell
except Exception:
    ChatMessageCell = None

__id__ = "clawd_pet"
__name__ = "Clawd"
__description__ = (
    "пиксельный питомец Claude, который живёт поверх интерфейса.\n\n"
    "**Тап** — ткнуть, **двойной тап** — прыжок, **удержание** — погладить "
    "(води пальцем туда-сюда), **перетаскивание** — перенести, **резкий бросок** — полёт.\n\n"
    "Иногда съедает кнопки и иконки — тапни его, и он всё выплюнет обратно. "
    "Переевший Clawd оставляет кучку — тапни её, чтобы убрать.\n\n"
    "Замечает новые сообщения и твои ответы, радуется «привет», «люблю» и «ахаха», "
    "комментирует открытые чаты и танцует под музыку.\n\n"
    "**Потряси телефон** — Clawd полетит кувырком. С гравитацией съезжает при наклоне, "
    "стоит на пузырях сообщений и поле ввода, а если подбросить к статус-бару — повиснет на руках."
)
__author__ = "@itskotovski"
__version__ = "1.5"
__icon__ = "clawd_poop_5050_1783593604/0"

GRID_W = 18
GRID_H = 13
SIZE_ITEMS = ["Маленький", "Средний", "Большой"]
SIZE_DP = [3.0, 4.0, 5.0]

PALETTES = [
    ("Claude", 0xFFD97757, 0xFF1A1A1A),
    ("Мятный", 0xFF4FBF98, 0xFF12241D),
    ("Лавандовый", 0xFF9B8AEE, 0xFF1B1631),
    ("Небесный", 0xFF4FA8E8, 0xFF0E1C29),
    ("Розовый", 0xFFEE85AE, 0xFF2A1220),
    ("Графит", 0xFF5E6670, 0xFFF2F2F2),
]
SLEEP_ITEMS = ["Через 1 минуту", "Через 3 минуты", "Через 10 минут", "Никогда"]
SLEEP_SECONDS = [60, 180, 600, 0]

HEART_COLOR = 0xFFFF5C7A
ZZZ_COLOR = 0xFF8FA3BF
SPARK_COLOR = 0xFFFFC94D
NOTE_COLOR = 0xFF6FB8FF
POOP_COLORS = {"X": 0xFF8A5A33, "o": 0xFFB27D4E, "w": 0xFFFFFFFF, "k": 0xFF1A1A1A}

PATTERNS = {
    "heart": (
        ".XX.XX.",
        "XXXXXXX",
        "XXXXXXX",
        ".XXXXX.",
        "..XXX..",
        "...X...",
    ),
    "zzz": (
        "XXXXX",
        "...X.",
        "..X..",
        ".X...",
        "XXXXX",
    ),
    "spark": (
        "..X..",
        "..X..",
        "XX.XX",
        "..X..",
        "..X..",
    ),
    "note": (
        "..XXXXX",
        "..X...X",
        "..X...X",
        "..X...X",
        "XXX.XXX",
        "XXX.XXX",
    ),
    "poop": (
        "....X....",
        "...XXo...",
        "..XXXXX..",
        ".XXXXXXo.",
        ".XwkXwkX.",
        "XXXXXXXXo",
        "XXXXXXXXX",
    ),
}

PHRASES = {
    "hello": ["Привет!", "Я Clawd!", "О, это ты!"],
    "idle": [
        "Чем помочь?",
        "Хм…",
        "Думаю…",
        "Можно я тут посижу?",
        "Ты молодец!",
        "Не забудь попить воды",
        "Пип-пип!",
        "Мне нравится этот чат",
        "*оглядывается*",
        "Пишу код… шучу",
        "Интересно, что там в чатах",
    ],
    "tap": ["Ой!", "Эй!", "Чего?", "Я тут!", "Хи!"],
    "annoyed": ["Хватит тыкать!", "Щекотно же!", "Ну всё, обиделся", "Эй, полегче!"],
    "pet": ["Мрр…", "Ещё!", "Приятно ♥", "Хи-хи", "Лучший день!", "♥♥♥"],
    "drag": ["Ааа!", "Куда мы?", "Поставь меня!"],
    "throw": ["Уиии!", "Я лечу!", "Ваааа!"],
    "dizzy": ["Голова кружится…", "@_@", "Где я?"],
    "wake": ["А? Что?", "Я не спал!", "*зевает*"],
    "jump": ["Хоп!", "Йе!", "Ура!"],
    "eat": ["Ням!", "Хрум-хрум", "Вкусная кнопка", "*жуёт*", "Ммм, пиксели", "Это было лишнее", "Добавки!"],
    "spit": ["Ладно, отдаю!", "Тьфу!", "Ой, простите", "Забирай…", "Не бей!", "Оно само!"],
    "mail": ["Тебе пишут!", "Сообщение!", "Кто-то написал!", "Динь-динь!", "Смотри, новенькое!"],
    "sent": ["Отправлено!", "Улетело!", "Хорошо сказано!", "Пиу!", "Жду ответа!"],
    "essay": ["Ого, целое эссе", "Это роман?", "Много букв!", "Я не дочитал…"],
    "love": ["Ааа, мило ♥", "♥♥♥", "Как трогательно!", "Любовь!"],
    "laugh": ["Ахаха!", "Хи-хи-хи", "Смешно!", "Ха-ха!"],
    "hi": ["Привет-привет!", "И тебе привет!", "Здрасьте!", "*машет*"],
    "music": ["О, музыка!", "Танцуем!", "Это мой трек!", "♪♪♪"],
    "listen": ["Слушаю…", "Кто это говорит?", "*внимательно слушает*"],
    "music_end": ["Эх, закончилось", "Ещё!", "Хороший трек"],
    "chat_again": ["О, опять этот чат", "Снова сюда?", "Опять {name}?", "Мы тут уже были"],
    "chat_new": ["О, {name}!", "Это же {name}!", "Что пишет {name}?", "Привет, {name}!"],
    "chat_self": ["Твои заметки!", "Избранное, уютно", "Что сохраним?"],
    "shake": ["Аааа!", "Землетрясение!", "Хватит трясти!", "Уоооу!"],
    "slide": ["Скользко!", "Уиии~", "Меня несёт!"],
    "hang": ["Держусь!", "Помогите!", "Высоковато…", "Не отпущу!"],
    "hang_fall": ["Ой-ой-ой!", "Не удержался!", "Падаю!"],
    "peek": ["Ку-ку!", "Я тебя вижу", "*подглядывает*", "Ты тут?"],
    "found": ["Нашёл меня!", "Эх, раскусил", "Я просто смотрел!"],
    "poop": ["Упс…", "Это не я!", "Ой…", "Кнопки были лишними", "*краснеет*"],
    "clean": ["Спасибо!", "Прости…", "Больше не буду!", "Ты лучший"],
}

KEYWORDS = (
    ("love", re.compile(r"❤|♥|💕|💖|😘|😍|🥰|\bлюблю\b|\bобнима\w*|\bцелую\b|\blove\b")),
    ("laugh", re.compile(r"(?:ха){2,}|(?:хе){2,}|(?:хи){2,}|\bлол\b|\blol\b|\bржу\w*|😂|🤣")),
    ("hi", re.compile(r"\b(?:привет\w*|здравствуй\w*|хай|салют|hello|hi|hey)\b")),
)
MAX_KEYWORD_TEXT = 400
MAX_SCAN_MESSAGES = 10
MAX_CHAT_VISITS = 200

MAX_PARTICLES = 14
MAX_FRAME_CACHE = 48

MAX_EATEN = 5
EAT_SCAN_LIMIT = 900
EAT_COOLDOWN = (40.0, 120.0)

MAX_POOPS = 3
POOP_EVERY = (3, 5)

EARTH_G = 9.81
SHAKE_JERK = 14.0
SHAKE_HITS = 4
SHAKE_WINDOW = 0.9
TILT_DEAD = 0.22

PEEK_AFTER = 45.0
PEEK_HIDDEN_UNITS = 3
PEEK_OUT_UNITS = 7


def as_java_color(color: int) -> int:
    value = int(color) & 0xFFFFFFFF
    return value - 0x100000000 if value >= 0x80000000 else value


def mix_color(a: int, b: int, t: float) -> int:
    out = 0xFF000000
    for shift in (16, 8, 0):
        ca = (a >> shift) & 0xFF
        cb = (b >> shift) & 0xFF
        out |= int(ca + (cb - ca) * t) << shift
    return out


def clamp(value, lo, hi):
    if hi < lo:
        return lo
    return lo if value < lo else hi if value > hi else value


class PetTouchListener(dynamic_proxy(View.OnTouchListener)):
    def __init__(self, plugin):
        super().__init__()
        self.plugin = plugin

    def onTouch(self, view, event):
        try:
            return bool(self.plugin._on_touch(event))
        except Exception as e:
            log(f"[{__id__}] touch error: {e}")
            return False


class ShakeListener(dynamic_proxy(SensorEventListener)):
    def __init__(self, plugin):
        super().__init__()
        self.plugin = plugin

    def onSensorChanged(self, event):
        try:
            values = event.values
            self.plugin._on_accel(float(values[0]), float(values[1]), float(values[2]))
        except Exception as e:
            log(f"[{__id__}] sensor error: {e}")

    def onAccuracyChanged(self, sensor, accuracy):
        pass


class PoopClickListener(dynamic_proxy(View.OnClickListener)):
    def __init__(self, plugin):
        super().__init__()
        self.plugin = plugin

    def onClick(self, view):
        try:
            self.plugin._clean_poop(view)
        except Exception as e:
            log(f"[{__id__}] poop click error: {e}")


class NewMessagesDelegate(dynamic_proxy(NotificationCenter.NotificationCenterDelegate)):
    def __init__(self, plugin):
        super().__init__()
        self.plugin = plugin

    def didReceivedNotification(self, id, account, args):
        plugin = self.plugin
        if plugin is None or plugin._msg_delegate is not self:
            return
        try:
            plugin._on_notification(id, int(account), args)
        except Exception as e:
            log(f"[{__id__}] notification error: {e}")


class ClawdPetPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self._handler = None
        self._tick_r = None
        self._running = False
        self._paused = False
        self._unloaded = False
        self._cfg = {}

        self._decor = None
        self._pet = None
        self._bubble = None
        self._touch_listener = None
        self._msg_delegate = None
        self._observed_accounts = []
        self._chat_visits = {}
        self._sensor_manager = None
        self._sensor_listener = None
        self._poop_listener = None
        self._poops = []
        self._particles = []
        self._flying = []
        self._eaten = []
        self._spitting = []

        self._unit = 1
        self._w = 1
        self._h = 1
        self._frames = {}
        self._particle_bitmaps = {}
        self._frame_key = None

        self._reset_state()

    def on_plugin_load(self):
        self._reload_cfg()
        self._handler = Handler(Looper.getMainLooper())
        self._tick_r = R(self._tick)
        self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.DRAWER_MENU,
                text="Питомец Clawd",
                on_click=self._on_menu_toggle,
                item_id="clawd_toggle",
                icon="msg_emoji_cat",
            )
        )
        self.add_on_send_message_hook()
        self._msg_delegate = NewMessagesDelegate(self)
        self._on_ui(self._add_observers)
        run_on_ui_thread(lambda: self._start(greet=True))

    def on_plugin_unload(self):
        self._unloaded = True
        self._running = False
        if self._handler is not None and self._tick_r is not None:
            self._handler.removeCallbacks(self._tick_r)
        delegate = self._msg_delegate
        self._msg_delegate = None
        accounts = self._observed_accounts
        self._observed_accounts = []
        self._on_ui(lambda: self._remove_observers(delegate, accounts))
        self._on_ui(self._stop_sensor)
        self._on_ui(self._detach_views)

    def _add_observers(self):
        delegate = self._msg_delegate
        if delegate is None or self._unloaded:
            return
        for account in range(16):
            try:
                NotificationCenter.getInstance(account).addObserver(delegate, NotificationCenter.didReceiveNewMessages)
                self._observed_accounts.append(account)
            except Exception:
                break

    def _remove_observers(self, delegate, accounts):
        if delegate is None:
            return
        for account in accounts:
            try:
                NotificationCenter.getInstance(account).removeObserver(delegate, NotificationCenter.didReceiveNewMessages)
            except Exception as e:
                log(f"[{__id__}] remove observer error: {e}")

    def on_send_message_hook(self, account, params):
        try:
            text = getattr(params, "message", None)
            length = len(str(text)) if text is not None else 0
            run_on_ui_thread(lambda: self._on_sent(length))
        except Exception as e:
            log(f"[{__id__}] send hook error: {e}")
        return HookResult()

    def on_app_event(self, event_type):
        if event_type == AppEvent.PAUSE:
            self._paused = True
            if self._handler is not None and self._tick_r is not None:
                self._handler.removeCallbacks(self._tick_r)
            self._on_ui(self._stop_sensor)
        elif event_type == AppEvent.RESUME:
            self._paused = False
            run_on_ui_thread(lambda: self._start(greet=False))

    def create_settings(self) -> List[Any]:
        pets = int(self.get_setting("pets_count", 0) or 0)
        eaten = int(self.get_setting("eaten_count", 0) or 0)
        cleaned = int(self.get_setting("poops_cleaned", 0) or 0)
        return [
            Header(text="Питомец"),
            Switch(
                key="show",
                text="Показывать питомца",
                default=True,
                subtext="Также переключается пунктом в боковом меню",
                icon="msg_emoji_cat",
                on_change=lambda v: self._on_cfg_change("show", v),
            ),
            Selector(
                key="size",
                text="Размер",
                default=1,
                items=SIZE_ITEMS,
                icon="msg_photo_settings",
                on_change=lambda v: self._on_cfg_change("size", v),
            ),
            Selector(
                key="color",
                text="Окрас",
                default=0,
                items=[p[0] for p in PALETTES],
                icon="msg_colors",
                on_change=lambda v: self._on_cfg_change("color", v),
            ),
            Header(text="Поведение"),
            Switch(
                key="gravity",
                text="Гравитация",
                default=False,
                subtext="Падает вниз, стоит на пузырях сообщений и поле ввода, съезжает при наклоне",
                icon="msg_download",
                on_change=lambda v: self._on_cfg_change("gravity", v),
            ),
            Switch(
                key="wander",
                text="Гуляет сам",
                default=True,
                subtext="Ходит, прыгает, машет, а от скуки прячется за край экрана",
                icon="msg_location",
                on_change=lambda v: self._on_cfg_change("wander", v),
            ),
            Switch(
                key="phrases",
                text="Разговаривает",
                default=True,
                subtext="Иногда показывает реплики в облачке",
                icon="msg_discussion",
                on_change=lambda v: self._on_cfg_change("phrases", v),
            ),
            Selector(
                key="sleep",
                text="Засыпает",
                default=1,
                items=SLEEP_ITEMS,
                icon="msg_mute",
                on_change=lambda v: self._on_cfg_change("sleep", v),
            ),
            Switch(
                key="eat",
                text="Ест интерфейс",
                default=True,
                subtext="Иногда съедает кнопки и иконки. Тапни его — выплюнет обратно",
                icon="msg_emoji_food",
                on_change=lambda v: self._on_cfg_change("eat", v),
            ),
            Switch(
                key="poop",
                text="Какает",
                default=True,
                subtext="После 3–5 съеденных кнопок оставляет кучку. Тапни её — уберёшь",
                icon="msg_clear",
                on_change=lambda v: self._on_cfg_change("poop", v),
            ),
            Switch(
                key="shake",
                text="Чувствует тряску",
                default=True,
                subtext="Потряси телефон — Clawd полетит кувырком. Использует акселерометр",
                icon="msg_photo_rotate",
                on_change=lambda v: self._on_cfg_change("shake", v),
            ),
            Switch(
                key="tg_events",
                text="Реагирует на Telegram",
                default=True,
                subtext="Новые сообщения, твои ответы, «привет», «люблю», «ахаха» и открытые чаты",
                icon="msg_notifications",
                on_change=lambda v: self._on_cfg_change("tg_events", v),
            ),
            Switch(
                key="dance",
                text="Танцует под музыку",
                default=True,
                subtext="Пока играет трек или голосовое",
                icon="msg_voice_headphones",
                on_change=lambda v: self._on_cfg_change("dance", v),
            ),
            Switch(
                key="haptics",
                text="Вибрация",
                default=True,
                icon="msg_sticker",
                on_change=lambda v: self._on_cfg_change("haptics", v),
            ),
            Divider(),
            Text(
                text="Позвать в центр экрана",
                icon="msg_retry",
                accent=True,
                on_click=lambda _view=None: run_on_ui_thread(self._recenter),
            ),
            Text(
                text="Вернуть съеденное",
                icon="msg_reset",
                on_click=lambda _view=None: run_on_ui_thread(self._give_back),
            ),
            Text(
                text="Погладили",
                subtext=f"{pets} раз",
                icon="msg_reactions",
            ),
            Text(
                text="Съедено элементов",
                subtext=f"{eaten} шт.",
                icon="msg_stats",
            ),
            Text(
                text="Убрано кучек",
                subtext=f"{cleaned} шт.",
                icon="msg_delete",
            ),
            Divider(
                text="Тап — ткнуть, двойной тап — прыжок, удержание — погладить "
                     "(води пальцем туда-сюда), перетаскивание — перенести, резкий бросок — полёт. "
                     "Если Clawd что-то съел — тапни его, и он вернёт всё на место. "
                     "Потряси телефон, чтобы подбросить его, а если подкинуть к статус-бару — повиснет."
            ),
        ]

    def _reload_cfg(self):
        self._cfg = {
            "show": bool(self.get_setting("show", True)),
            "size": int(self.get_setting("size", 1) or 0),
            "color": int(self.get_setting("color", 0) or 0),
            "gravity": bool(self.get_setting("gravity", False)),
            "wander": bool(self.get_setting("wander", True)),
            "phrases": bool(self.get_setting("phrases", True)),
            "sleep": int(self.get_setting("sleep", 1) or 0),
            "haptics": bool(self.get_setting("haptics", True)),
            "eat": bool(self.get_setting("eat", True)),
            "tg_events": bool(self.get_setting("tg_events", True)),
            "dance": bool(self.get_setting("dance", True)),
            "poop": bool(self.get_setting("poop", True)),
            "shake": bool(self.get_setting("shake", True)),
        }

    def _on_cfg_change(self, key, value):
        self._cfg[key] = value
        if key == "show":
            run_on_ui_thread(lambda: self._apply_show(bool(value)))
        elif key in ("size", "color"):
            run_on_ui_thread(self._rebuild_sprites)
        elif key == "sleep" and self._state == "sleep":
            self._last_interaction = time.monotonic()
        elif key == "eat" and not value:
            run_on_ui_thread(self._give_back)
        elif key == "poop" and not value:
            run_on_ui_thread(self._remove_poops)
        elif key == "shake":
            run_on_ui_thread(self._start_sensor if value else self._stop_sensor)
        elif key == "gravity" and not value:
            self._platform = None
            self._slide_v = 0.0

    def _on_menu_toggle(self, context=None):
        show = not self._cfg.get("show", True)
        self._cfg["show"] = show
        self.set_setting("show", show)
        run_on_ui_thread(lambda: self._apply_show(show))

    def _apply_show(self, show):
        if show:
            self._start(greet=True)
        else:
            self._running = False
            if self._handler is not None:
                self._handler.removeCallbacks(self._tick_r)
            self._stop_sensor()
            self._detach_views()

    def _on_ui(self, fn):
        try:
            if Looper.myLooper() == Looper.getMainLooper():
                fn()
                return
        except Exception:
            pass
        run_on_ui_thread(fn)

    def _start(self, greet):
        if self._unloaded or self._paused or not self._cfg.get("show", True):
            return
        if greet:
            self._greet_pending = True
        try:
            self._ensure_attached()
        except Exception as e:
            log(f"[{__id__}] attach error: {e}")
        if self._pet is None:
            run_on_ui_thread(lambda: self._start(greet), 700)
            return
        self._running = True
        self._last_tick = time.monotonic()
        self._start_sensor()
        self._schedule(16)

    def _start_sensor(self):
        if (
            self._sensor_listener is not None
            or not self._running
            or self._paused
            or self._unloaded
            or not self._cfg.get("shake", True)
            or self._pet is None
        ):
            return
        try:
            manager = self._pet.getContext().getSystemService(Context.SENSOR_SERVICE)
            sensor = manager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER) if manager is not None else None
            if sensor is None:
                return
            listener = ShakeListener(self)
            if manager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_UI):
                self._sensor_manager = manager
                self._sensor_listener = listener
                self._accel_prev = None
                self._shake_hits = []
        except Exception as e:
            log(f"[{__id__}] sensor start error: {e}")

    def _stop_sensor(self):
        manager = self._sensor_manager
        listener = self._sensor_listener
        self._sensor_manager = None
        self._sensor_listener = None
        self._tilt = 0.0
        if manager is None or listener is None:
            return
        try:
            manager.unregisterListener(listener)
        except Exception as e:
            log(f"[{__id__}] sensor stop error: {e}")

    def _current_activity(self):
        activity = None
        try:
            activity = LaunchActivity.instance
        except Exception:
            activity = None
        if activity is None:
            try:
                fragment = get_last_fragment()
                activity = fragment.getParentActivity() if fragment is not None else None
            except Exception:
                activity = None
        return activity

    def _ensure_attached(self):
        activity = self._current_activity()
        if activity is None or activity.isFinishing():
            return
        decor = activity.getWindow().getDecorView()
        if (
            self._pet is not None
            and self._decor is not None
            and self._pet.getParent() is not None
            and self._decor.equals(decor)
        ):
            return
        self._detach_views()
        self._decor = decor
        self._build_views(activity)

    def _build_views(self, context):
        self._update_metrics()
        pet = ImageView(context)
        pet.setScaleType(ImageView.ScaleType.FIT_XY)
        pet.setClickable(True)
        self._touch_listener = PetTouchListener(self)
        pet.setOnTouchListener(self._touch_listener)
        self._decor.addView(pet, FrameLayout.LayoutParams(self._w, self._h))
        self._pet = pet

        bubble = TextView(context)
        bubble.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13.0)
        bubble.setMaxWidth(AndroidUtilities.dp(220))
        bubble.setPadding(AndroidUtilities.dp(10), AndroidUtilities.dp(6), AndroidUtilities.dp(10), AndroidUtilities.dp(6))
        try:
            bubble.setTypeface(AndroidUtilities.bold())
        except Exception:
            pass
        bubble.setVisibility(View.GONE)
        self._decor.addView(bubble, FrameLayout.LayoutParams(-2, -2))
        self._bubble = bubble
        self._bubble_visible = False

        self._frame_key = None
        self._pos_ready = False

    def _detach_views(self):
        self._restore_all()
        self._remove_poops()
        for view in list(self._particles):
            self._remove_view(view)
        self._particles = []
        self._remove_view(self._bubble)
        self._remove_view(self._pet)
        self._pet = None
        self._bubble = None
        self._decor = None
        self._touch_listener = None
        self._poop_listener = None
        self._touch = None
        self._platform = None
        self._platforms = []
        if self._state in ("press", "drag", "pet", "hang", "peek"):
            self._state = "idle"

    def _remove_view(self, view):
        if view is None:
            return
        try:
            view.animate().cancel()
        except Exception:
            pass
        try:
            parent = view.getParent()
            if parent is not None:
                parent.removeView(view)
        except Exception:
            pass

    def _update_metrics(self):
        size = clamp(int(self._cfg.get("size", 1)), 0, len(SIZE_DP) - 1)
        self._unit = max(2, AndroidUtilities.dp(SIZE_DP[size]))
        self._w = GRID_W * self._unit
        self._h = GRID_H * self._unit

    def _rebuild_sprites(self):
        old_w, old_h = self._w, self._h
        self._frames = {}
        self._particle_bitmaps = {}
        self._frame_key = None
        self._update_metrics()
        if self._pet is None:
            return
        self._x += (old_w - self._w) / 2.0
        self._y += old_h - self._h
        lp = self._pet.getLayoutParams()
        lp.width = self._w
        lp.height = self._h
        self._pet.setLayoutParams(lp)
        self._hop(AndroidUtilities.dp(300))
        self._schedule(16)

    def _palette(self):
        idx = clamp(int(self._cfg.get("color", 0)), 0, len(PALETTES) - 1)
        return PALETTES[idx]

    def _get_frame(self, key):
        bmp = self._frames.get(key)
        if bmp is None:
            if len(self._frames) >= MAX_FRAME_CACHE:
                self._frames.pop(next(iter(self._frames)))
            bmp = self._render_frame(key)
            self._frames[key] = bmp
        return bmp

    def _render_frame(self, key):
        eyes, look, arms, legs, squash, blush, mouth = key
        _, body_color, eye_color = self._palette()
        blush_color = mix_color(body_color, 0xFFFF4F79, 0.5)
        s = self._unit
        bmp = Bitmap.createBitmap(GRID_W * s, GRID_H * s, Bitmap.Config.ARGB_8888)
        canvas = Canvas(bmp)
        paint = Paint()

        def px(x0, y0, x1, y1, color):
            paint.setColor(jint(as_java_color(color)))
            canvas.drawRect(float(x0 * s), float(y0 * s), float(x1 * s), float(y1 * s), paint)

        left, right, top, bottom = 3, 15, 3, 10
        if squash > 0:
            left, right, top = 2, 16, 5
        elif squash < 0:
            left, right, top = 4, 14, 2
        if legs == "tuck":
            top += 1
            bottom += 1

        heights = {
            "stand": (2, 2, 2, 2),
            "walk1": (1, 2, 1, 2),
            "walk2": (2, 1, 2, 1),
            "dangle": (3, 3, 3, 3),
            "tuck": (1, 1, 1, 1),
        }.get(legs, (2, 2, 2, 2))
        for col, h in zip((left + 1, left + 3, right - 4, right - 2), heights):
            px(col, bottom, col + 1, bottom + h, body_color)

        def arm(side, style):
            if style == "up":
                if side < 0:
                    px(left - 2, top - 1, left - 1, top + 3, body_color)
                    px(left - 1, top + 2, left, top + 3, body_color)
                else:
                    px(right + 1, top - 1, right + 2, top + 3, body_color)
                    px(right, top + 2, right + 1, top + 3, body_color)
            elif style == "tuck":
                if side < 0:
                    px(left - 1, top + 3, left, top + 5, body_color)
                else:
                    px(right, top + 3, right + 1, top + 5, body_color)
            else:
                if side < 0:
                    px(left - 2, top + 3, left, top + 5, body_color)
                else:
                    px(right, top + 3, right + 2, top + 5, body_color)

        left_arm = right_arm = arms
        if arms == "wave1":
            left_arm, right_arm = "up", "normal"
        elif arms == "wave2":
            left_arm, right_arm = "normal", "up"
        arm(-1, left_arm)
        arm(1, right_arm)

        px(left, top, right, bottom, body_color)

        ey = top + 2
        for side, cx in ((-1, left + 3 + look), (1, right - 4 + look)):
            if eyes == "blink":
                px(cx, ey + 1, cx + 1, ey + 2, eye_color)
            elif eyes == "happy":
                px(cx - 1, ey + 1, cx, ey + 2, eye_color)
                px(cx, ey, cx + 1, ey + 1, eye_color)
                px(cx + 1, ey + 1, cx + 2, ey + 2, eye_color)
            elif eyes == "surprised":
                px(cx, ey - 1, cx + 1, ey + 2, eye_color)
            elif eyes == "dizzy":
                for dx, dy in ((-1, -1), (1, -1), (0, 0), (-1, 1), (1, 1)):
                    px(cx + dx, ey + dy, cx + dx + 1, ey + dy + 1, eye_color)
            elif eyes == "sleep":
                px(cx - 1, ey + 1, cx + 2, ey + 2, eye_color)
            elif eyes == "angry":
                px(cx, ey, cx + 1, ey + 2, eye_color)
                brow = cx - 1 if side < 0 else cx + 1
                px(brow, ey - 1, brow + 1, ey, eye_color)
            else:
                px(cx, ey, cx + 1, ey + 2, eye_color)
            if blush:
                if side < 0:
                    px(cx - 2, ey + 2, cx, ey + 3, blush_color)
                else:
                    px(cx + 1, ey + 2, cx + 3, ey + 3, blush_color)

        if mouth:
            mx = (left + right) // 2
            my = ey + 2
            if mouth == "open":
                px(mx - 1, my, mx + 1, max(my + 1, min(my + 2, bottom)), eye_color)
            else:
                px(mx - 1, my, mx + 1, my + 1, eye_color)
        return bmp

    def _particle_bitmap(self, kind):
        bmp = self._particle_bitmaps.get(kind)
        if bmp is not None:
            return bmp
        rows = PATTERNS[kind]
        if kind == "poop":
            colors = POOP_COLORS
            s = max(2, int(self._unit * 0.9))
        else:
            colors = {"X": {"heart": HEART_COLOR, "zzz": ZZZ_COLOR, "spark": SPARK_COLOR, "note": NOTE_COLOR}[kind]}
            s = max(2, int(self._unit * 0.75))
        bmp = Bitmap.createBitmap(len(rows[0]) * s, len(rows) * s, Bitmap.Config.ARGB_8888)
        canvas = Canvas(bmp)
        paints = {}
        for ch, color in colors.items():
            paint = Paint()
            paint.setColor(jint(as_java_color(color)))
            paints[ch] = paint
        for y, row in enumerate(rows):
            for x, ch in enumerate(row):
                paint = paints.get(ch)
                if paint is not None:
                    canvas.drawRect(float(x * s), float(y * s), float((x + 1) * s), float((y + 1) * s), paint)
        self._particle_bitmaps[kind] = bmp
        return bmp

    def _reset_state(self):
        now = time.monotonic()
        self._state = "idle"
        self._x = 0.0
        self._y = 0.0
        self._vx = 0.0
        self._vy = 0.0
        self._pos_ready = False
        self._last_tick = now
        self._last_interaction = now
        self._next_action = now + 4.0
        self._next_blink = now + 2.0
        self._blink_until = 0.0
        self._look = 0
        self._expr = None
        self._expr_until = 0.0
        self._squash_until = 0.0
        self._wave_until = 0.0
        self._hop_active = False
        self._hop_off = 0.0
        self._hop_vy = 0.0
        self._walk_target = 0.0
        self._touch = None
        self._last_tap = 0.0
        self._recent_taps = []
        self._next_heart = 0.0
        self._next_z = 0.0
        self._bubble_visible = False
        self._bubble_fade_in = False
        self._bubble_until = 0.0
        self._bubble_gen = 0
        self._last_phrase = None
        self._bounds_cache = None
        self._bounds_time = 0.0
        self._attach_check = 0.0
        self._greet_pending = False
        self._next_eat = now + 25.0
        self._mouth_open_until = 0.0
        self._chew_until = 0.0
        self._media = None
        self._media_check = 0.0
        self._next_note = 0.0
        self._next_msg_react = 0.0
        self._next_sent_react = 0.0
        self._chat_dialog = 0
        self._next_chat_comment = now + 8.0
        self._rotation = 0
        self._accel_prev = None
        self._shake_hits = []
        self._shake_pending = 0.0
        self._shake_dizzy = False
        self._tilt = 0.0
        self._slide_v = 0.0
        self._next_slide_say = 0.0
        self._platform = None
        self._platforms = []
        self._platform_scan = 0.0
        self._platform_moving = False
        self._walk_in = False
        self._hang_start = 0.0
        self._hang_until = 0.0
        self._peek_side = 0
        self._peek_pending = 0
        self._peek_phase = "hide"
        self._peek_phase_until = 0.0
        self._peek_end = 0.0
        self._peek_moving = False
        self._poop_due = 0.0
        self._poop_need = random.randint(*POOP_EVERY)
        self._platform_span = None
        self._next_peek = now + PEEK_AFTER

    def _schedule(self, delay):
        if self._handler is None or self._tick_r is None:
            return
        self._handler.removeCallbacks(self._tick_r)
        self._handler.postDelayed(self._tick_r, int(delay))

    def _tick(self):
        if not self._running or self._paused:
            return
        try:
            self._step()
        except Exception as e:
            log(f"[{__id__}] tick error: {e}")
        if self._running and not self._paused:
            self._schedule(self._next_delay())

    def _next_delay(self):
        if self._pet is None:
            return 1000
        now = time.monotonic()
        if (
            self._state in ("press", "drag", "pet", "fly", "walk")
            or self._hop_active
            or self._bubble_fade_in
            or self._walk_in
            or self._platform_moving
            or self._slide_v
            or self._shake_pending
            or now < self._squash_until
            or (self._expr == "dizzy" and now < self._expr_until)
        ):
            return 16
        if self._state in ("hang", "peek"):
            return 33
        if self._state == "sleep":
            return 250
        if self._media and self._state == "idle":
            return 33
        return 80

    def _dp(self, value):
        return float(AndroidUtilities.dp(value))

    def _bounds(self, now):
        if self._bounds_cache is not None and now - self._bounds_time < 0.4:
            return self._bounds_cache
        decor = self._decor
        width = decor.getWidth()
        height = decor.getHeight()
        top_inset = AndroidUtilities.statusBarHeight
        bottom_inset = AndroidUtilities.navigationBarHeight
        try:
            insets = decor.getRootWindowInsets()
            if insets is not None:
                top_inset = insets.getSystemWindowInsetTop()
                bottom_inset = insets.getSystemWindowInsetBottom()
        except Exception:
            pass
        try:
            display = decor.getDisplay()
            if display is not None:
                self._rotation = int(display.getRotation())
        except Exception:
            pass
        self._bounds_cache = (
            0.0,
            float(max(0, width - self._w)),
            float(top_inset),
            float(max(top_inset, height - bottom_inset - self._h)),
            width,
            height,
        )
        self._bounds_time = now
        return self._bounds_cache

    def _step(self):
        now = time.monotonic()
        dt = clamp(now - self._last_tick, 0.0, 0.05)
        self._last_tick = now

        if now >= self._attach_check:
            self._attach_check = now + 1.0
            self._ensure_attached()
            self._prune_eaten()
            if self._pet is not None and self._pos_ready:
                self._poll_chat(now)
        if self._pet is None or self._decor is None:
            return
        if self._pos_ready:
            self._poll_media(now)
        if self._decor.getWidth() <= 0:
            return

        min_x, max_x, min_y, max_y, width, height = self._bounds(now)
        if not self._pos_ready:
            self._restore_position(min_x, max_x, min_y, max_y)
            self._pos_ready = True
            if self._greet_pending:
                self._greet_pending = False
                self._hop(self._dp(420))
                self._say("hello")

        gravity = bool(self._cfg.get("gravity", False))
        state = self._state

        if state == "press" and self._touch is not None and now - self._touch["t0"] >= 0.4:
            self._start_pet(now)
            state = self._state

        if self._shake_pending and state not in ("press", "drag", "pet"):
            self._apply_shake(now, gravity)
            state = self._state

        if state == "pet":
            if now >= self._next_heart:
                self._next_heart = now + 0.45
                self._spawn_particle("heart")
                self._haptic(4)
        elif state == "fly":
            self._step_fly(dt, now, gravity, min_x, max_x, min_y, max_y)
        elif state == "hang":
            self._step_hang(now, gravity, min_x, max_x, min_y)
        elif state == "peek":
            self._step_peek(dt, now, width)
        elif state in ("idle", "walk", "sleep"):
            self._step_ground(dt, now, gravity, min_x, max_x, min_y, max_y)

        if self._hop_active:
            self._hop_off += self._hop_vy * dt
            self._hop_vy += self._dp(2600) * dt
            if self._hop_off >= 0.0:
                self._hop_off = 0.0
                self._hop_vy = 0.0
                self._hop_active = False
                self._squash_until = now + 0.1

        self._render(now)
        self._update_bubble(now, width, min_y)

    def _step_ground(self, dt, now, gravity, min_x, max_x, min_y, max_y):
        self._walk_in = False
        if self._state != "walk":
            if self._x < min_x - 0.5 or self._x > max_x + 0.5:
                step = self._dp(60) * dt
                if self._x < min_x:
                    self._x = min(min_x, self._x + step)
                    self._look = 1
                else:
                    self._x = max(max_x, self._x - step)
                    self._look = -1
                self._walk_in = True
            else:
                self._x = clamp(self._x, min_x, max_x)

        self._platform_moving = False
        if gravity:
            if self._platform is not None:
                if not self._follow_platform(min_y, max_y):
                    self._fall(self._dp(42) * self._look if self._state == "walk" else 0.0)
                    return
            elif self._y < max_y - 0.5 and not self._hop_active:
                self._fall(0.0)
                return
            else:
                self._y = min(self._y, max_y)
            if self._step_slide(dt, now, min_x, max_x):
                return
        else:
            self._y = clamp(self._y, min_y, max_y)
            self._slide_v = 0.0

        if self._state == "walk":
            direction = 1 if self._walk_target > self._x else -1
            self._look = direction
            self._x += direction * self._dp(42) * dt
            reached = (direction > 0 and self._x >= self._walk_target) or (direction < 0 and self._x <= self._walk_target)
            hit_edge = (direction < 0 and self._x <= min_x) or (direction > 0 and self._x >= max_x)
            if reached or hit_edge:
                self._x = clamp(self._x, min_x, max_x)
                self._state = "idle"
                self._next_action = now + random.uniform(2.5, 6.0)
                side = self._peek_pending
                self._peek_pending = 0
                edge = self._dp(4)
                if side and ((side < 0 and self._x <= min_x + edge) or (side > 0 and self._x >= max_x - edge)):
                    self._start_peek(now, side)
                    return
                self._save_position()
            return

        if self._state == "sleep":
            if now >= self._next_z:
                self._next_z = now + 1.6
                self._spawn_particle("zzz")
            return

        if now >= self._next_blink:
            self._blink_until = now + 0.13
            self._next_blink = now + (0.35 if random.random() < 0.15 else random.uniform(2.0, 5.5))

        sleep_idx = clamp(int(self._cfg.get("sleep", 1)), 0, len(SLEEP_SECONDS) - 1)
        timeout = SLEEP_SECONDS[sleep_idx]
        if (
            timeout
            and now - self._last_interaction > timeout
            and not self._hop_active
            and not self._media
            and not self._walk_in
        ):
            self._state = "sleep"
            self._look = 0
            self._next_z = now + 0.6
            self._hide_bubble()
            return

        if self._poop_due and now >= self._poop_due and not self._hop_active and not self._walk_in:
            self._do_poop(now, min_x, max_x)
            return

        if self._media:
            if now >= self._next_note:
                self._next_note = now + (0.8 if self._media == "music" else 1.5)
                self._spawn_particle("note")
            return

        if now < self._next_action or self._hop_active or self._walk_in:
            return
        self._next_action = now + random.uniform(3.5, 9.0)
        if (
            self._cfg.get("eat", True)
            and now >= self._next_eat
            and len(self._eaten) < MAX_EATEN
            and random.random() < 0.3
        ):
            if self._try_eat(now, min_y):
                return
            self._next_eat = now + 10.0
        roll = random.random()
        if not self._cfg.get("wander", True):
            if roll < 0.5:
                self._look = random.choice((-1, 0, 1))
            return
        if (
            self._platform is None
            and now >= self._next_peek
            and now - self._last_interaction > PEEK_AFTER
            and roll < 0.3
        ):
            side = -1 if self._x - min_x < max_x - self._x else 1
            self._walk_target = min_x if side < 0 else max_x
            self._peek_pending = side
            self._state = "walk"
            return
        if roll < 0.38:
            distance = random.uniform(self._dp(40), self._dp(160)) * random.choice((-1, 1))
            lo, hi = min_x, max_x
            if self._platform is not None and self._platform_span is not None:
                x0, x1 = self._platform_span
                lo = max(lo, x0 - self._unit * 4)
                hi = min(hi, x1 - self._unit * 14)
            target = clamp(self._x + distance, lo, hi)
            if abs(target - self._x) > self._dp(12):
                self._walk_target = target
                self._state = "walk"
        elif roll < 0.58:
            self._look = random.choice((-1, 0, 1))
        elif roll < 0.72:
            self._say("idle")
        elif roll < 0.84:
            self._hop(self._dp(360))
        else:
            self._wave_until = now + 1.6

    def _fall(self, vx):
        self._state = "fly"
        self._platform = None
        self._platforms = []
        self._platform_scan = 0.0
        self._vx = vx + self._slide_v
        self._vy = 0.0
        self._slide_v = 0.0

    def _tilt_accel(self):
        tilt = self._tilt if self._sensor_listener is not None else 0.0
        if abs(tilt) <= TILT_DEAD:
            return 0.0
        return (tilt - math.copysign(TILT_DEAD, tilt)) * self._dp(2200)

    def _step_slide(self, dt, now, min_x, max_x):
        if not self._hop_active:
            self._slide_v += self._tilt_accel() * dt
        self._slide_v *= math.exp(-2.5 * dt)
        if abs(self._slide_v) < self._dp(6):
            self._slide_v = 0.0
            return False
        self._x += self._slide_v * dt
        if self._x < min_x or self._x > max_x:
            self._x = clamp(self._x, min_x, max_x)
            if abs(self._slide_v) > self._dp(150):
                self._squash_until = now + 0.12
                self._haptic(3)
            self._slide_v = 0.0
            return False
        if abs(self._slide_v) < self._dp(60):
            return False
        if self._state == "sleep":
            self._set_expr("surprised", 0.8)
        self._state = "idle"
        self._peek_pending = 0
        self._look = 1 if self._slide_v > 0 else -1
        if now >= self._next_slide_say:
            self._next_slide_say = now + 8.0
            if random.random() < 0.5:
                self._say("slide")
        return True

    def _feet_on(self, x0, x1):
        feet0 = self._x + self._unit * 4
        feet1 = self._x + self._unit * 14
        return min(feet1, x1) - max(feet0, x0) >= self._unit * 2

    def _platform_rect(self, view, kind, dloc=None):
        try:
            if view is None or not view.isAttachedToWindow() or view.getVisibility() != View.VISIBLE:
                return None
            if dloc is None:
                dloc = jarray(jint)([0, 0])
                self._decor.getLocationInWindow(dloc)
            loc = jarray(jint)([0, 0])
            view.getLocationInWindow(loc)
            x = float(loc[0] - dloc[0])
            y = float(loc[1] - dloc[1])
            if kind == "msg":
                left = view.getBackgroundDrawableLeft()
                right = view.getBackgroundDrawableRight()
                top = view.getBackgroundDrawableTop()
                if right - left < self._dp(40):
                    return None
                return (x + left, x + right, y + top)
            return (x, x + view.getWidth(), y)
        except Exception:
            return None

    def _scan_platforms(self, min_y, max_y):
        result = []
        try:
            fragment = get_last_fragment()
            if fragment is None or not isinstance(fragment, ChatActivity):
                return result
            dloc = jarray(jint)([0, 0])
            self._decor.getLocationInWindow(dloc)
            views = []
            enter = fragment.getChatActivityEnterView()
            if enter is not None:
                views.append((enter, "input"))
            listing = fragment.getChatListView()
            if listing is not None and ChatMessageCell is not None:
                for i in range(listing.getChildCount()):
                    child = listing.getChildAt(i)
                    if child is not None and isinstance(child, ChatMessageCell):
                        views.append((child, "msg"))
            lowest = min_y - self._unit * 2 + self._h
            highest = max_y + self._h
            for view, kind in views:
                rect = self._platform_rect(view, kind, dloc)
                if rect is None:
                    continue
                x0, x1, top = rect
                if x1 - x0 >= self._w * 0.6 and lowest <= top <= highest:
                    result.append((view, kind, x0, x1, top))
        except Exception as e:
            log(f"[{__id__}] platform scan error: {e}")
        return result

    def _follow_platform(self, min_y, max_y):
        view, kind = self._platform
        rect = self._platform_rect(view, kind)
        if rect is None:
            return False
        x0, x1, top = rect
        if not self._feet_on(x0, x1):
            return False
        y = top - self._h
        if y < min_y - self._unit * 2 or y > max_y:
            return False
        self._platform_moving = abs(y - self._y) > 0.5
        self._platform_span = (x0, x1)
        self._y = y
        return True

    def _platform_landing(self, now, old_y, min_y, max_y):
        if now >= self._platform_scan:
            self._platform_scan = now + 0.25
            self._platforms = self._scan_platforms(min_y, max_y)
        prev_bottom = old_y + self._h
        new_bottom = self._y + self._h
        best = None
        for item in self._platforms:
            top = item[4]
            if prev_bottom <= top + 1 and new_bottom >= top and self._feet_on(item[2], item[3]):
                if best is None or top < best[4]:
                    best = item
        if best is None:
            return None
        view, kind, x0, x1, top = best
        speed = self._vy
        self._y = top - self._h
        if speed > self._dp(380):
            self._vy = -speed * 0.42
            self._vx *= 0.75
            self._squash_until = now + 0.1
        else:
            self._platform_span = (x0, x1)
            self._land(now, (view, kind))
        return speed

    def _step_fly(self, dt, now, gravity, min_x, max_x, min_y, max_y):
        old_y = self._y
        self._x += self._vx * dt
        self._y += self._vy * dt
        if gravity:
            self._vy += self._dp(2600) * dt
            self._vx *= math.exp(-0.35 * dt)
            self._vx += self._tilt_accel() * dt
        else:
            friction = math.exp(-2.0 * dt)
            self._vx *= friction
            self._vy *= friction

        impact = 0.0
        if self._x < min_x:
            self._x = min_x
            impact = max(impact, abs(self._vx))
            self._vx = abs(self._vx) * 0.55
        elif self._x > max_x:
            self._x = max_x
            impact = max(impact, abs(self._vx))
            self._vx = -abs(self._vx) * 0.55

        landed = None
        if gravity and self._vy > 0 and self._y > old_y:
            landed = self._platform_landing(now, old_y, min_y, max_y)
        if landed is not None:
            impact = max(impact, landed)
        elif self._y < min_y:
            if self._vy < -self._dp(250) and not self._shake_dizzy:
                self._start_hang(now, min_y)
                return
            self._y = min_y
            impact = max(impact, abs(self._vy))
            self._vy = abs(self._vy) * 0.55
        elif self._y > max_y:
            self._y = max_y
            impact = max(impact, abs(self._vy))
            if gravity and self._vy < self._dp(380):
                self._land(now)
            else:
                self._vy = -abs(self._vy) * (0.42 if gravity else 0.55)
                self._vx *= 0.75 if gravity else 1.0
                self._squash_until = now + 0.1

        if impact > self._dp(2300):
            self._set_expr("dizzy", 2.6)
            self._say("dizzy")
            self._haptic(1)
        elif impact > self._dp(900):
            self._haptic(3)

        if self._state == "fly" and not gravity and math.hypot(self._vx, self._vy) < self._dp(45):
            self._land(now)

    def _land(self, now, platform=None):
        self._state = "idle"
        self._vx = 0.0
        self._vy = 0.0
        self._platform = platform
        self._platforms = []
        self._squash_until = now + 0.14
        self._next_action = now + random.uniform(2.0, 5.0)
        if self._shake_dizzy:
            self._shake_dizzy = False
            self._set_expr("dizzy", 3.0)
            self._say("dizzy")
            if self._bounds_cache is not None:
                min_x, max_x = self._bounds_cache[:2]
                distance = random.uniform(self._dp(30), self._dp(70)) * random.choice((-1, 1))
                self._walk_target = clamp(self._x + distance, min_x, max_x)
                if abs(self._walk_target - self._x) > self._dp(12):
                    self._state = "walk"
        self._save_position()

    def _on_accel(self, ax, ay, az):
        if not self._running or self._paused or self._pet is None:
            return
        now = time.monotonic()
        prev = self._accel_prev
        self._accel_prev = (ax, ay, az)
        if prev is not None and self._state not in ("press", "drag", "pet"):
            jerk = math.sqrt((ax - prev[0]) ** 2 + (ay - prev[1]) ** 2 + (az - prev[2]) ** 2)
            if jerk > SHAKE_JERK:
                hits = [t for t in self._shake_hits if now - t < SHAKE_WINDOW]
                hits.append(now)
                if len(hits) >= SHAKE_HITS:
                    hits = []
                    self._shake_pending = max(self._shake_pending, jerk)
                    self._schedule(0)
                self._shake_hits = hits
        rotation = self._rotation
        if rotation == Surface.ROTATION_90:
            sx, sy = ay, ax
        elif rotation == Surface.ROTATION_180:
            sx, sy = ax, -ay
        elif rotation == Surface.ROTATION_270:
            sx, sy = -ay, -ax
        else:
            sx, sy = -ax, ay
        raw = clamp(sx / EARTH_G, -1.0, 1.0) if sy > 2.0 else 0.0
        self._tilt = self._tilt * 0.7 + raw * 0.3

    def _apply_shake(self, now, gravity):
        power = min(self._shake_pending, 40.0)
        self._shake_pending = 0.0
        was_sleeping = self._state == "sleep"
        if self._state == "peek":
            self._peek_side = 0
        self._peek_pending = 0
        self._platform = None
        self._platforms = []
        self._platform_scan = 0.0
        self._hop_active = False
        self._hop_off = 0.0
        self._hop_vy = 0.0
        self._slide_v = 0.0
        self._state = "fly"
        speed = self._dp(900 + power * 55)
        angle = random.uniform(0.0, math.pi * 2.0)
        self._vx = math.cos(angle) * speed
        self._vy = math.sin(angle) * speed - (self._dp(700) if gravity else 0.0)
        self._shake_dizzy = True
        self._last_interaction = now
        self._set_expr("surprised", 1.0)
        self._haptic(1)
        if was_sleeping:
            self._say("wake")
        elif random.random() < 0.6:
            self._say("shake")

    def _start_hang(self, now, min_y):
        self._state = "hang"
        self._vx = 0.0
        self._vy = 0.0
        self._platform = None
        self._y = min_y - self._unit * 2
        self._hang_start = now
        self._hang_until = now + random.uniform(2.0, 3.5)
        self._haptic(3)
        if random.random() < 0.7:
            self._say("hang")

    def _step_hang(self, now, gravity, min_x, max_x, min_y):
        self._x = clamp(self._x, min_x, max_x)
        self._y = min_y - self._unit * 2
        if now < self._hang_until:
            return
        self._state = "fly"
        self._vx = 0.0
        self._vy = self._dp(150) if gravity else self._dp(700)
        self._set_expr("surprised", 0.8)
        self._haptic(3)
        if random.random() < 0.6:
            self._say("hang_fall")

    def _peek_x(self, width, units):
        if self._peek_side > 0:
            return float(width - units * self._unit)
        return float(units * self._unit - self._w)

    def _start_peek(self, now, side):
        self._state = "peek"
        self._peek_side = side
        self._peek_phase = "hide"
        self._peek_phase_until = now + random.uniform(3.0, 6.0)
        self._peek_end = now + random.uniform(30.0, 60.0)
        self._look = side

    def _step_peek(self, dt, now, width):
        phase = self._peek_phase
        target = self._peek_x(width, PEEK_OUT_UNITS if phase == "out" else PEEK_HIDDEN_UNITS)
        step = self._dp(50) * dt
        self._peek_moving = abs(target - self._x) > 0.5
        if self._peek_moving:
            self._x = min(target, self._x + step) if target > self._x else max(target, self._x - step)
        if phase == "hide" and self._peek_moving:
            self._look = self._peek_side
        else:
            self._look = -self._peek_side
        if now >= self._next_blink:
            self._blink_until = now + 0.13
            self._next_blink = now + random.uniform(2.0, 5.5)
        if now < self._peek_phase_until or self._peek_moving:
            return
        if now >= self._peek_end:
            self._end_peek(now)
            return
        if phase == "hide":
            self._peek_phase = "out"
            self._peek_phase_until = now + random.uniform(1.5, 2.8)
            if random.random() < 0.5:
                self._say("peek")
        else:
            self._peek_phase = "hide"
            self._peek_phase_until = now + random.uniform(3.0, 7.0)

    def _end_peek(self, now):
        side = self._peek_side
        self._peek_side = 0
        self._peek_moving = False
        self._next_peek = now + random.uniform(60.0, 120.0)
        self._state = "idle"
        if self._bounds_cache is None or not side:
            return
        min_x, max_x = self._bounds_cache[:2]
        inside = min_x + self._dp(30) if side < 0 else max_x - self._dp(30)
        self._walk_target = clamp(inside, min_x, max_x)
        self._state = "walk"
        self._next_action = now + random.uniform(3.0, 6.0)

    def _compose(self, now):
        eyes = "open"
        look = self._look
        arms = "normal"
        legs = "stand"
        squash = 0
        blush = False
        mouth = None
        state = self._state

        if state in ("drag", "fly"):
            eyes, arms, legs = "surprised", "up", "dangle"
        elif state == "pet":
            eyes, blush = "happy", True
            squash = 1 if int(now * 3) % 2 else 0
            look = 0
        elif state == "sleep":
            eyes, arms, legs, look = "sleep", "tuck", "tuck", 0
        elif state == "walk":
            legs = "walk1" if int(now * 7) % 2 else "walk2"
        elif state == "press":
            squash = 1
        elif state == "hang":
            eyes = "surprised" if now - self._hang_start < 0.5 else "open"
            arms, legs = "up", "dangle"
            look = -1 if int(now * 1.5) % 2 else 1
        elif state == "peek":
            if self._peek_moving:
                legs = "walk1" if int(now * 7) % 2 else "walk2"
            elif self._peek_phase == "out" and int(now * 3) % 2:
                arms = "wave1" if self._peek_side > 0 else "wave2"

        if self._walk_in and state == "idle":
            legs = "walk1" if int(now * 7) % 2 else "walk2"

        if state == "idle" and self._media and not self._hop_active:
            beat = int(now * 4) % 2
            eyes = "happy"
            if self._media == "music":
                arms = "wave1" if beat else "wave2"
                legs = "walk1" if beat else "walk2"
                squash = 1 if beat else 0
                look = -1 if int(now * 2) % 2 else 1
            else:
                look = 0

        if state in ("idle", "walk") and now < self._wave_until:
            arms = "wave1" if int(now * 5) % 2 else "wave2"

        if state in ("idle", "walk") and abs(self._slide_v) > self._dp(60):
            eyes, arms, legs = "surprised", "up", "stand"

        if self._hop_active and state in ("idle", "walk"):
            legs = "dangle"
            if self._hop_vy < 0:
                squash = -1
                arms = "up"

        if now < self._expr_until and self._expr and state != "sleep":
            eyes = self._expr
            if self._expr == "happy":
                blush = True
        if now < self._squash_until and squash == 0:
            squash = 1
        if eyes == "open" and state in ("idle", "walk", "peek", "hang") and now < self._blink_until:
            eyes = "blink"
        if legs == "tuck" and squash:
            squash = 0
        if state != "sleep":
            if now < self._mouth_open_until:
                mouth = "open"
            elif now < self._chew_until and int(now * 6) % 2:
                mouth = "chew"
        return (eyes, look, arms, legs, squash, blush, mouth)

    def _render(self, now):
        key = self._compose(now)
        if key != self._frame_key:
            self._frame_key = key
            self._pet.setImageBitmap(self._get_frame(key))
        self._pet.setTranslationX(float(self._x))
        self._pet.setTranslationY(float(self._y + self._hop_off))

        rotation = 0.0
        if self._state == "fly":
            rotation = clamp(self._vx / self._dp(70), -25.0, 25.0)
        elif self._state == "hang":
            rotation = math.sin((now - self._hang_start) * 3.0) * 6.0
        elif self._slide_v:
            rotation = clamp(-self._slide_v / self._dp(30), -12.0, 12.0)
        elif self._expr == "dizzy" and now < self._expr_until:
            rotation = math.sin(now * 10.0) * 7.0
        elif self._media and self._state == "idle" and not self._hop_active:
            if self._media == "music":
                rotation = math.sin(now * math.pi * 2.0) * 6.0
            else:
                rotation = math.sin(now * 2.5) * 3.0
        elif self._state == "drag" and self._touch is not None:
            if now - self._touch["hist"][-1][0] < 0.08:
                rotation = clamp(self._touch.get("vx", 0.0) / self._dp(90), -18.0, 18.0)
        self._pet.setRotation(float(rotation))

    def _restore_position(self, min_x, max_x, min_y, max_y):
        fx = self.get_setting("pos_x", None)
        fy = self.get_setting("pos_y", None)
        try:
            fx = float(fx) / 1000.0
            fy = float(fy) / 1000.0
        except Exception:
            fx, fy = 0.8, 0.72
        self._x = min_x + (max_x - min_x) * clamp(fx, 0.0, 1.0)
        self._y = min_y + (max_y - min_y) * clamp(fy, 0.0, 1.0)

    def _save_position(self):
        if self._bounds_cache is None:
            return
        min_x, max_x, min_y, max_y = self._bounds_cache[:4]
        fx = (self._x - min_x) / max(1.0, max_x - min_x)
        fy = (self._y - min_y) / max(1.0, max_y - min_y)
        try:
            self.set_setting("pos_x", int(clamp(fx, 0.0, 1.0) * 1000))
            self.set_setting("pos_y", int(clamp(fy, 0.0, 1.0) * 1000))
        except Exception as e:
            log(f"[{__id__}] save position error: {e}")

    def _recenter(self):
        if not self._cfg.get("show", True):
            self._cfg["show"] = True
            self.set_setting("show", True)
        self._start(greet=False)
        if self._decor is None or self._decor.getWidth() <= 0:
            return
        now = time.monotonic()
        min_x, max_x, min_y, max_y = self._bounds(now)[:4]
        self._x = (min_x + max_x) / 2.0
        self._y = (min_y + max_y) / 2.0
        self._pos_ready = True
        self._state = "idle"
        self._vx = self._vy = 0.0
        self._slide_v = 0.0
        self._platform = None
        self._peek_side = 0
        self._peek_pending = 0
        self._last_interaction = now
        self._hop(self._dp(500))
        for _ in range(3):
            self._spawn_particle("spark")
        self._say("hello")
        self._save_position()
        self._schedule(16)

    def _on_touch(self, event):
        action = event.getActionMasked()
        rx = float(event.getRawX())
        ry = float(event.getRawY())
        now = time.monotonic()

        if action == MotionEvent.ACTION_DOWN:
            was_sleeping = self._state == "sleep"
            was_peeking = self._state == "peek"
            self._last_interaction = now
            self._touch = {
                "x0": rx, "y0": ry, "t0": now,
                "ox": rx - self._x, "oy": ry - self._y,
                "hist": [(now, rx, ry)],
                "anchor": rx, "dir": 0, "vx": 0.0,
            }
            self._hop_active = False
            self._hop_off = 0.0
            self._hop_vy = 0.0
            self._vx = self._vy = 0.0
            self._slide_v = 0.0
            self._peek_side = 0
            self._peek_pending = 0
            self._state = "press"
            if was_sleeping:
                self._set_expr("surprised", 0.9)
                self._say("wake")
            elif was_peeking:
                self._next_peek = now + random.uniform(60.0, 120.0)
                self._set_expr("surprised", 0.7)
                self._say("found")
            self._schedule(0)
            return True

        touch = self._touch
        if touch is None:
            return False

        if action == MotionEvent.ACTION_MOVE:
            hist = touch["hist"]
            hist.append((now, rx, ry))
            while len(hist) > 2 and now - hist[0][0] > 0.1:
                hist.pop(0)
            if self._state == "press":
                if math.hypot(rx - touch["x0"], ry - touch["y0"]) > self._dp(8):
                    self._state = "drag"
                    self._platform = None
                    self._haptic(3)
                    if random.random() < 0.25:
                        self._say("drag")
            if self._state == "drag":
                width = self._decor.getWidth() if self._decor is not None else 0
                height = self._decor.getHeight() if self._decor is not None else 0
                self._x = clamp(rx - touch["ox"], -self._w * 0.3, width - self._w * 0.7)
                self._y = clamp(ry - touch["oy"], 0.0, height - self._h * 0.5)
                t0, x0, _ = hist[0]
                if now - t0 > 0.005:
                    touch["vx"] = (rx - x0) / (now - t0)
            elif self._state == "pet":
                dx = rx - touch["anchor"]
                if abs(dx) > self._dp(10):
                    direction = 1 if dx > 0 else -1
                    if touch["dir"] and direction != touch["dir"]:
                        self._stroke()
                    touch["dir"] = direction
                    touch["anchor"] = rx
            return True

        if action in (MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL):
            self._touch = None
            self._last_interaction = now
            state = self._state
            if state == "drag":
                self._release_drag(now, touch, action == MotionEvent.ACTION_CANCEL)
            elif state == "pet":
                self._end_pet(now)
            elif state == "press":
                self._state = "idle"
                if action == MotionEvent.ACTION_UP:
                    self._tap(now)
            self._schedule(0)
            return True
        return True

    def _release_drag(self, now, touch, cancelled):
        vx = vy = 0.0
        hist = touch["hist"]
        if not cancelled and len(hist) >= 2:
            t0, x0, y0 = hist[0]
            t1, x1, y1 = hist[-1]
            if t1 - t0 > 0.005 and now - t1 < 0.08:
                vx = (x1 - x0) / (t1 - t0)
                vy = (y1 - y0) / (t1 - t0)
        limit = self._dp(6000)
        vx = clamp(vx, -limit, limit)
        vy = clamp(vy, -limit, limit)
        speed = math.hypot(vx, vy)
        min_y = self._bounds_cache[2] if self._bounds_cache is not None else None
        if speed <= self._dp(700) and min_y is not None and self._y <= min_y + self._unit * 2:
            self._start_hang(now, min_y)
        elif speed > self._dp(700) or self._cfg.get("gravity", False):
            self._state = "fly"
            self._vx = vx
            self._vy = vy
            if speed > self._dp(2600):
                self._say("throw")
        else:
            self._state = "idle"
            self._squash_until = now + 0.12
            self._next_action = now + random.uniform(2.0, 5.0)
            self._save_position()

    def _tap(self, now):
        if self._eaten:
            self._last_tap = 0.0
            self._recent_taps = []
            self._spit_all(now)
            return
        if now - self._last_tap < 0.32:
            self._last_tap = 0.0
            self._hop(self._dp(720))
            self._set_expr("happy", 1.3)
            for _ in range(3):
                self._spawn_particle("spark")
            self._haptic(0)
            if random.random() < 0.4:
                self._say("jump")
            return
        self._last_tap = now
        self._recent_taps = [t for t in self._recent_taps if now - t < 2.5] + [now]
        if len(self._recent_taps) >= 6:
            self._recent_taps = []
            self._set_expr("angry", 2.5)
            self._say("annoyed", force=True)
            self._haptic(1)
            return
        self._squash_until = now + 0.12
        self._set_expr("surprised", 0.45)
        self._hop(self._dp(330))
        self._haptic(3)
        if random.random() < 0.35:
            self._say("tap")

    def _start_pet(self, now):
        self._state = "pet"
        self._next_heart = now + 0.1
        self._haptic(0)

    def _stroke(self):
        self._spawn_particle("heart")
        self._haptic(4)

    def _end_pet(self, now):
        self._state = "idle"
        self._set_expr("happy", 2.2)
        self._next_action = now + random.uniform(3.0, 6.0)
        try:
            count = int(self.get_setting("pets_count", 0) or 0) + 1
            self.set_setting("pets_count", count)
        except Exception:
            pass
        if random.random() < 0.6:
            self._say("pet")

    def _can_react(self):
        return (
            self._running
            and not self._paused
            and self._pet is not None
            and self._pos_ready
            and self._cfg.get("tg_events", True)
            and self._state in ("idle", "walk", "sleep", "peek")
        )

    def _wake_for_event(self, now):
        self._last_interaction = now
        if self._state == "peek":
            self._end_peek(now)
        elif self._state == "sleep":
            self._state = "idle"
            self._set_expr("surprised", 0.9)

    def _on_notification(self, id, account, args):
        if id != NotificationCenter.didReceiveNewMessages or not self._can_react():
            return
        if args is None or len(args) < 2:
            return
        if len(args) > 2 and bool(args[2]):
            return
        if len(args) > 3 and args[3] is not None and int(args[3]) != 0:
            return
        dialog_id = int(args[0])
        messages = args[1]
        texts = []
        incoming = False
        for i in range(min(int(messages.size()), MAX_SCAN_MESSAGES)):
            message = messages.get(i)
            if message is None or message.isOut():
                continue
            incoming = True
            text = message.messageOwner.message if message.messageOwner is not None else None
            if text:
                texts.append(str(text)[:MAX_KEYWORD_TEXT].lower())
        if not incoming:
            return
        in_chat = dialog_id == self._chat_dialog
        if not in_chat:
            try:
                if MessagesController.getInstance(account).isDialogMuted(dialog_id, 0):
                    return
            except Exception:
                pass
        self._react_incoming(self._match_keyword(texts), in_chat)

    def _match_keyword(self, texts):
        for mood, pattern in KEYWORDS:
            for text in texts:
                if pattern.search(text):
                    return mood
        return None

    def _react_incoming(self, mood, in_chat):
        now = time.monotonic()
        if mood is None and in_chat:
            return
        if now < self._next_msg_react:
            return
        self._next_msg_react = now + (3.0 if mood else 6.0)
        self._wake_for_event(now)
        if mood == "love":
            self._set_expr("happy", 2.2)
            for _ in range(3):
                self._spawn_particle("heart")
            self._say("love")
        elif mood == "laugh":
            self._set_expr("happy", 1.6)
            self._hop(self._dp(420))
            self._spawn_particle("spark")
            self._say("laugh")
        elif mood == "hi":
            self._wave_until = now + 1.8
            self._say("hi")
        else:
            self._look = random.choice((-1, 1))
            self._set_expr("surprised", 0.7)
            self._hop(self._dp(300))
            self._say("mail")
        self._next_action = max(self._next_action, now + 2.5)
        self._schedule(0)

    def _on_sent(self, length):
        if not self._can_react():
            return
        now = time.monotonic()
        if now < self._next_sent_react:
            return
        self._next_sent_react = now + 1.5
        self._wake_for_event(now)
        self._set_expr("happy", 1.2)
        self._hop(self._dp(520))
        for _ in range(2):
            self._spawn_particle("spark")
        if length > 300:
            self._say("essay")
        elif random.random() < 0.3:
            self._say("sent")
        self._next_action = max(self._next_action, now + 2.5)
        self._schedule(0)

    def _poll_media(self, now):
        if now < self._media_check:
            return
        self._media_check = now + 0.5
        kind = None
        if self._cfg.get("dance", True):
            try:
                controller = MediaController.getInstance()
                playing = controller.getPlayingMessageObject()
                if playing is not None and controller.isPlayingMessage(playing) and not controller.isMessagePaused():
                    if playing.isMusic():
                        kind = "music"
                    elif playing.isVoice() or playing.isRoundVideo():
                        kind = "voice"
            except Exception:
                kind = None
        if kind == self._media:
            return
        previous = self._media
        self._media = kind
        if kind is None:
            self._last_interaction = now
            if previous == "music" and self._state == "idle" and random.random() < 0.5:
                self._say("music_end")
            self._next_action = now + random.uniform(2.0, 4.0)
            return
        self._last_interaction = now
        self._next_note = now + 0.3
        if self._state == "peek":
            self._end_peek(now)
        elif self._state == "sleep":
            self._state = "idle"
            self._set_expr("surprised", 0.8)
        if previous is None and self._state in ("idle", "walk"):
            self._say("music" if kind == "music" else "listen")

    def _poll_chat(self, now):
        fragment = None
        dialog_id = 0
        try:
            fragment = get_last_fragment()
            if fragment is not None and isinstance(fragment, ChatActivity):
                dialog_id = int(fragment.getDialogId())
        except Exception:
            dialog_id = 0
        if dialog_id == self._chat_dialog:
            return
        self._chat_dialog = dialog_id
        if not dialog_id or not self._cfg.get("tg_events", True):
            return
        visits = self._chat_visits.pop(dialog_id, 0) + 1
        self._chat_visits[dialog_id] = visits
        if len(self._chat_visits) > MAX_CHAT_VISITS:
            self._chat_visits.pop(next(iter(self._chat_visits)))
        if now < self._next_chat_comment or self._state not in ("idle", "walk") or random.random() > 0.4:
            return
        self._next_chat_comment = now + 25.0
        is_self, name = self._chat_name(fragment)
        if is_self:
            self._say("chat_self")
        elif visits > 1:
            self._say("chat_again", name=name)
        else:
            self._say("chat_new", name=name)

    def _chat_name(self, fragment):
        name = None
        try:
            user = fragment.getCurrentUser()
            if user is not None:
                if UserObject.isUserSelf(user):
                    return True, None
                name = UserObject.getFirstName(user)
            else:
                chat = fragment.getCurrentChat()
                if chat is not None:
                    name = chat.title
        except Exception:
            name = None
        name = str(name).strip() if name else ""
        if len(name) > 20:
            name = name[:19].rstrip() + "…"
        return False, (name or None)

    def _set_expr(self, expr, duration):
        self._expr = expr
        self._expr_until = time.monotonic() + duration

    def _hop(self, strength):
        if self._state not in ("idle", "walk"):
            return
        self._hop_active = True
        self._hop_vy = -float(strength)
        if self._hop_off > 0:
            self._hop_off = 0.0

    def _haptic(self, kind):
        if not self._cfg.get("haptics", True) or self._pet is None:
            return
        try:
            self._pet.performHapticFeedback(kind)
        except Exception:
            pass

    def _spawn_particle(self, kind, origin=None):
        if self._decor is None or self._pet is None or len(self._particles) >= MAX_PARTICLES:
            return
        try:
            bmp = self._particle_bitmap(kind)
            bw, bh = bmp.getWidth(), bmp.getHeight()
            view = ImageView(self._pet.getContext())
            view.setImageBitmap(bmp)
            self._decor.addView(view, FrameLayout.LayoutParams(bw, bh))
            self._particles.append(view)

            head_y = self._y + self._hop_off + self._unit * 2 - bh
            if origin is not None:
                sx = origin[0] - bw / 2.0 + random.uniform(-self._dp(8), self._dp(8))
                sy = origin[1] - bh
                drift = random.uniform(-self._dp(16), self._dp(16))
            elif kind == "zzz":
                sx = self._x + self._w * 0.72
                sy = head_y + self._unit * 2
                drift = self._dp(18)
            else:
                sx = self._x + self._w / 2.0 - bw / 2.0 + random.uniform(-self._w * 0.25, self._w * 0.25)
                sy = head_y
                drift = random.uniform(-self._dp(16), self._dp(16))
            view.setTranslationX(float(sx))
            view.setTranslationY(float(sy))
            view.setScaleX(0.5)
            view.setScaleY(0.5)
            view.setRotation(float(random.uniform(-15, 15)))

            anim = view.animate()
            anim.translationXBy(float(drift))
            anim.translationYBy(float(-self._dp(random.uniform(40, 58))))
            anim.scaleX(1.0).scaleY(1.0).alpha(0.0)
            anim.setDuration(1100 if kind != "zzz" else 1600)
            anim.withEndAction(R(lambda v=view: self._remove_particle(v)))
            anim.start()
        except Exception as e:
            log(f"[{__id__}] particle error: {e}")

    def _remove_particle(self, view):
        try:
            self._particles.remove(view)
        except ValueError:
            pass
        self._remove_view(view)

    def _say(self, category, force=False, name=None):
        if self._bubble is None or (not force and not self._cfg.get("phrases", True)):
            return
        phrases = PHRASES.get(category, [category])
        if name is None:
            phrases = [p for p in phrases if "{name}" not in p] or phrases
        options = [p for p in phrases if p != self._last_phrase] or phrases
        text = random.choice(options)
        if "{name}" in text:
            text = text.replace("{name}", name or "")
        self._last_phrase = text
        now = time.monotonic()
        try:
            bg = GradientDrawable()
            bg.setCornerRadius(float(AndroidUtilities.dp(12)))
            bg.setColor(Theme.getColor(Theme.key_windowBackgroundWhite))
            bg.setStroke(max(1, AndroidUtilities.dp(1)), Theme.getColor(Theme.key_divider))
            self._bubble.setBackground(bg)
            self._bubble.setTextColor(Theme.getColor(Theme.key_windowBackgroundWhiteBlackText))
            try:
                self._bubble.setElevation(float(AndroidUtilities.dp(2)))
            except Exception:
                pass
            self._bubble.animate().cancel()
            self._bubble.setText(text)
            self._bubble.setAlpha(0.0)
            self._bubble.setVisibility(View.VISIBLE)
            self._bubble_gen += 1
            self._bubble_visible = True
            self._bubble_fade_in = True
            self._bubble_until = now + 2.0 + len(text) * 0.05
        except Exception as e:
            log(f"[{__id__}] bubble error: {e}")

    def _hide_bubble(self):
        if self._bubble is None or not self._bubble_visible:
            return
        self._bubble_visible = False
        self._bubble_fade_in = False
        self._bubble_gen += 1
        gen = self._bubble_gen
        bubble = self._bubble

        def _gone():
            if gen == self._bubble_gen and bubble is not None:
                bubble.setVisibility(View.GONE)

        anim = bubble.animate().alpha(0.0).setDuration(180)
        anim.withEndAction(R(_gone))
        anim.start()

    def _update_bubble(self, now, width, min_y):
        if self._bubble is None or not self._bubble_visible:
            return
        if now >= self._bubble_until:
            self._hide_bubble()
            return
        bw = self._bubble.getWidth()
        bh = self._bubble.getHeight()
        if bw <= 0:
            return
        margin = self._dp(4)
        bx = clamp(self._x + self._w / 2.0 - bw / 2.0, margin, width - bw - margin)
        by = self._y + self._hop_off + self._unit - bh - margin
        if by < min_y:
            by = self._y + self._hop_off + self._h + margin
        self._bubble.setTranslationX(float(bx))
        self._bubble.setTranslationY(float(by))
        if self._bubble_fade_in:
            self._bubble_fade_in = False
            self._bubble.animate().alpha(1.0).setDuration(150).start()

    def _pick_target(self, top):
        decor = self._decor
        root = decor.findViewById(jclass("android.R$id").content)
        if root is None:
            return None
        width = decor.getWidth()
        height = decor.getHeight()
        dloc = jarray(jint)([0, 0])
        decor.getLocationInWindow(dloc)
        loc = jarray(jint)([0, 0])
        min_side = AndroidUtilities.dp(14)
        max_w = int(width * 0.6)
        max_h = AndroidUtilities.dp(140)

        candidates = []
        stack = [(root, 1.0, False)]
        visited = 0
        while stack and visited < EAT_SCAN_LIMIT:
            view, alpha, in_list = stack.pop()
            visited += 1
            if view.getVisibility() != View.VISIBLE:
                continue
            alpha *= float(view.getAlpha())
            if alpha < 0.6:
                continue
            w = view.getWidth()
            h = view.getHeight()
            if w <= 0 or h <= 0:
                continue
            count = view.getChildCount() if isinstance(view, ViewGroup) else 0
            if count > 0:
                is_list = isinstance(view, RecyclerView)
                for i in range(count):
                    child = view.getChildAt(i)
                    if child is not None:
                        stack.append((child, alpha, is_list))
                continue
            if in_list or w < min_side or h < min_side or w > max_w or h > max_h:
                continue
            if isinstance(view, (EditText, SurfaceView, TextureView)):
                continue
            if view.getScaleX() < 0.9 or view.getScaleY() < 0.9 or view.getRotation() != 0.0:
                continue
            view.getLocationInWindow(loc)
            x = loc[0] - dloc[0]
            y = loc[1] - dloc[1]
            if x < 0 or y < top or x + w > width or y + h > height:
                continue
            candidates.append((view, x, y, w, h))

        if not candidates:
            return None
        cx = self._x + self._w / 2.0
        cy = self._y + self._h / 2.0
        reach = self._dp(160)
        weights = [
            1.0 / (1.0 + (math.hypot(x + w / 2.0 - cx, y + h / 2.0 - cy) / reach) ** 2)
            for _, x, y, w, h in candidates
        ]
        return random.choices(candidates, weights)[0]

    def _snapshot(self, view, w, h):
        bmp = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888)
        canvas = Canvas(bmp)
        canvas.translate(float(-view.getScrollX()), float(-view.getScrollY()))
        view.draw(canvas)
        return bmp

    def _mouth_xy(self):
        return self._x + self._w / 2.0, self._y + self._hop_off + self._unit * 8.0

    def _make_copy(self, bmp, w, h):
        view = ImageView(self._pet.getContext())
        view.setImageBitmap(bmp)
        view.setScaleType(ImageView.ScaleType.FIT_XY)
        index = max(0, self._decor.indexOfChild(self._pet))
        self._decor.addView(view, index, FrameLayout.LayoutParams(w, h))
        self._flying.append(view)
        return view

    def _remove_flying(self, view):
        try:
            self._flying.remove(view)
        except ValueError:
            pass
        self._remove_view(view)

    def _try_eat(self, now, top):
        try:
            target = self._pick_target(top)
            if target is None:
                return False
            view, x, y, w, h = target
            bmp = self._snapshot(view, w, h)
            item = {"view": view, "alpha": float(view.getAlpha()), "bmp": bmp, "w": w, "h": h}
            view.setAlpha(0.0)
            self._eaten.append(item)

            copy = self._make_copy(bmp, w, h)
            copy.setTranslationX(float(x))
            copy.setTranslationY(float(y))
            mx, my = self._mouth_xy()
            self._look = 1 if x + w / 2.0 > mx else -1
            self._mouth_open_until = now + 0.7
            self._chew_until = 0.0
            self._next_action = now + 3.0
            self._next_eat = now + random.uniform(*EAT_COOLDOWN)

            anim = copy.animate()
            anim.translationX(float(mx - w / 2.0)).translationY(float(my - h / 2.0))
            anim.scaleX(0.05).scaleY(0.05)
            anim.rotation(float(random.choice((-1, 1)) * random.uniform(120.0, 300.0)))
            anim.setInterpolator(AccelerateInterpolator(1.6))
            anim.setDuration(560)
            anim.withEndAction(R(lambda v=copy: self._on_swallowed(v)))
            anim.start()
            return True
        except Exception as e:
            log(f"[{__id__}] eat error: {e}")
            return False

    def _on_swallowed(self, copy):
        self._remove_flying(copy)
        if self._pet is None:
            return
        now = time.monotonic()
        self._mouth_open_until = 0.0
        self._chew_until = now + 1.4
        self._squash_until = now + 0.12
        self._set_expr("happy", 1.6)
        self._haptic(3)
        try:
            self.set_setting("eaten_count", int(self.get_setting("eaten_count", 0) or 0) + 1)
        except Exception:
            pass
        if random.random() < 0.7:
            self._say("eat")
        if self._cfg.get("poop", True):
            try:
                meter = int(self.get_setting("poop_meter", 0) or 0) + 1
                if meter >= self._poop_need:
                    meter = 0
                    self._poop_need = random.randint(*POOP_EVERY)
                    self._poop_due = now + random.uniform(5.0, 10.0)
                self.set_setting("poop_meter", meter)
            except Exception as e:
                log(f"[{__id__}] poop meter error: {e}")

    def _do_poop(self, now, min_x, max_x):
        self._poop_due = 0.0
        if not self._cfg.get("poop", True) or self._decor is None or self._pet is None:
            return
        if len(self._poops) >= MAX_POOPS:
            return
        side = -self._look if self._look else random.choice((-1, 1))
        try:
            bmp = self._particle_bitmap("poop")
            bw, bh = bmp.getWidth(), bmp.getHeight()
            pad = AndroidUtilities.dp(6)
            view = ImageView(self._pet.getContext())
            view.setImageBitmap(bmp)
            view.setScaleType(ImageView.ScaleType.CENTER)
            view.setPadding(pad, pad, pad, pad)
            if self._poop_listener is None:
                self._poop_listener = PoopClickListener(self)
            view.setOnClickListener(self._poop_listener)
            index = max(0, self._decor.indexOfChild(self._pet))
            self._decor.addView(view, index, FrameLayout.LayoutParams(bw + pad * 2, bh + pad * 2))
            feet = self._y + self._unit * 12
            px = self._x + self._w / 2.0 + side * self._w * 0.42 - bw / 2.0 - pad
            px = clamp(px, 0.0, float(self._decor.getWidth() - bw - pad * 2))
            view.setTranslationX(float(px))
            view.setTranslationY(float(feet - bh - pad))
            view.setScaleX(0.1)
            view.setScaleY(0.1)
            anim = view.animate().scaleX(1.0).scaleY(1.0)
            anim.setInterpolator(OvershootInterpolator(2.0))
            anim.setDuration(320)
            anim.start()
            self._poops.append(view)
        except Exception as e:
            log(f"[{__id__}] poop error: {e}")
            return
        self._squash_until = now + 0.2
        self._set_expr("happy", 1.8)
        self._haptic(3)
        self._say("poop")
        self._look = -side
        target = clamp(self._x - side * random.uniform(self._dp(40), self._dp(80)), min_x, max_x)
        if abs(target - self._x) > self._dp(12):
            self._walk_target = target
            self._state = "walk"
        self._next_action = now + random.uniform(3.0, 6.0)

    def _clean_poop(self, view):
        target = None
        for poop in self._poops:
            if poop.equals(view):
                target = poop
                break
        if target is None:
            return
        self._poops.remove(target)
        target.setClickable(False)
        cx = float(target.getTranslationX()) + target.getWidth() / 2.0
        cy = float(target.getTranslationY()) + target.getHeight() / 2.0
        for _ in range(3):
            self._spawn_particle("spark", origin=(cx, cy))
        anim = target.animate().scaleX(0.0).scaleY(0.0).alpha(0.0).rotation(90.0).setDuration(260)
        anim.withEndAction(R(lambda v=target: self._remove_view(v)))
        anim.start()
        try:
            self.set_setting("poops_cleaned", int(self.get_setting("poops_cleaned", 0) or 0) + 1)
        except Exception:
            pass
        self._haptic(3)
        if self._pet is not None and self._state in ("idle", "walk"):
            self._set_expr("happy", 1.5)
            if random.random() < 0.7:
                self._say("clean")
            self._schedule(0)

    def _remove_poops(self):
        poops = self._poops
        self._poops = []
        for view in poops:
            self._remove_view(view)

    def _prune_eaten(self):
        kept = []
        for item in self._eaten:
            try:
                if float(item["view"].getAlpha()) < 0.01:
                    kept.append(item)
            except Exception:
                pass
        self._eaten = kept

    def _restore_item(self, item):
        try:
            view = item["view"]
            if float(view.getAlpha()) < 0.01:
                view.setAlpha(float(item["alpha"]))
        except Exception:
            pass

    def _restore_all(self):
        items = self._eaten + self._spitting
        self._eaten = []
        self._spitting = []
        for view in list(self._flying):
            self._remove_view(view)
        self._flying = []
        for item in items:
            self._restore_item(item)

    def _give_back(self):
        if self._eaten and self._pet is not None and self._decor is not None:
            self._spit_all(time.monotonic())
        else:
            self._restore_all()

    def _spit_all(self, now):
        items = self._eaten
        self._eaten = []
        if not items:
            return
        self._chew_until = 0.0
        self._mouth_open_until = now + 0.35 + 0.08 * len(items)
        self._set_expr("surprised", 0.9)
        self._squash_until = now + 0.12
        self._hop(self._dp(300))
        self._haptic(1)
        self._say("spit")
        self._next_eat = max(self._next_eat, now + random.uniform(*EAT_COOLDOWN))
        mx, my = self._mouth_xy()
        for i, item in enumerate(items):
            self._spit_one(item, mx, my, i * 80)
        self._schedule(0)

    def _spit_one(self, item, mx, my, delay):
        view = item["view"]
        w, h = item["w"], item["h"]
        try:
            if self._decor is None or not view.isAttachedToWindow():
                self._restore_item(item)
                return
            dloc = jarray(jint)([0, 0])
            loc = jarray(jint)([0, 0])
            self._decor.getLocationInWindow(dloc)
            view.getLocationInWindow(loc)
            x = loc[0] - dloc[0]
            y = loc[1] - dloc[1]
            width = self._decor.getWidth()
            height = self._decor.getHeight()
            if x + w <= 0 or y + h <= 0 or x >= width or y >= height:
                self._restore_item(item)
                return

            self._spitting.append(item)
            copy = self._make_copy(item["bmp"], w, h)
            copy.setTranslationX(float(mx - w / 2.0))
            copy.setTranslationY(float(my - h / 2.0))
            copy.setScaleX(0.01)
            copy.setScaleY(0.01)
            copy.setRotation(float(random.choice((-1, 1)) * random.uniform(120.0, 300.0)))

            def _done(v=copy, it=item):
                self._remove_flying(v)
                try:
                    self._spitting.remove(it)
                except ValueError:
                    return
                self._restore_item(it)

            anim = copy.animate()
            anim.translationX(float(x)).translationY(float(y))
            anim.scaleX(1.0).scaleY(1.0).rotation(0.0)
            anim.setInterpolator(OvershootInterpolator(1.2))
            anim.setStartDelay(int(delay))
            anim.setDuration(460)
            anim.withEndAction(R(_done))
            anim.start()
        except Exception as e:
            log(f"[{__id__}] spit error: {e}")
            try:
                self._spitting.remove(item)
            except ValueError:
                pass
            self._restore_item(item)
