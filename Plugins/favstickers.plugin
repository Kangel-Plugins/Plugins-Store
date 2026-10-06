import datetime as dt
import json
import os
import uuid

from android_utils import log, run_on_ui_thread
from base_plugin import BasePlugin, MethodHook
from client_utils import get_last_fragment, send_document
from file_utils import get_cache_dir, read_file, write_file
from hook_utils import find_class
from java import jclass, jint
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Text

__id__ = "favstickers"
__name__ = "Unlim favorite stickers"
__description__ = "Remove limits on adding stickers to favorites, with backup"
__author__ = "@DaShMore & @teleplugit"
__version__ = "2.2.1"
__icon__ = "plugins_covers/0"
__min_version__ = "11.12.0"


class Jclass:
    def __getattr__(self, name: str):
        return jclass(f"java.lang.{name.title()}")


J = Jclass()
TLRPC = jclass("org.telegram.tgnet.TLRPC")
ArrayList = jclass("java.util.ArrayList")

BACKUP_VERSION = 1
BACKUP_SUFFIX = ".stickers"
BACKUP_CAPTION = "Бекап избранных стикеров"
REQUIRED_STICKER_FIELDS = ("id", "access_hash", "dc_id", "mime_type")


class BackupError(Exception):
    """Причина отказа импорта, пригодная для показа пользователю"""


def serialize_sticker(sticker) -> dict[str, str | int]:
    # Иногда обертка стикера содержит .document
    if hasattr(sticker, "document") and sticker.document is not None:
        sticker = sticker.document

    data = {
        "id": int(sticker.id),
        "access_hash": int(sticker.access_hash),
        "dc_id": int(sticker.dc_id),
        "mime_type": str(sticker.mime_type),
    }

    if hasattr(sticker, "attributes") and sticker.attributes is not None:
        attr_list = sticker.attributes
        for i in range(attr_list.size()):
            att = attr_list.get(i)

            if hasattr(att, "stickerset") and att.stickerset is not None:
                # У TL_inputStickerSetEmpty и TL_inputStickerSetShortName
                # поля id нет вовсе - такой стикер сохраняем без набора
                set_id = getattr(att.stickerset, "id", None)
                if set_id is not None:
                    data["sticker_set_id"] = set_id
    return data


def deserialize_sticker(data: dict):
    """Десериализует словарь в телеграм стикер"""
    doc = TLRPC.TL_document()

    doc.id = data["id"]
    doc.access_hash = data["access_hash"]
    doc.dc_id = data["dc_id"]
    doc.mime_type = data["mime_type"]
    # FIXME
    doc.attributes = ArrayList()

    a = TLRPC.TL_documentAttributeSticker()
    ss = TLRPC.TL_inputStickerSetID()
    # serialize_sticker кладет sticker_set_id только если у стикера нашелся
    # атрибут stickerset - у стикера без набора ключа не будет
    ss.id = data.get("sticker_set_id", 0)
    a.stickerset = ss
    doc.attributes.add(a)

    return doc


def backup_filename(scope: str) -> str:
    """Имя файла бекапа: favorites-2026-08-22.stickers"""
    suffix = "-all" if scope == "all" else ""
    return f"favorites{suffix}-{dt.date.today().isoformat()}{BACKUP_SUFFIX}"


def detect_scope(data: dict) -> str:
    """Слепок базы отличается от списка одного аккаунта ключом accounts

    Условие обязано совпадать с первой веткой parse_backup: иначе файл
    пройдёт разбор как один формат, а классифицируется как другой.
    """
    return (
        "all"
        if isinstance(data.get("accounts"), dict)
        and isinstance(data.get("stickers"), dict)
        else "account"
    )


def parse_backup(text: str) -> dict:
    """Разбор файла бекапа. На любой брак - BackupError с причиной"""
    try:
        data = json.loads(text)
    except (ValueError, RecursionError):
        raise BackupError("Файл не похож на бекап стикеров")
    if not isinstance(data, dict):
        raise BackupError("Файл не похож на бекап стикеров")
    version = data.get("version")
    if not isinstance(version, int):
        raise BackupError("Файл не похож на бекап стикеров")
    if version > BACKUP_VERSION:
        raise BackupError("Файл сделан более новой версией плагина")
    if version != BACKUP_VERSION:
        raise BackupError("Файл сделан несовместимой версией плагина")
    if isinstance(data.get("accounts"), dict) and isinstance(
        data.get("stickers"), dict
    ):
        return data
    if isinstance(data.get("stickers"), list):
        return data
    raise BackupError("Файл не похож на бекап стикеров")


class StickersDB:
    """
    Класс для работы с базой данных стикеров

    {
        "accounts": {
            <account_id>: [
                <sticker_id>,
                <sticker_id>,
            ]
        },
        "stickers": {
            <sticker_id>: {
                "id": <sticker_id>,
                "access_hash": <access_hash>,
                "dc_id": <dc_id>,
                "mime_type": <mime_type>,
                "sticker_set_id": <sticker_set_id>
            }
        }
    }
    """

    # JSON умеет только строковые ключи, поэтому id аккаунта приводится к
    # str на входе в каждый публичный метод - иначе вызов с int тихо
    # промахнется мимо своих же записей и вернет пустой список
    def __init__(self, db_path: str):
        self.__db_path = db_path
        self.__cache = {}
        self.__load_db()

    @staticmethod
    def __has_expected_shape(stickers) -> bool:
        return (
            isinstance(stickers, dict)
            and isinstance(stickers.get("accounts"), dict)
            and isinstance(stickers.get("stickers"), dict)
        )

    def __load_db(self):
        """Чтение всех стикеров из базы в self.stickers"""
        self.__cache.clear()
        self.__stickers = {"accounts": {}, "stickers": {}}
        if not os.path.exists(self.__db_path):
            return
        try:
            with open(self.__db_path, encoding="utf-8") as f:
                loaded = json.load(f)
        except (OSError, ValueError) as e:
            # Битый или недоступный файл - начинаем с пустой базы,
            # иначе плагин не загрузится вообще
            log(f"[favstickers] Не удалось прочитать базу стикеров: {e}")
            return
        if not self.__has_expected_shape(loaded):
            # Валидный JSON не той формы разбор проходит, а падает потом на
            # каждом обращении - уже внутри хука, где исключение никто не увидит
            log("[favstickers] База стикеров не той формы, начинаем с пустой")
            return
        self.__stickers = loaded

    def __save_db(self):
        """Сохранение всех стикеров из self.stickers в базу"""
        tmp_path = self.__db_path + ".tmp"
        try:
            with open(tmp_path, "w", encoding="utf-8") as f:
                json.dump(self.__stickers, f, ensure_ascii=False)
            # Подмена файла целиком: прерывание записи не оставит битую базу
            os.replace(tmp_path, self.__db_path)
        except OSError as e:
            # Молча продолжать нельзя: пользователь будет добавлять стикеры
            # весь сеанс и потеряет их при перезапуске
            log(f"[favstickers] Не удалось сохранить базу стикеров: {e}")
            BulletinHelper.show_error("Failed to save favorite stickers")

    def get_all_stickers(self, account) -> list:
        """Получение всех стикеров в виде объектов TLRPC$TL_document

        Результат кешируется: метод зовется на каждой перерисовке панели и
        на каждом нажатии клавиши в подсказках по эмодзи, а разбор одной
        записи - это четыре конструктора и семь присваиваний через JNI.
        """
        account = str(account)
        cached = self.__cache.get(account)
        if cached is not None:
            return cached
        stickers = []
        for sticker_id in self.__ordered_ids(account):
            data = self.__stickers["stickers"].get(sticker_id)
            if data is None:
                continue
            try:
                stickers.append(deserialize_sticker(data))
            except Exception as e:
                # Одна битая запись не должна уносить весь список: исключение
                # отсюда уходит в хук, где его глотает мост, и пользователь
                # видит ванильное избранное вместо своего
                log(f"[favstickers] Пропущен стикер {sticker_id}: {e}")
        self.__cache[account] = stickers
        return stickers

    def add_sticker(self, sticker, account):
        """Сериализация и добавление стикера в базу без дубликатов"""
        account = str(account)
        serialized_sticker = serialize_sticker(sticker)
        sticker_id = str(serialized_sticker["id"])
        changed = False
        if sticker_id not in self.__stickers["stickers"]:
            self.__stickers["stickers"][sticker_id] = serialized_sticker
            changed = True
        if account not in self.__stickers["accounts"]:
            self.__stickers["accounts"][account] = []
        if sticker_id not in self.__stickers["accounts"][account]:
            self.__stickers["accounts"][account].append(sticker_id)
            changed = True
        if changed:
            self.__cache.clear()
            self.__save_db()

    def remove_sticker(self, sticker, account):
        """Удаление стикера из базы и self.stickers."""
        account = str(account)
        serialized_sticker = serialize_sticker(sticker)
        sticker_id = str(serialized_sticker["id"])
        changed = False
        if sticker_id in self.__stickers["accounts"].get(account, []):
            self.__stickers["accounts"][account].remove(sticker_id)
            changed = True
        # Если ни у кого стикер не сохранен - удаляем из общего списка
        if sticker_id in self.__stickers["stickers"] and not any(
            sticker_id in i for i in self.__stickers["accounts"].values()
        ):
            del self.__stickers["stickers"][sticker_id]
            changed = True
        if changed:
            self.__cache.clear()
            self.__save_db()

    def is_sticker_favorite(self, sticker, account):
        """Проверка, есть ли стикер в избранных"""
        serialized_sticker = serialize_sticker(sticker)
        return str(serialized_sticker["id"]) in self.__stickers["accounts"].get(
            str(account), []
        )

    def __ordered_ids(self, account: str) -> list:
        """Id стикеров аккаунта в порядке панели, без висячих ссылок

        Правило одно для чтения, экспорта и счётчиков: иначе на кнопке
        экспорта окажется одно число, а в файле другое.
        """
        return [
            sticker_id
            for sticker_id in reversed(self.__stickers["accounts"].get(account, []))
            if sticker_id in self.__stickers["stickers"]
        ]

    def count_stickers(self, account) -> int:
        """Сколько стикеров у аккаунта попадёт в панель и в экспорт"""
        return len(self.__ordered_ids(str(account)))

    def count_all(self) -> tuple[int, int]:
        """(всего стикеров, аккаунтов с непустым избранным)

        Аккаунт, у которого удалили последний стикер, остаётся в базе с
        пустым списком - в счётчик для экрана настроек он попадать не должен.
        """
        accounts = sum(
            1 for account in self.__stickers["accounts"] if self.__ordered_ids(account)
        )
        return len(self.__stickers["stickers"]), accounts

    def export_account(self, account) -> dict:
        """Плоский список в порядке панели, без упоминания аккаунтов"""
        account = str(account)
        return {
            "version": BACKUP_VERSION,
            "stickers": [
                self.__stickers["stickers"][sticker_id]
                for sticker_id in self.__ordered_ids(account)
            ],
        }

    def export_all(self) -> dict:
        """Слепок базы: восстанавливает все аккаунты по их id"""
        return {
            "version": BACKUP_VERSION,
            "accounts": {
                account: list(ids)
                for account, ids in self.__stickers["accounts"].items()
            },
            "stickers": dict(self.__stickers["stickers"]),
        }

    @staticmethod
    def __is_valid_record(record) -> bool:
        return isinstance(record, dict) and all(
            key in record for key in REQUIRED_STICKER_FIELDS
        )

    def __drop_orphans(self):
        """Записи, на которые не ссылается ни один аккаунт, не нужны"""
        used = {
            sticker_id
            for ids in self.__stickers["accounts"].values()
            for sticker_id in ids
        }
        for sticker_id in list(self.__stickers["stickers"]):
            if sticker_id not in used:
                del self.__stickers["stickers"][sticker_id]

    def import_account(self, data: dict, account, replace: bool = False) -> tuple:
        """Импорт плоского списка в аккаунт

        ImportResult из спеки - это кортеж (применено, пропущено):
        заводить датакласс ради двух чисел незачем.
        """
        records = data.get("stickers", [])
        # Проверяем форму до мутации: режим замены стирает список аккаунта,
        # и на чужом формате это была бы потеря данных с отчётом (0, N),
        # неотличимым от "файл был пустой"
        if not isinstance(records, list):
            raise BackupError("Файл не похож на бекап стикеров")
        account = str(account)
        applied = skipped = 0
        if replace:
            self.__stickers["accounts"][account] = []
        ids = self.__stickers["accounts"].setdefault(account, [])
        # В файле новые сверху, база хранит наоборот
        for record in reversed(records):
            try:
                if not self.__is_valid_record(record):
                    # skipped считаем только после успешного лога: если сам
                    # str(record) уронит эту строку, счётчик поправит внешний
                    # except - а не оба обработчика на одной записи разом
                    log(f"[favstickers] Пропущена запись бекапа: {str(record)[:80]}")
                    skipped += 1
                    continue
                sticker_id = str(record["id"])
                self.__stickers["stickers"][sticker_id] = record
                if sticker_id not in ids:
                    ids.append(sticker_id)
                applied += 1
            except Exception as e:
                # Данные пришли из файла: и сама запись, и её строковое
                # представление могут оказаться чем угодно. Одна такая
                # не должна оставлять базу изменённой на середине импорта
                skipped += 1
                log(f"[favstickers] Запись бекапа не разобрана: {type(e).__name__}")
        self.__drop_orphans()
        self.__cache.clear()
        self.__save_db()
        return applied, skipped

    def import_all(self, data: dict, replace: bool = False) -> tuple:
        """Импорт слепка: каждый аккаунт восстанавливается по своему id"""
        accounts = data.get("accounts", {})
        records = data.get("stickers", {})
        # Форму слепка проверяем до первой мутации: import_account сохраняет
        # базу после каждого аккаунта, и падение на середине оставило бы
        # восстановление наполовину сделанным, да ещё и на диске
        if not isinstance(accounts, dict) or not isinstance(records, dict):
            raise BackupError("Файл не похож на бекап стикеров")
        applied = skipped = 0
        for account, ids in accounts.items():
            if not isinstance(ids, list):
                # Строка или словарь развернулись бы через reversed() молча,
                # и аккаунт восстановился бы из мусора вместо стикеров
                log(
                    f"[favstickers] Пропущен аккаунт {account}: "
                    "список стикеров испорчен"
                )
                continue
            # import_account ждет порядок панели, в слепке порядок базы
            payload = {
                "stickers": [records[i] for i in reversed(ids) if i in records]
            }
            account_applied, account_skipped = self.import_account(
                payload, account, replace
            )
            applied += account_applied
            skipped += account_skipped
        return applied, skipped


class ChangeFavoriteStickerHook(MethodHook):
    def __init__(self, on_add_favorite, on_remove_favorite, refresh_panel, get_account_id):
        self.__on_add_favorite = on_add_favorite
        self.__on_remove_favorite = on_remove_favorite
        self.__refresh_panel = refresh_panel
        self.__get_account_id = get_account_id

    def before_hooked_method(self, param):
        # Проверяем, что выбран пункт добавления / удаления из избранное (TYPE_FAVE = 2)
        if param.args[0] != 2:
            return
        try:
            self.__change_favorite(param)
        except Exception as e:
            # Мост глотает исключение молча: без setResult отработает оригинал
            # и уведет стикер в ванильную пятерку. Скажем хотя бы об этом
            log(f"[favstickers] Не удалось изменить избранное: {e!r}")
            BulletinHelper.show_error("Failed to change favorites")

    def __change_favorite(self, param):
        sticker = param.args[2]
        account_id = self.__get_account_id()
        # И что стикер не находится в избранном (not inFavs)
        if not param.args[4]:
            self.__on_add_favorite(sticker, account_id)
            BulletinHelper.show_success("Sticker added to favorites")
        else:
            self.__on_remove_favorite(sticker, account_id)
            BulletinHelper.show_error("Sticker removed from favorites")
        param.setResult(None)
        # Оригинал отменен, а перерисовку панели делал именно он
        self.__refresh_panel()


class GetFavoriteStickersHook(MethodHook):
    def __init__(self, get_favorite_stickers, get_account_id, import_stickers):
        self.__get_favorite_stickers = get_favorite_stickers
        self.__get_account_id = get_account_id
        self.__import_stickers = import_stickers

    def after_hooked_method(self, param):
        # Хук висит на всех перегрузках getRecentStickers / getRecentStickersNoCopy,
        # а тип списка везде идет первым аргументом (TYPE_FAVE = 2).
        # Недавние, маски и премиум-стикеры не наши - их подменять нельзя.
        if param.args[0] != 2:
            return
        try:
            self.__replace_with_saved(param)
        except Exception as e:
            # Без setResult останется ванильный список - урезанный, но живой
            log(f"[favstickers] Не удалось прочитать избранное: {e!r}")

    def __replace_with_saved(self, param):
        account = self.__get_account_id()
        favorite_stickers = self.__get_favorite_stickers(account)
        if not favorite_stickers:
            # Пустая база - это еще не заполненная база. Засеваем ее из
            # ванильного списка и пропускаем его дальше: на загрузке плагина
            # он мог быть еще не подтянут с сервера, а здесь он уже готов
            self.__import_stickers(param.getResult(), account)
            return
        new_list = jclass("java.util.ArrayList")()
        for sticker in favorite_stickers:
            new_list.add(sticker)
        param.setResult(new_list)


class IsStickerInFavoritesHook(MethodHook):
    def __init__(self, is_favorite_sticker, get_account_id):
        self.__is_favorite_sticker = is_favorite_sticker
        self.__get_account_id = get_account_id

    def before_hooked_method(self, param):
        try:
            favorite = self.__is_favorite_sticker(param.args[0], self.__get_account_id())
        except Exception as e:
            # Без setResult ответит оригинал: соврет про наши стикеры,
            # но меню стикера откроется
            log(f"[favstickers] Не удалось проверить избранное: {e!r}")
            return
        param.setResult(favorite)


class ImportBackupHook(MethodHook):
    """Перехват тапа по файлу .stickers в чате

    Висит на всех перегрузках AndroidUtilities.openForView, то есть на любом
    открываемом файле. Штатное открытие гасится строго после того, как имя
    опознано - иначе плагин сломает открытие чужих вложений.
    """

    def __init__(self, on_backup_tap):
        self.__on_backup_tap = on_backup_tap

    def before_hooked_method(self, param):
        try:
            if not param.args:
                return
            path = self.__backup_path(param.args[0])
        except Exception as e:
            # Фаза опознания: файл ещё может быть чужим, поэтому только лог.
            # Плашка тут напугала бы человека, открывающего свой pdf
            log(f"[favstickers] Не удалось опознать файл: {e!r}")
            return
        if path is None:
            return
        param.setResult(False)
        try:
            self.__on_backup_tap(path)
        except Exception as e:
            # Файл наш и открытие уже отменено: без плашки тап останется
            # мёртвым, а причина - невидимой
            log(f"[favstickers] Ошибка обработки бекапа: {e!r}")
            BulletinHelper.show_error("Не удалось открыть бекап", get_last_fragment())

    @staticmethod
    def __backup_path(arg):
        """Путь к нашему файлу, иначе None"""
        if arg is None:
            return None
        if hasattr(arg, "getAbsolutePath"):
            path = str(arg.getAbsolutePath())
            return path if path.lower().endswith(BACKUP_SUFFIX) else None
        name = str(arg.getDocumentName()) if hasattr(arg, "getDocumentName") else ""
        if not name.lower().endswith(BACKUP_SUFFIX):
            return None
        # Файл ещё не скачан: не перехватываем намеренно. Перехват отменяет
        # оригинал, а именно он, судя по всему, и запускает докачку - иначе
        # тап показывал бы "дождитесь загрузки" и сам же её и предотвращал.
        # Случай "путь есть, а файла на диске нет" ловится дальше, при чтении
        attach_path = getattr(getattr(arg, "messageOwner", None), "attachPath", None)
        return str(attach_path) if attach_path else None


class MyPlugin(BasePlugin):
    __DB = None

    def __get_context(self):
        """
        Возвращает контекст приложения
        """
        current_app = jclass("android.app.ActivityThread").currentApplication()
        if not current_app:
            raise RuntimeError("app not find")
        return current_app

    @property
    def db(self):
        if self.__DB is None:
            self.__DB = StickersDB(
                os.path.join(str(self.__get_context().getFilesDir()), "stickers.json")
            )
        return self.__DB

    @staticmethod
    def __get_current_account() -> int:
        UserConfig = find_class("org.telegram.messenger.UserConfig")
        # currentAccount — это индекс аккаунта (обычно 0)
        return UserConfig.selectedAccount

    @staticmethod
    def __get_current_account_id() -> str:
        UserConfig = find_class("org.telegram.messenger.UserConfig")
        user_id = UserConfig.getInstance(UserConfig.selectedAccount).clientUserId
        return str(user_id)

    @staticmethod
    def __notify_favorites_changed():
        """Просит панель стикеров перечитать избранное

        В стоке это уведомление посылал addRecentSticker, а мы его
        отменяем - без замены список не перерисуется, пока панель не
        закрыть и открыть заново.

        jint обязателен: postNotificationName принимает Object..., мост
        боксирует питоновский int в Long, а слушатель делает
        (Integer) args[1] и падает с ClassCastException внутри UI.
        """
        TYPE_FAVE = 2
        NotificationCenter = find_class("org.telegram.messenger.NotificationCenter")
        center = NotificationCenter.getInstance(MyPlugin.__get_current_account())
        center.postNotificationName(
            NotificationCenter.recentDocumentsDidLoad, False, jint(TYPE_FAVE)
        )

    def __import_vanilla_favorites(self, stickers, account: str):
        """Первичное заполнение базы ванильным избранным

        Зовется из хука на чтение списка, а не при загрузке плагина: там
        recentStickers еще мог не подтянуться с сервера, и импорт молча
        забирал пустоту или огрызок.
        """
        if stickers is None:
            return
        # С конца, чтобы порядок в базе совпал с порядком в панели
        for i in range(stickers.size() - 1, -1, -1):
            self.db.add_sticker(stickers.get(i), account)

    def __hook_one(self, method, hook, name: str, unhooks: list):
        """Перехват конкретной сигнатуры.

        hook_method умеет вернуть None вместо исключения - для нас это
        такой же отказ, молчать о нем нельзя.
        """
        unhook = self.hook_method(method, hook)
        if unhook is None:
            raise RuntimeError(f"не удалось перехватить {name}")
        unhooks.append(unhook)

    @staticmethod
    def __load_class(name: str, loader):
        """Настоящий java.lang.Class по имени, иначе None

        getDeclaredMethods есть только у Class, а find_class отдаёт обёртку
        jclass - на ней зовут статику вроде UserConfig.getInstance. Загрузчик
        берём явный: у вызова из Python своего загрузчика нет.
        """
        try:
            return loader.loadClass(name)
        except Exception as e:
            log(f"[favstickers] Класс {name} не загрузился: {e!r}")
            return None

    def __hook_by_name(self, owner_class, name: str, hook, unhooks: list, required=True):
        """Перехват всех перегрузок метода по имени

        Перегрузки перебираем сами и ставим через hook_method, а не через
        hook_all_methods: у пользователя exteraGram тот вернул пусто для
        getRecentStickers, хотя hook_method на addRecentSticker этого же
        класса отработал, а ручной перебор те же перегрузки нашёл и
        перехватил. Пустой ответ SDK не отличает "метода нет" от "хук не
        встал", а здесь причина попадает и в лог, и в текст исключения.
        """
        methods = []
        installed = []
        problem = None
        try:
            methods = [m for m in owner_class.getDeclaredMethods() if str(m.getName()) == name]
            if not methods:
                problem = "метода нет в классе"
            for method in methods:
                method.setAccessible(True)
                unhook = self.hook_method(method, hook)
                if unhook is None:
                    problem = f"hook_method отказал на {method}"
                    continue
                installed.append(unhook)
        except Exception as e:
            problem = repr(e)
        # Откат в on_plugin_load снимает только то, что лежит в unhooks,
        # поэтому поставленное сдаём и при отказе
        unhooks.extend(installed)
        if problem is None:
            return
        reason = f"{name}: {problem} (найдено {len(methods)}, перехвачено {len(installed)})"
        if required:
            # Перегрузка без хука - это перегрузка, которую UI может позвать
            # мимо нас: полурабочий плагин хуже неработающего
            raise RuntimeError(f"не удалось перехватить {reason}")
        # Метод есть не во всех сборках, и без него теряется только
        # часть функциональности - не повод отказывать в загрузке
        log(f"[favstickers] {reason}, пропускаем")

    def create_settings(self) -> list:
        """Экран настроек плагина

        Счетчики в тексте кнопок, а не в subtext: у Text в установленной
        версии ui.settings параметра subtext нет, и лишний аргумент уронил
        бы построение экрана целиком.
        """
        try:
            own = self.db.count_stickers(self.__get_current_account_id())
            total, accounts = self.db.count_all()
            return [
                Header(text="Бекап избранного"),
                Text(
                    text=f"Экспорт текущего аккаунта ({own})",
                    on_click=lambda view: self.__export_current(),
                ),
                Text(
                    text=f"Экспорт всех аккаунтов ({total} в {accounts})",
                    on_click=lambda view: self.__export_all(),
                ),
                Divider(
                    text="Чтобы восстановить, откройте файл .stickers "
                    "в любом чате и подтвердите импорт"
                ),
            ]
        except Exception as e:
            # Экран строится в Java-UI, где исключение глотается молча:
            # без этой ветки пользователь получил бы пустые настройки
            # безо всякого объяснения
            log(f"[favstickers] Не удалось построить настройки: {e!r}")
            return [
                Header(text="Бекап избранного"),
                Divider(text="Не удалось прочитать базу стикеров"),
            ]

    def __export_current(self):
        self.__export(
            lambda: self.db.export_account(self.__get_current_account_id()),
            "account",
            "В избранном пусто, нечего экспортировать",
        )

    def __export_all(self):
        self.__export(
            lambda: self.db.export_all(),
            "all",
            "База пуста, нечего экспортировать",
        )

    def __export(self, build_data, scope: str, empty_message: str):
        """Записать бекап в кеш и отправить себе в Избранное

        Данные строит переданная функция, а не вызывающий: обращение к базе
        и к id аккаунта может бросить, а зовут нас прямо из Java-UI, где
        исключение проглотится молча.
        """
        fragment = None
        try:
            fragment = get_last_fragment()
            data = build_data()
            if not data.get("stickers"):
                BulletinHelper.show_info(empty_message, fragment)
                return
            path = os.path.join(get_cache_dir(), backup_filename(scope))
            write_file(path, json.dumps(data, ensure_ascii=False))
            # send_document ждет числовой peer, база ключуется строкой
            send_document(
                int(self.__get_current_account_id()), path, BACKUP_CAPTION
            )
            BulletinHelper.show_success("Бекап отправлен в Избранное", fragment)
        except Exception as e:
            log(f"[favstickers] Экспорт не удался: {e!r}")
            BulletinHelper.show_error("Не удалось выполнить экспорт", fragment)

    def __on_backup_tap(self, file_path: str):
        """Тап по файлу бекапа: прочитать, разобрать, спросить"""
        fragment = None
        try:
            fragment = get_last_fragment()
            if not os.path.exists(file_path):
                BulletinHelper.show_info(
                    "Файл ещё скачивается, дождитесь загрузки", fragment
                )
                return
            data = parse_backup(read_file(file_path))
        except BackupError as e:
            BulletinHelper.show_error(str(e), fragment)
            return
        except Exception as e:
            log(f"[favstickers] Не удалось прочитать бекап: {e!r}")
            BulletinHelper.show_error("Не удалось прочитать файл бекапа", fragment)
            return
        run_on_ui_thread(lambda: self.__confirm_import(data))

    def __confirm_import(self, data: dict):
        """Диалог выбора между слиянием и заменой"""
        fragment = None
        try:
            fragment = get_last_fragment()
            activity = fragment.getParentActivity() if fragment else None
            if activity is None:
                # Без Activity диалог не показать, а молча заменять нельзя
                self.__apply_import(data, replace=False)
                return
            if detect_scope(data) == "all":
                count = len(data["stickers"])
                what = f"В файле {count} стикеров из {len(data['accounts'])} аккаунтов"
            else:
                what = f"В файле {len(data['stickers'])} стикеров"
            builder = AlertDialogBuilder(activity)
            builder.set_title("Импорт избранного")
            builder.set_message(f"{what}. Добавить их к текущим или заменить?")
            builder.set_positive_button(
                "Добавить",
                lambda b, w: (b.dismiss(), self.__apply_import(data, False)),
            )
            builder.set_neutral_button(
                "Заменить",
                lambda b, w: (b.dismiss(), self.__apply_import(data, True)),
            )
            builder.set_negative_button("Отмена", lambda b, w: b.dismiss())
            builder.show()
        except Exception as e:
            log(f"[favstickers] Не удалось показать диалог импорта: {e!r}")
            BulletinHelper.show_error("Не удалось показать диалог импорта", fragment)

    def __save_safety_snapshot(self):
        """Слепок перед заменой: единственная разрушительная операция плагина

        Уходит в Избранное, а не только в кеш: кеш - приватный каталог
        приложения, и без рута пользователь оттуда ничего не достанет.
        В Избранном слепок можно просто открыть и импортировать обратно.

        Хвост в имени не для красоты: отправка может читать файл позже,
        и одноимённый слепок следующей замены испортил бы её содержимое.
        """
        stamp = dt.date.today().isoformat()
        path = os.path.join(
            get_cache_dir(),
            f"favorites-before-import-{stamp}-{uuid.uuid4().hex[:6]}{BACKUP_SUFFIX}",
        )
        write_file(path, json.dumps(self.db.export_all(), ensure_ascii=False))
        log(f"[favstickers] Страховочный слепок базы: {path}")
        try:
            send_document(
                int(self.__get_current_account_id()),
                path,
                "Избранное до импорта",
            )
        except Exception as e:
            # Копия в кеше уже есть, поэтому замену не отменяем: отказать
            # в импорте из-за неудачной отправки хуже, чем импортировать
            log(f"[favstickers] Не удалось отправить страховочный слепок: {e!r}")

    def __apply_import(self, data: dict, replace: bool):
        fragment = None
        try:
            fragment = get_last_fragment()
            if replace:
                self.__save_safety_snapshot()
            if detect_scope(data) == "all":
                applied, skipped = self.db.import_all(data, replace)
            else:
                applied, skipped = self.db.import_account(
                    data, self.__get_current_account_id(), replace
                )
            self.__notify_favorites_changed()
            message = f"Импортировано {applied}"
            if skipped:
                message += f", пропущено {skipped}"
            if replace:
                message += ". Прежнее избранное отправлено в Избранное"
            BulletinHelper.show_success(message, fragment)
        except Exception as e:
            log(f"[favstickers] Импорт не удался: {e!r}")
            BulletinHelper.show_error("Не удалось импортировать бекап", fragment)

    def on_plugin_load(self):
        MediaController = find_class("org.telegram.messenger.MediaDataController")
        media_instance = MediaController.getInstance(self.__get_current_account())
        media_class = media_instance.getClass()

        # Список копится по ходу установки, чтобы откатить уже поставленное:
        # полурабочий плагин хуже неработающего. Если оборвать загрузку на
        # середине, добавление продолжит уходить в базу, а список и звездочка
        # будут браться из ванили - и пользователь увидит вечные пять стикеров
        unhooks = []
        try:
            self.__install_hooks(media_class, media_instance, unhooks)
        except Exception:
            for unhook in unhooks:
                self.unhook_method(unhook)
            raise

    def __install_hooks(self, media_class, media_instance, unhooks: list):
        """Ставит все хуки, складывая хендлы отката в unhooks

        Хендлы нужны только на время загрузки: при выгрузке плагина мы хуки
        намеренно не снимаем.

        Порядок важен: сначала обязательные хуки, необязательные последними.
        Откат снимает всё поставленное до отказа, и при обратном порядке
        отказ обязательного хука снимал бы уже работающие необязательные.
        """
        TLRPCDocument = find_class("org.telegram.tgnet.TLRPC$Document")

        # Перехват метода добавления стикера в избраные
        addRecentStickerMethod = media_class.getDeclaredMethod(
            "addRecentSticker",
            J.Integer.TYPE,
            J.Object,
            TLRPCDocument,
            J.Integer.TYPE,
            J.Boolean.TYPE,
        )
        addRecentStickerMethod.setAccessible(True)
        self.__hook_one(
            addRecentStickerMethod,
            ChangeFavoriteStickerHook(
                on_add_favorite=self.db.add_sticker,
                on_remove_favorite=self.db.remove_sticker,
                refresh_panel=self.__notify_favorites_changed,
                get_account_id=self.__get_current_account_id,
            ),
            "addRecentSticker",
            unhooks,
        )

        # Перехват методов получения списка избранных стикеров.
        # Все перегрузки по имени, а не getDeclaredMethod: набор перегрузок
        # getRecentStickers в разных сборках exteraGram отличается, и привязка
        # к одной сигнатуре молча промахивается мимо той, которую зовет UI.
        # getRecentStickersNoCopy идет тем же хуком - через него читают
        # избранное подсказки стикеров по эмодзи.
        # NoCopy не обязателен: без него избранное не увидят только подсказки
        for method_name, required in (
            ("getRecentStickers", True),
            ("getRecentStickersNoCopy", False),
        ):
            self.__hook_by_name(
                media_class,
                method_name,
                GetFavoriteStickersHook(
                    self.db.get_all_stickers,
                    self.__get_current_account_id,
                    self.__import_vanilla_favorites,
                ),
                unhooks,
                required=required,
            )

        # Перехват метода проверки наличия стикера в избранных
        isStickerInFavoritesMethod = media_class.getDeclaredMethod(
            "isStickerInFavorites", jclass("org.telegram.tgnet.TLRPC$Document")
        )
        isStickerInFavoritesMethod.setAccessible(True)
        self.__hook_one(
            isStickerInFavoritesMethod,
            IsStickerInFavoritesHook(
                self.db.is_sticker_favorite, self.__get_current_account_id
            ),
            "isStickerInFavorites",
            unhooks,
        )

        # Перехват тапа по файлу бекапа. Не обязателен: без него теряется
        # только импорт, а ради него ронять весь плагин незачем
        android_utils_class = self.__load_class(
            "org.telegram.messenger.AndroidUtilities", media_class.getClassLoader()
        )
        if android_utils_class is not None:
            self.__hook_by_name(
                android_utils_class,
                "openForView",
                ImportBackupHook(self.__on_backup_tap),
                unhooks,
                required=False,
            )
