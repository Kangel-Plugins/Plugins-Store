import json
import re

from base_plugin import BasePlugin, HookResult, HookStrategy
from android_utils import run_on_ui_thread
from client_utils import get_last_fragment
from ui.settings import Header, Input, Switch, Text, Divider
from ui.bulletin import BulletinHelper
from ui.alert import AlertDialogBuilder

__id__ = "bot_tags"
__name__ = "Bot Tags"
__icon__ = "excess_icon/4"
__description__ = (
    "Добавляет тег к каждому сообщению в выбранных чатах."
    " Быстрая настройка командой .tag прямо в чате.\n"
    "Полезно в ботах общения/поддержки."
)
__author__ = "@cobra_S0FT | @excess_plugins"
__version__ = "1.0.0"
__min_version__ = "12.5.1"

COMMANDS = (".tag", ".тег")
OFF_WORDS = ("off", "-", "выкл", "удалить", "del")


def normalize_tag(raw: str) -> str:
    """'моя метка' -> '#моя_метка', '##a' -> '#a', '' -> ''."""
    raw = re.sub(r"\s+", "_", (raw or "").strip()).lstrip("#")
    return "#" + raw if raw else ""


def _u16len(s: str) -> int:
    return len(s.encode("utf-16-le", "surrogatepass")) // 2


class BotTagsPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()

    def _tags(self) -> dict:
        try:
            data = json.loads(self.get_setting("tags", "{}"))
            return data if isinstance(data, dict) else {}
        except Exception:
            return {}

    def _save(self, data: dict, reload: bool = False):
        """reload=True — перерисовать открытый экран настроек (обновить список)."""
        value = json.dumps(data, ensure_ascii=False)
        if reload:
            try:
                self.set_setting("tags", value, reload_settings=True)
                return
            except TypeError:  
                pass
        self.set_setting("tags", value)

    def _toast(self, text: str):
        run_on_ui_thread(lambda: BulletinHelper.show_info(text))

    @staticmethod
    def _shift_entities(params, shift: int):
        try:
            entities = getattr(params, "entities", None)
            if entities is None or shift == 0:
                return
            for i in range(entities.size()):
                entities.get(i).offset += shift
        except Exception:
            pass

    @staticmethod
    def _clamp_entities(params, limit: int):
        """После rstrip() сущность не должна выходить за длину текста."""
        try:
            entities = getattr(params, "entities", None)
            if entities is None:
                return
            for i in range(entities.size()):
                e = entities.get(i)
                if e.offset + e.length > limit:
                    e.length = max(0, limit - e.offset)
        except Exception:
            pass

    def on_send_message_hook(self, account, params) -> HookResult:
        try:
            msg = getattr(params, "message", None)
            if not isinstance(msg, str) or not msg.strip():
                return HookResult()
            if not self.get_setting("enabled", True):
                return HookResult()

            peer = str(int(params.peer))
            stripped = msg.strip()
            low = stripped.lower()

            for cmd in COMMANDS:
                if low == cmd or (low.startswith(cmd) and low[len(cmd)].isspace()):
                    self._handle_command(peer, stripped[len(cmd):].strip())
                    return HookResult(strategy=HookStrategy.CANCEL)

            tag = self._tags().get(peer)
            if not tag:
                return HookResult()

            if self.get_setting("skip_commands", True) and stripped.startswith("/"):
                return HookResult()

            sep = "\n" if self.get_setting("newline", False) else " "

            if self.get_setting("at_start", False):
                body = msg.lstrip()
                if body.split(None, 1)[0].lower() == tag.lower():
                    return HookResult()
                prefix = tag + sep
                params.message = prefix + body
                shift = _u16len(prefix) - (_u16len(msg) - _u16len(body))
                self._shift_entities(params, shift)
            else:
                body = msg.rstrip()
                if body.split()[-1].lower() == tag.lower():
                    return HookResult()
                params.message = body + sep + tag
                self._clamp_entities(params, _u16len(body))
            return HookResult(strategy=HookStrategy.MODIFY, params=params)
        except Exception:
            return HookResult()

    def _handle_command(self, peer: str, arg: str):
        tags = self._tags()
        if not arg:
            cur = tags.get(peer)
            self._toast(f"Тег этого чата: {cur}" if cur else "Для этого чата тег не задан. Пример: .tag кобра")
            return
        if arg.lower() in OFF_WORDS:
            if tags.pop(peer, None):
                self._save(tags)
                self._toast("Тег для этого чата удалён")
            else:
                self._toast("Тег для этого чата не был задан")
            return
        tag = normalize_tag(arg)
        if not tag:
            self._toast("Тег не может быть пустым")
            return
        tags[peer] = tag
        self._save(tags)
        self._toast(f"Готово: {peer} – {tag}")
    def _add_from_form(self, view=None):
        chat_id = str(self.get_setting("form_id", "") or "").strip()
        tag = normalize_tag(str(self.get_setting("form_tag", "") or ""))
        if not chat_id.lstrip("-").isdigit() or not tag:
            self._toast("Введите числовой ID и тег")
            return
        key = str(int(chat_id))
        tags = self._tags()
        existed = key in tags
        tags[key] = tag
        self.set_setting("form_id", "")
        self.set_setting("form_tag", "")
        self._save(tags, reload=True)
        self._toast(f"{'Обновлено' if existed else 'Добавлено'}: {key} – {tag}")

    def _remove(self, chat_id: str):
        tags = self._tags()
        if tags.pop(chat_id, None):
            self._save(tags, reload=True)
            self._toast("Тег удалён")

    def _update_tag(self, chat_id: str, new_tag: str):
        tag = normalize_tag(new_tag)
        if not tag:
            self._toast("Тег не может быть пустым")
            return
        tags = self._tags()
        if chat_id not in tags:
            return
        if tags[chat_id] == tag:
            return
        tags[chat_id] = tag
        self._save(tags, reload=True)
        self._toast(f"Тег изменён: {chat_id} – {tag}")

    def _open_tag_menu(self, chat_id: str, tag: str):
        try:
            activity = get_last_fragment().getParentActivity()
            b = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_MESSAGE)
            b.set_title("Тег чата")
            b.set_message(f"ID: {chat_id}\nТег: {tag}")
            b.set_positive_button("Изменить", lambda d, w: (d.dismiss(), self._open_edit(chat_id, tag)))
            b.set_neutral_button("Удалить", lambda d, w: (d.dismiss(), self._remove(chat_id)))
            b.set_negative_button("Отмена", lambda d, w: d.dismiss())

            def show():
                b.show()
                try:
                    b.make_button_red(AlertDialogBuilder.BUTTON_NEUTRAL)
                except Exception:
                    pass

            run_on_ui_thread(show)
        except Exception:
            self._toast("Не удалось открыть диалог")

    def _open_edit(self, chat_id: str, tag: str):
        try:
            from android.widget import EditText
            from android.text import InputType
            from org.telegram.messenger import AndroidUtilities

            activity = get_last_fragment().getParentActivity()
            et = EditText(activity)
            et.setText(tag)
            et.setHint("Тег")
            et.setSingleLine(True)
            et.setInputType(InputType.TYPE_CLASS_TEXT)
            try:
                from org.telegram.ui.ActionBar import Theme
                et.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
                et.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
            except Exception:
                pass
            pad = AndroidUtilities.dp(24)
            et.setPadding(pad, AndroidUtilities.dp(12), pad, AndroidUtilities.dp(12))
            et.setSelection(len(tag))

            b = AlertDialogBuilder(activity, AlertDialogBuilder.ALERT_TYPE_MESSAGE)
            b.set_title("Изменить тег")
            b.set_message(f"ID: {chat_id}")
            b.set_view(et)
            b.set_positive_button("Сохранить",
                                  lambda d, w: (d.dismiss(), self._update_tag(chat_id, str(et.getText().toString()))))
            b.set_negative_button("Отмена", lambda d, w: d.dismiss())

            def focus():
                try:
                    et.requestFocus()
                    AndroidUtilities.showKeyboard(et)
                except Exception:
                    pass

            def show():
                b.show()
                try:
                    run_on_ui_thread(focus, 200)
                except Exception:
                    pass

            run_on_ui_thread(show)
        except Exception:
            self._toast("Не удалось открыть редактор. Используйте команду .tag в чате")

    def create_settings(self):
        items = [
            Header(text="Основное"),
            Switch(key="enabled", text="Плагин включён", default=True),
            Switch(key="at_start", text="Тег в начале сообщения", default=False),
            Switch(key="newline", text="Тег на отдельной строке", default=False),
            Switch(key="skip_commands", text="Не добавлять к командам (/start…)", default=True),
            Divider(text="Быстрый способ: напишите в чате .tag name – тег привяжется к этому чату.\n.tag off – убрать тег\n.tag – показать текущий."),
            Header(text="Добавить вручную"),
            Input(key="form_id", text="ID", default="", subtext="Например: 123456789"),
            Input(key="form_tag", text="Тег", default="", subtext="Например: name или #name"),
            Text(text="Добавить", accent=True, on_click=self._add_from_form),
            Header(text="Чаты с тегами (нажмите, чтобы изменить или удалить)"),
        ]
        tags = self._tags()
        if not tags:
            items.append(Divider(text="Пока пусто"))
        for cid, tag in tags.items():
            items.append(Text(text=f"{cid} – {tag}",
                              on_click=lambda v, c=cid, t=tag: self._open_tag_menu(c, t)))
        return items
