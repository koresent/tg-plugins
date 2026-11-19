# -*- coding: utf-8 -*-
import json
import re
from typing import Any, Dict, List, Optional

from android.graphics import Bitmap, Canvas, Color, Paint, RectF, Typeface
from android.graphics.drawable import BitmapDrawable, GradientDrawable
from android.text import InputType, TextWatcher
from android.util import TypedValue
from android.view import Gravity, View
from android.widget import (
    EditText,
    FrameLayout,
    HorizontalScrollView,
    ImageView,
    LinearLayout,
    ProgressBar,
    ScrollView,
    TextView,
)
from java import dynamic_proxy, jclass
from java.lang import Integer as JInt, Long as JLong
from java.util import ArrayList
from org.telegram.messenger import (
    AndroidUtilities,
    ApplicationLoader,
    UserConfig,
    Utilities,
)
from org.telegram.tgnet import TLRPC, RequestDelegate
from org.telegram.ui.ActionBar import BottomSheet, Theme

from android_utils import run_on_ui_thread
from base_plugin import BasePlugin, MethodHook
from client_utils import get_last_fragment
from hook_utils import find_class
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Input

__id__ = "kanban_board"
__name__ = "Telegram Kanban"
__description__ = "Native mobile Kanban board synced with Telegram chats, forum topics, and Saved Messages"
__author__ = "@koresent"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__category__ = ["productivity", "chats"]
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.4.3"

KANBAN_BLOCK_REGEX = re.compile(r"\n*---\n*<!--kanban:(.*?)-->\s*$", re.DOTALL)
KANBAN_CONFIG_REGEX = re.compile(r"<!--kanban-config:(.*?)(?:-->|$)", re.DOTALL)
KANBAN_BOARDS_REGEX = re.compile(r"<!--kanban-boards:(.*?)-->", re.DOTALL)
CHECKLIST_LINE_REGEX = re.compile(r"^(\s*[-*]\s+)\[([ xX])\]\s*(.*)$")

DEFAULT_COLUMNS = [
    {"id": "todo", "title": "To Do"},
    {"id": "in_progress", "title": "In Progress"},
    {"id": "done", "title": "Done"},
]

DEFAULT_PRIORITIES = [
    {"id": "urgent", "label": "Urgent", "color": "#fb4934"},
    {"id": "high", "label": "High", "color": "#fe8019"},
    {"id": "medium", "label": "Medium", "color": "#fabd2f"},
    {"id": "low", "label": "Low", "color": "#b8bb26"},
]

MENU_ITEM_ID = 889911


def dp(val: float) -> int:
    return int(AndroidUtilities.dp(val))


def get_activity():
    try:
        frag = get_last_fragment()
        if frag and hasattr(frag, "getParentActivity") and frag.getParentActivity():
            return frag.getParentActivity()
    except Exception:
        pass
    try:
        LaunchActivity = jclass("org.telegram.ui.LaunchActivity")
        if LaunchActivity.instance is not None:
            return LaunchActivity.instance
    except Exception:
        pass
    return None


class RequestDelegateProxy(dynamic_proxy(RequestDelegate)):
    def __init__(self, callback):
        super().__init__()
        self.callback = callback

    def run(self, response, error):
        self.callback(response, error)


class OnClickListenerProxy(dynamic_proxy(View.OnClickListener)):
    def __init__(self, callback):
        super().__init__()
        self.callback = callback

    def onClick(self, view):
        try:
            self.callback(view)
        except Exception:
            pass


def create_kanban_icon_drawable(size_dp: float = 24.0, color: int = 0xFFFFFFFF) -> BitmapDrawable:
    size = dp(size_dp)
    bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
    canvas = Canvas(bitmap)
    paint = Paint(Paint.ANTI_ALIAS_FLAG)

    c_int = int(color)
    a = int((c_int >> 24) & 0xFF)
    r = int((c_int >> 16) & 0xFF)
    g = int((c_int >> 8) & 0xFF)
    b = int(c_int & 0xFF)
    paint.setARGB(a, r, g, b)

    scale = size / 24.0
    rad = 1.6 * scale
    canvas.drawRoundRect(RectF(3.5 * scale, 3.5 * scale, 8.5 * scale, 20.5 * scale), rad, rad, paint)
    canvas.drawRoundRect(RectF(10.0 * scale, 3.5 * scale, 15.0 * scale, 16.5 * scale), rad, rad, paint)
    canvas.drawRoundRect(RectF(16.5 * scale, 3.5 * scale, 21.5 * scale, 12.5 * scale), rad, rad, paint)

    return BitmapDrawable(ApplicationLoader.applicationContext.getResources(), bitmap)


def create_rounded_drawable(bg_color: int, radius_dp: float, stroke_color: Optional[int] = None, stroke_dp: float = 0) -> GradientDrawable:
    d = GradientDrawable()
    d.setShape(GradientDrawable.RECTANGLE)
    d.setCornerRadius(dp(radius_dp))
    d.setColor(bg_color)
    if stroke_color is not None and stroke_dp > 0:
        d.setStroke(dp(stroke_dp), stroke_color)
    return d


class KanbanBoardPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self._active_board: Optional[Dict[str, Any]] = None
        self._boards: List[Dict[str, Any]] = []
        self._tasks: List[Dict[str, Any]] = []
        self._columns: List[Dict[str, Any]] = list(DEFAULT_COLUMNS)
        self._priorities: List[Dict[str, Any]] = list(DEFAULT_PRIORITIES)
        self._config_mid: Optional[int] = None
        self._active_col_id: str = "todo"
        self._search_query: str = ""
        self._sheet: Optional[Any] = None
        self._keep_alive_proxies: List[Any] = []

    def on_plugin_load(self):
        run_on_ui_thread(self._attach_header_button)
        self._hook_dialogs_resume()

    def on_plugin_unload(self):
        run_on_ui_thread(self._detach_header_button)
        if self._sheet:
            try:
                self._sheet.dismiss()
            except Exception:
                pass
            self._sheet = None

    def _hook_dialogs_resume(self):
        plugin_self = self
        DialogsActivity = find_class("org.telegram.ui.DialogsActivity")
        if not DialogsActivity:
            return

        class DialogsResumeHook(MethodHook):
            def after_hook(self, target, args, result):
                run_on_ui_thread(plugin_self._attach_header_button)

        try:
            self.hook_method(DialogsActivity, "onResume", DialogsResumeHook)
        except Exception as e:
            self.log(f"DialogsActivity.onResume hook failed: {e}")

    def _attach_header_button(self):
        try:
            frag = get_last_fragment()
            da = getattr(frag, "dialogsActivity", None) or frag
            ab = getattr(da, "actionBar", None) or (da.getActionBar() if hasattr(da, "getActionBar") else None)
            if not ab:
                return

            menu = ab.createMenu()
            for i in range(menu.getChildCount() - 1, -1, -1):
                ch = menu.getChildAt(i)
                if ch and (ch.getTag() == "kanban_btn" or getattr(ch, "getId", lambda: -1)() == MENU_ITEM_ID):
                    menu.removeView(ch)

            color = Theme.getColor(Theme.key_actionBarDefaultIcon)
            drawable = create_kanban_icon_drawable(24.0, color)

            item = menu.addItem(MENU_ITEM_ID, drawable)
            item.setTag("kanban_btn")
            item.setOverrideMenuClick(True)
            item.setContentDescription("Kanban Board")

            def on_click(v):
                self._open_kanban_sheet()

            proxy = OnClickListenerProxy(on_click)
            self._keep_alive_proxies.append(proxy)
            item.setOnClickListener(proxy)
            menu.requestLayout()
        except Exception as e:
            self.log(f"Error attaching header button: {e}")

    def _detach_header_button(self):
        try:
            frag = get_last_fragment()
            da = getattr(frag, "dialogsActivity", None) or frag
            ab = getattr(da, "actionBar", None) or (da.getActionBar() if hasattr(da, "getActionBar") else None)
            if ab:
                menu = ab.createMenu()
                for i in range(menu.getChildCount() - 1, -1, -1):
                    ch = menu.getChildAt(i)
                    if ch and (ch.getTag() == "kanban_btn" or getattr(ch, "getId", lambda: -1)() == MENU_ITEM_ID):
                        menu.removeView(ch)
                menu.requestLayout()
        except Exception as e:
            self.log(f"Error detaching header button: {e}")

    def _get_account_instance(self):
        account = UserConfig.selectedAccount
        from org.telegram.messenger import AccountInstance
        return AccountInstance.getInstance(account)

    def _load_boards_registry(self, on_done):
        ai = self._get_account_instance()
        req = TLRPC.TL_messages_getHistory()
        req.peer = ai.getMessagesController().getInputPeer(ai.getUserConfig().getClientUserId())
        req.limit = 50
        req.offset_id = 0
        req.offset_date = 0
        req.add_offset = 0
        req.max_id = 0
        req.min_id = 0
        req.hash = 0

        def handle_resp(resp, err):
            boards = []
            if not err and hasattr(resp, "messages"):
                msgs = resp.messages
                for i in range(msgs.size()):
                    m = msgs.get(i)
                    txt = getattr(m, "message", "") or ""
                    if "<!--kanban-boards:" in txt:
                        match = KANBAN_BOARDS_REGEX.search(txt)
                        if match:
                            try:
                                json_boards = json.loads(match.group(1))
                                if isinstance(json_boards, list):
                                    for jb in json_boards:
                                        p_id = jb.get("peerId")
                                        t_id = jb.get("topicId")
                                        if p_id is not None:
                                            boards.append({
                                                "id": jb.get("id") or f"board_{p_id}_{t_id or 0}",
                                                "peerId": int(p_id),
                                                "topicId": int(t_id) if t_id else None,
                                                "title": jb.get("title")
                                            })
                            except Exception:
                                pass

                        for line in txt.splitlines():
                            m_link = re.search(r"t\.me/c/(\d+)(?:/(\d+))?", line)
                            if m_link:
                                c_id = int(m_link.group(1))
                                t_id = int(m_link.group(2)) if m_link.group(2) else None
                                b_id = f"board_{c_id}_{t_id or 0}"
                                if not any(b["id"] == b_id for b in boards):
                                    boards.append({
                                        "id": b_id,
                                        "peerId": -c_id,
                                        "topicId": t_id,
                                        "title": None
                                    })
                        break

            mc = ai.getMessagesController()
            tc = mc.getTopicsController()
            for b in boards:
                if not b["title"]:
                    p_id = b["peerId"]
                    c_id = abs(p_id)
                    if c_id > 1000000000000:
                        c_id -= 1000000000000
                    if b["topicId"] and tc:
                        try:
                            topic = tc.findTopic(JLong(c_id).longValue(), JLong(b["topicId"]).longValue())
                            if topic and getattr(topic, "title", None):
                                b["title"] = topic.title
                        except Exception:
                            pass
                    if not b["title"]:
                        try:
                            chat = mc.getChat(JLong(c_id).longValue())
                            if chat and getattr(chat, "title", None):
                                b["title"] = chat.title
                        except Exception:
                            pass
                if not b["title"]:
                    b["title"] = f"Board {b['id']}"

            if not boards:
                boards.append({
                    "id": "saved_messages",
                    "peerId": int(ai.getUserConfig().getClientUserId()),
                    "topicId": None,
                    "title": "Saved Messages"
                })

            self._boards = boards
            last_id = self.get_setting("last_active_board_id", boards[0]["id"])
            active = next((b for b in boards if b["id"] == last_id), boards[0])
            self._active_board = active
            on_done(boards, active)

        proxy = RequestDelegateProxy(handle_resp)
        self._keep_alive_proxies.append(proxy)
        ai.getConnectionsManager().sendRequest(req, proxy)

    def _load_board_tasks(self, board: Dict[str, Any], on_done):
        ai = self._get_account_instance()
        mc = ai.getMessagesController()
        peer_id = board["peerId"]
        topic_id = board["topicId"]

        abs_p = abs(peer_id)
        chat_id = abs_p - 1000000000000 if abs_p > 1000000000000 else abs_p
        chat = mc.getChat(JLong(chat_id).longValue())

        def handle_resp(resp, err):
            columns = list(DEFAULT_COLUMNS)
            priorities = list(DEFAULT_PRIORITIES)
            tasks = []
            config_mid = None

            if not err and hasattr(resp, "messages"):
                msgs = resp.messages
                for i in range(msgs.size()):
                    m = msgs.get(i)
                    txt = getattr(m, "message", "") or ""

                    if KANBAN_CONFIG_REGEX.search(txt):
                        match = KANBAN_CONFIG_REGEX.search(txt)
                        if match:
                            try:
                                cfg = json.loads(match.group(1))
                                if isinstance(cfg.get("columns"), list) and len(cfg["columns"]) > 0:
                                    columns = cfg["columns"]
                                if isinstance(cfg.get("priorities"), list) and len(cfg["priorities"]) > 0:
                                    priorities = cfg["priorities"]
                                config_mid = m.id
                            except Exception:
                                pass
                        continue

                    match_task = KANBAN_BLOCK_REGEX.search(txt)
                    if match_task:
                        try:
                            meta = json.loads(match_task.group(1))
                        except Exception:
                            meta = {}

                        body = txt[:match_task.start()].strip()
                        lines = body.splitlines()
                        title = lines[0].strip() if lines else "Untitled"
                        desc_lines = lines[1:] if len(lines) > 1 else []
                        desc = "\n".join(desc_lines).strip()

                        checklist = []
                        for idx, l in enumerate(lines):
                            cm = CHECKLIST_LINE_REGEX.match(l)
                            if cm:
                                checklist.append({
                                    "line_idx": idx,
                                    "checked": cm.group(2).lower() == "x",
                                    "text": cm.group(3).strip()
                                })

                        tasks.append({
                            "id": m.id,
                            "col": meta.get("col", "todo"),
                            "pos": meta.get("pos", 1000),
                            "prio": meta.get("prio", "medium"),
                            "title": title,
                            "description": desc,
                            "raw_text": txt,
                            "checklist": checklist,
                            "total_check": len(checklist),
                            "done_check": sum(1 for c in checklist if c["checked"]),
                            "date": m.date
                        })

            tasks.sort(key=lambda t: t.get("pos", 1000))
            self._columns = columns
            self._priorities = priorities
            self._tasks = tasks
            self._config_mid = config_mid
            on_done(columns, priorities, tasks)

        proxy = RequestDelegateProxy(handle_resp)
        self._keep_alive_proxies.append(proxy)

        if topic_id and chat:
            peer = TLRPC.TL_inputPeerChannel()
            peer.channel_id = chat.id
            peer.access_hash = chat.access_hash
            req = TLRPC.TL_messages_getReplies()
            req.peer = peer
            req.msg_id = topic_id
            req.limit = 100
            req.offset_id = 0
            req.offset_date = 0
            req.add_offset = 0
            req.max_id = 0
            req.min_id = 0
            req.hash = 0
            ai.getConnectionsManager().sendRequest(req, proxy)
        else:
            req = TLRPC.TL_messages_getHistory()
            req.peer = mc.getInputPeer(JLong(peer_id).longValue())
            req.limit = 100
            req.offset_id = 0
            req.offset_date = 0
            req.add_offset = 0
            req.max_id = 0
            req.min_id = 0
            req.hash = 0
            ai.getConnectionsManager().sendRequest(req, proxy)

    def _send_edit_message(self, mid: int, new_text: str, on_done=None):
        ai = self._get_account_instance()
        mc = ai.getMessagesController()
        board = self._active_board
        if not board:
            return

        peer_id = board["peerId"]
        abs_p = abs(peer_id)
        chat_id = abs_p - 1000000000000 if abs_p > 1000000000000 else abs_p
        chat = mc.getChat(JLong(chat_id).longValue())

        req = TLRPC.TL_messages_editMessage()
        if chat:
            peer = TLRPC.TL_inputPeerChannel()
            peer.channel_id = chat.id
            peer.access_hash = chat.access_hash
            req.peer = peer
        else:
            req.peer = mc.getInputPeer(JLong(peer_id).longValue())

        req.id = mid
        req.message = new_text
        req.no_webpage = True

        def handle_resp(resp, err):
            if err:
                run_on_ui_thread(lambda: BulletinHelper.show_error(f"Edit failed: {err.text}"))
            if on_done:
                on_done(not bool(err))

        proxy = RequestDelegateProxy(handle_resp)
        self._keep_alive_proxies.append(proxy)
        ai.getConnectionsManager().sendRequest(req, proxy)

    def _send_new_task_message(self, col_id: str, title: str, description: str, priority: str, on_done=None):
        ai = self._get_account_instance()
        mc = ai.getMessagesController()
        board = self._active_board
        if not board:
            return

        peer_id = board["peerId"]
        topic_id = board["topicId"]
        abs_p = abs(peer_id)
        chat_id = abs_p - 1000000000000 if abs_p > 1000000000000 else abs_p
        chat = mc.getChat(JLong(chat_id).longValue())

        body = f"{title.strip()}\n{description.strip()}" if description.strip() else title.strip()
        col_tasks = [t for t in self._tasks if t.get("col") == col_id]
        next_pos = (max(t.get("pos", 1000) for t in col_tasks) + 1000) if col_tasks else 1000
        meta = {"col": col_id, "pos": next_pos, "prio": priority}
        text = f"{body}\n\n---\n<!--kanban:{json.dumps(meta)}-->"

        req = TLRPC.TL_messages_sendMessage()
        if chat:
            peer = TLRPC.TL_inputPeerChannel()
            peer.channel_id = chat.id
            peer.access_hash = chat.access_hash
            req.peer = peer
        else:
            req.peer = mc.getInputPeer(JLong(peer_id).longValue())

        req.message = text
        req.random_id = Utilities.random.nextLong()

        if topic_id:
            reply_to = TLRPC.TL_inputReplyToMessage()
            reply_to.reply_to_msg_id = topic_id
            reply_to.top_msg_id = topic_id
            req.reply_to = reply_to

        def handle_resp(resp, err):
            if err:
                run_on_ui_thread(lambda: BulletinHelper.show_error(f"Create failed: {err.text}"))
            if on_done:
                on_done(not bool(err))

        proxy = RequestDelegateProxy(handle_resp)
        self._keep_alive_proxies.append(proxy)
        ai.getConnectionsManager().sendRequest(req, proxy)

    def _send_delete_message(self, mid: int, on_done=None):
        ai = self._get_account_instance()
        mc = ai.getMessagesController()
        board = self._active_board
        if not board:
            return

        peer_id = board["peerId"]
        abs_p = abs(peer_id)
        chat_id = abs_p - 1000000000000 if abs_p > 1000000000000 else abs_p
        chat = mc.getChat(JLong(chat_id).longValue())

        def handle_resp(resp, err):
            if err:
                run_on_ui_thread(lambda: BulletinHelper.show_error(f"Delete failed: {err.text}"))
            if on_done:
                on_done(not bool(err))

        proxy = RequestDelegateProxy(handle_resp)
        self._keep_alive_proxies.append(proxy)

        if chat:
            req = TLRPC.TL_channels_deleteMessages()
            req.channel = mc.getInputChannel(chat)
            arr = ArrayList()
            arr.add(JInt(mid))
            req.id = arr
            ai.getConnectionsManager().sendRequest(req, proxy)
        else:
            req = TLRPC.TL_messages_deleteMessages()
            arr = ArrayList()
            arr.add(JInt(mid))
            req.id = arr
            req.revoke = True
            ai.getConnectionsManager().sendRequest(req, proxy)

    def _open_kanban_sheet(self):
        activity = get_activity()
        if not activity:
            BulletinHelper.show_error("Failed to open board")
            return

        builder = BottomSheet.Builder(activity)
        ctx = activity

        root = FrameLayout(ctx)
        root.setLayoutParams(FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, dp(650)))
        bg_color = Theme.getColor(Theme.key_dialogBackground)
        root.setBackgroundColor(bg_color)

        header = LinearLayout(ctx)
        header.setOrientation(LinearLayout.HORIZONTAL)
        header.setGravity(Gravity.CENTER_VERTICAL)
        header.setLayoutParams(FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, dp(54)))
        header.setPadding(dp(16), 0, dp(12), 0)

        title_box = LinearLayout(ctx)
        title_box.setOrientation(LinearLayout.VERTICAL)
        title_box.setLayoutParams(LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0))

        board_title_tv = TextView(ctx)
        board_title_tv.setText("Loading boards...")
        board_title_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
        board_title_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 17)
        board_title_tv.setTypeface(Typeface.DEFAULT_BOLD)
        title_box.addView(board_title_tv)

        subtitle_tv = TextView(ctx)
        subtitle_tv.setText("")
        subtitle_tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
        subtitle_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 12)
        title_box.addView(subtitle_tv)
        header.addView(title_box)

        res = ApplicationLoader.applicationContext.getResources()
        pkg = ApplicationLoader.applicationContext.getPackageName()

        search_btn = ImageView(ctx)
        search_btn.setLayoutParams(LinearLayout.LayoutParams(dp(40), dp(40)))
        search_btn.setScaleType(ImageView.ScaleType.CENTER)
        d_search = res.getDrawable(res.getIdentifier("ic_ab_search", "drawable", pkg))
        if d_search:
            d_search = d_search.mutate()
            d_search.setTint(Theme.getColor(Theme.key_dialogTextGray))
            search_btn.setImageDrawable(d_search)
        header.addView(search_btn)

        refresh_btn = ImageView(ctx)
        refresh_btn.setLayoutParams(LinearLayout.LayoutParams(dp(40), dp(40)))
        refresh_btn.setScaleType(ImageView.ScaleType.CENTER)
        d_refresh = res.getDrawable(res.getIdentifier("msg_retry", "drawable", pkg))
        if d_refresh:
            d_refresh = d_refresh.mutate()
            d_refresh.setTint(Theme.getColor(Theme.key_dialogTextGray))
            refresh_btn.setImageDrawable(d_refresh)
        header.addView(refresh_btn)

        close_btn = ImageView(ctx)
        close_btn.setLayoutParams(LinearLayout.LayoutParams(dp(40), dp(40)))
        close_btn.setScaleType(ImageView.ScaleType.CENTER)
        d_close = res.getDrawable(res.getIdentifier("msg_close", "drawable", pkg))
        if d_close:
            d_close = d_close.mutate()
            d_close.setTint(Theme.getColor(Theme.key_dialogTextGray))
            close_btn.setImageDrawable(d_close)
        header.addView(close_btn)
        root.addView(header)

        search_container = LinearLayout(ctx)
        search_container.setOrientation(LinearLayout.HORIZONTAL)
        search_container.setGravity(Gravity.CENTER_VERTICAL)
        search_lp = FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, dp(44))
        search_lp.topMargin = dp(54)
        search_lp.leftMargin = dp(16)
        search_lp.rightMargin = dp(16)
        search_container.setLayoutParams(search_lp)
        search_container.setBackground(create_rounded_drawable(0x18FFFFFF, 8.0))
        search_container.setPadding(dp(12), 0, dp(12), 0)
        search_container.setVisibility(View.GONE)

        search_edit = EditText(ctx)
        search_edit.setLayoutParams(LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT))
        search_edit.setBackground(None)
        search_edit.setHint("Search tasks...")
        search_edit.setHintTextColor(0x77FFFFFF)
        search_edit.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
        search_edit.setTextSize(TypedValue.COMPLEX_UNIT_SP, 14)
        search_edit.setInputType(InputType.TYPE_CLASS_TEXT)
        search_container.addView(search_edit)
        root.addView(search_container)

        tabs_scroll = HorizontalScrollView(ctx)
        tabs_scroll.setHorizontalScrollBarEnabled(False)
        tabs_lp = FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, dp(44))
        tabs_lp.topMargin = dp(54)
        tabs_scroll.setLayoutParams(tabs_lp)

        tabs_layout = LinearLayout(ctx)
        tabs_layout.setOrientation(LinearLayout.HORIZONTAL)
        tabs_layout.setPadding(dp(14), dp(4), dp(14), dp(4))
        tabs_scroll.addView(tabs_layout)
        root.addView(tabs_scroll)

        cards_scroll = ScrollView(ctx)
        cards_lp = FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT)
        cards_lp.topMargin = dp(104)
        cards_lp.bottomMargin = dp(64)
        cards_scroll.setLayoutParams(cards_lp)

        cards_layout = LinearLayout(ctx)
        cards_layout.setOrientation(LinearLayout.VERTICAL)
        cards_layout.setPadding(dp(16), dp(8), dp(16), dp(16))
        cards_scroll.addView(cards_layout)
        root.addView(cards_scroll)

        progress = ProgressBar(ctx)
        progress.setLayoutParams(FrameLayout.LayoutParams(dp(36), dp(36), Gravity.CENTER))
        root.addView(progress)

        fab = FrameLayout(ctx)
        fab_size = dp(52)
        fab_lp = FrameLayout.LayoutParams(fab_size, fab_size, Gravity.RIGHT | Gravity.BOTTOM)
        fab_lp.rightMargin = dp(20)
        fab_lp.bottomMargin = dp(16)
        fab.setLayoutParams(fab_lp)
        fab_color = Theme.getColor(Theme.key_chats_actionBackground)
        fab.setBackground(create_rounded_drawable(fab_color, 26.0))
        fab.setElevation(dp(4))

        fab_iv = ImageView(ctx)
        fab_iv.setLayoutParams(FrameLayout.LayoutParams(dp(26), dp(26), Gravity.CENTER))
        d_plus = res.getDrawable(res.getIdentifier("msg_add", "drawable", pkg))
        if d_plus:
            d_plus = d_plus.mutate()
            d_plus.setTint(Color.WHITE)
            fab_iv.setImageDrawable(d_plus)
        fab.addView(fab_iv)
        root.addView(fab)

        def refresh_tabs():
            tabs_layout.removeAllViews()
            active_col = self._active_col_id

            for col in self._columns:
                c_id = col["id"]
                c_title = col.get("title", c_id)
                count = sum(1 for t in self._tasks if t.get("col") == c_id)

                tab_btn = LinearLayout(ctx)
                tab_btn.setOrientation(LinearLayout.HORIZONTAL)
                tab_btn.setGravity(Gravity.CENTER)
                tab_btn.setPadding(dp(14), dp(6), dp(14), dp(6))
                t_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                t_lp.rightMargin = dp(8)
                tab_btn.setLayoutParams(t_lp)

                is_active = (c_id == active_col)
                if is_active:
                    accent_col = Theme.getColor(Theme.key_chats_actionBackground)
                    tab_btn.setBackground(create_rounded_drawable(accent_col, 16.0))
                else:
                    tab_btn.setBackground(create_rounded_drawable(0x14FFFFFF, 16.0))

                tv = TextView(ctx)
                tv.setText(f"{c_title} ({count})")
                tv.setTextColor(Color.WHITE if is_active else Theme.getColor(Theme.key_dialogTextGray))
                tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                if is_active:
                    tv.setTypeface(Typeface.DEFAULT_BOLD)
                tab_btn.addView(tv)

                def on_tab_tap(v, selected_id=c_id):
                    self._active_col_id = selected_id
                    refresh_tabs()
                    render_cards()

                tab_proxy = OnClickListenerProxy(on_tab_tap)
                self._keep_alive_proxies.append(tab_proxy)
                tab_btn.setOnClickListener(tab_proxy)
                tabs_layout.addView(tab_btn)

        def render_cards():
            cards_layout.removeAllViews()
            active_col = self._active_col_id
            query = self._search_query.lower().strip()

            filtered = [
                t for t in self._tasks
                if t.get("col") == active_col and (
                    not query or query in t.get("title", "").lower() or query in t.get("description", "").lower()
                )
            ]

            subtitle_tv.setText(f"{len(self._tasks)} total tasks")

            if not filtered:
                empty_tv = TextView(ctx)
                empty_tv.setText("No tasks in this column")
                empty_tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
                empty_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 14)
                empty_tv.setGravity(Gravity.CENTER)
                empty_tv.setLayoutParams(LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, dp(150)))
                cards_layout.addView(empty_tv)
                return

            cols = self._columns
            curr_idx = next((i for i, c in enumerate(cols) if c["id"] == active_col), 0)

            for task in filtered:
                card = LinearLayout(ctx)
                card.setOrientation(LinearLayout.VERTICAL)
                c_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                c_lp.bottomMargin = dp(10)
                card.setLayoutParams(c_lp)
                card.setBackground(create_rounded_drawable(0xFF252525, 12.0, 0x1EFFFFFF, 1.0))
                card.setPadding(dp(14), dp(12), dp(14), dp(12))

                top_row = LinearLayout(ctx)
                top_row.setOrientation(LinearLayout.HORIZONTAL)
                top_row.setGravity(Gravity.CENTER_VERTICAL)
                top_row.setLayoutParams(LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT))

                title_tv = TextView(ctx)
                title_tv.setText(task.get("title", ""))
                title_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
                title_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 15)
                title_tv.setTypeface(Typeface.DEFAULT_BOLD)
                t_lp = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
                t_lp.rightMargin = dp(8)
                title_tv.setLayoutParams(t_lp)
                top_row.addView(title_tv)

                if curr_idx > 0:
                    prev_col = cols[curr_idx - 1]
                    left_btn = TextView(ctx)
                    left_btn.setText("<")
                    left_btn.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
                    left_btn.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                    left_btn.setPadding(dp(8), dp(4), dp(8), dp(4))
                    left_btn.setBackground(create_rounded_drawable(0x1AFFFFFF, 6.0))

                    def on_move_left(v, t=task, target_id=prev_col["id"]):
                        self._move_task_column(t, target_id)

                    lp_proxy = OnClickListenerProxy(on_move_left)
                    self._keep_alive_proxies.append(lp_proxy)
                    left_btn.setOnClickListener(lp_proxy)
                    top_row.addView(left_btn)

                if curr_idx < len(cols) - 1:
                    next_col = cols[curr_idx + 1]
                    right_btn = TextView(ctx)
                    right_btn.setText(">")
                    right_btn.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
                    right_btn.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                    right_btn.setPadding(dp(8), dp(4), dp(8), dp(4))
                    right_btn.setBackground(create_rounded_drawable(0x1AFFFFFF, 6.0))
                    r_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                    r_lp.leftMargin = dp(6)
                    right_btn.setLayoutParams(r_lp)

                    def on_move_right(v, t=task, target_id=next_col["id"]):
                        self._move_task_column(t, target_id)

                    rp_proxy = OnClickListenerProxy(on_move_right)
                    self._keep_alive_proxies.append(rp_proxy)
                    right_btn.setOnClickListener(rp_proxy)
                    top_row.addView(right_btn)

                card.addView(top_row)

                desc = task.get("description", "")
                if desc:
                    pure_lines = [l for l in desc.splitlines() if not CHECKLIST_LINE_REGEX.match(l)]
                    pure_desc = "\n".join(pure_lines).strip()
                    if pure_desc:
                        desc_tv = TextView(ctx)
                        desc_tv.setText(pure_desc)
                        desc_tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
                        desc_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                        desc_tv.setMaxLines(3)
                        d_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                        d_lp.topMargin = dp(6)
                        desc_tv.setLayoutParams(d_lp)
                        card.addView(desc_tv)

                checklist = task.get("checklist", [])
                if checklist:
                    check_box = LinearLayout(ctx)
                    check_box.setOrientation(LinearLayout.VERTICAL)
                    cb_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                    cb_lp.topMargin = dp(8)
                    check_box.setLayoutParams(cb_lp)

                    for item_dict in checklist:
                        row = LinearLayout(ctx)
                        row.setOrientation(LinearLayout.HORIZONTAL)
                        row.setGravity(Gravity.CENTER_VERTICAL)
                        r_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                        r_lp.bottomMargin = dp(4)
                        row.setLayoutParams(r_lp)

                        icon_tv = TextView(ctx)
                        is_checked = item_dict["checked"]
                        icon_tv.setText("[x] " if is_checked else "[ ] ")
                        icon_tv.setTextColor(0xFF4CAF50 if is_checked else Theme.getColor(Theme.key_dialogTextGray))
                        icon_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                        icon_tv.setTypeface(Typeface.MONOSPACE)
                        row.addView(icon_tv)

                        item_tv = TextView(ctx)
                        item_tv.setText(item_dict["text"])
                        item_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlack) if not is_checked else Theme.getColor(Theme.key_dialogTextGray))
                        item_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
                        row.addView(item_tv)

                        def on_check_tap(v, t=task, item=item_dict):
                            self._toggle_task_checklist(t, item)

                        check_proxy = OnClickListenerProxy(on_check_tap)
                        self._keep_alive_proxies.append(check_proxy)
                        row.setOnClickListener(check_proxy)
                        check_box.addView(row)

                    card.addView(check_box)

                bottom_row = LinearLayout(ctx)
                bottom_row.setOrientation(LinearLayout.HORIZONTAL)
                bottom_row.setGravity(Gravity.CENTER_VERTICAL)
                b_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
                b_lp.topMargin = dp(10)
                bottom_row.setLayoutParams(b_lp)

                prio_id = task.get("prio", "medium")
                prio_info = next((p for p in self._priorities if p["id"] == prio_id), {"label": prio_id.title(), "color": "#fabd2f"})
                prio_pill = TextView(ctx)
                prio_pill.setText(prio_info.get("label", prio_id).upper())
                prio_pill.setTextSize(TypedValue.COMPLEX_UNIT_SP, 10)
                prio_pill.setTypeface(Typeface.DEFAULT_BOLD)
                prio_color = Color.parseColor(prio_info.get("color", "#fabd2f"))
                prio_pill.setTextColor(prio_color)
                prio_pill.setPadding(dp(6), dp(2), dp(6), dp(2))
                prio_pill.setBackground(create_rounded_drawable(0x22FFFFFF, 4.0, prio_color, 1.0))
                bottom_row.addView(prio_pill)

                if task.get("total_check", 0) > 0:
                    prog_tv = TextView(ctx)
                    prog_tv.setText(f"  {task['done_check']}/{task['total_check']}")
                    prog_tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
                    prog_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 11)
                    bottom_row.addView(prog_tv)

                id_tv = TextView(ctx)
                id_tv.setText(f"  #{task['id']}")
                id_tv.setTextColor(0x66FFFFFF)
                id_tv.setTextSize(TypedValue.COMPLEX_UNIT_SP, 11)
                id_tv.setLayoutParams(LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0))
                bottom_row.addView(id_tv)

                del_btn = TextView(ctx)
                del_btn.setText("Delete")
                del_btn.setTextColor(0xFFE53935)
                del_btn.setTextSize(TypedValue.COMPLEX_UNIT_SP, 11)
                del_btn.setPadding(dp(6), dp(4), dp(6), dp(4))

                def on_delete_tap(v, t=task):
                    self._confirm_delete_task(t)

                del_proxy = OnClickListenerProxy(on_delete_tap)
                self._keep_alive_proxies.append(del_proxy)
                del_btn.setOnClickListener(del_proxy)
                bottom_row.addView(del_btn)

                card.addView(bottom_row)
                cards_layout.addView(card)

        def load_current_board():
            progress.setVisibility(View.VISIBLE)
            if not self._active_board:
                return
            board_title_tv.setText(self._active_board.get("title", "Board") + " v")
            self._load_board_tasks(self._active_board, lambda cols, prios, tasks: run_on_ui_thread(on_data_ready))

        def on_data_ready():
            progress.setVisibility(View.GONE)
            refresh_tabs()
            render_cards()

        def on_boards_ready(boards, active):
            self._boards = boards
            self._active_board = active
            load_current_board()

        def on_title_click(v):
            if not self._boards or len(self._boards) <= 1:
                return
            builder_alert = AlertDialogBuilder(activity)
            builder_alert.setTitle("Select Board")
            items = [b.get("title") or b["id"] for b in self._boards]

            def on_board_selected(idx):
                self._active_board = self._boards[idx]
                self.set_setting("last_active_board_id", self._active_board["id"])
                load_current_board()

            builder_alert.setItems(items, on_board_selected)
            builder_alert.show()

        title_proxy = OnClickListenerProxy(on_title_click)
        self._keep_alive_proxies.append(title_proxy)
        title_box.setOnClickListener(title_proxy)

        def on_search_click(v):
            if search_container.getVisibility() == View.VISIBLE:
                search_container.setVisibility(View.GONE)
                tabs_lp.topMargin = dp(54)
                cards_lp.topMargin = dp(104)
                self._search_query = ""
                search_edit.setText("")
                render_cards()
            else:
                search_container.setVisibility(View.VISIBLE)
                tabs_lp.topMargin = dp(104)
                cards_lp.topMargin = dp(154)
                search_edit.requestFocus()
            root.requestLayout()

        search_proxy = OnClickListenerProxy(on_search_click)
        self._keep_alive_proxies.append(search_proxy)
        search_btn.setOnClickListener(search_proxy)

        class SearchWatcher(dynamic_proxy(TextWatcher)):
            def beforeTextChanged(self, s, start, count, after):
                pass
            def onTextChanged(self, s, start, before, count):
                pass
            def afterTextChanged(self, s):
                plugin_self._search_query = str(s.toString()) if s else ""
                render_cards()

        plugin_self = self
        watcher_proxy = SearchWatcher()
        self._keep_alive_proxies.append(watcher_proxy)
        search_edit.addTextChangedListener(watcher_proxy)

        def on_refresh_click(v):
            load_current_board()

        refresh_proxy = OnClickListenerProxy(on_refresh_click)
        self._keep_alive_proxies.append(refresh_proxy)
        refresh_btn.setOnClickListener(refresh_proxy)

        def on_close_click(v):
            if self._sheet:
                self._sheet.dismiss()

        close_proxy = OnClickListenerProxy(on_close_click)
        self._keep_alive_proxies.append(close_proxy)
        close_btn.setOnClickListener(close_proxy)

        def on_fab_click(v):
            self._show_create_task_dialog(activity, lambda: load_current_board())

        fab_proxy = OnClickListenerProxy(on_fab_click)
        self._keep_alive_proxies.append(fab_proxy)
        fab.setOnClickListener(fab_proxy)

        builder.setCustomView(root)
        sheet = builder.create()
        try:
            sheet.setCanDismissWithSwipe(False)
            sheet.setAllowNestedScroll(False)
        except Exception:
            pass
        self._sheet = sheet
        sheet.show()

        self._load_boards_registry(lambda boards, active: run_on_ui_thread(lambda: on_boards_ready(boards, active)))

    def _toggle_task_checklist(self, task: Dict[str, Any], item_dict: Dict[str, Any]):
        raw_text = task["raw_text"]
        match_task = KANBAN_BLOCK_REGEX.search(raw_text)
        if not match_task:
            return

        body = raw_text[:match_task.start()]
        footer = raw_text[match_task.start():]
        lines = body.splitlines()

        line_idx = item_dict["line_idx"]
        if line_idx < 0 or line_idx >= len(lines):
            return

        curr_line = lines[line_idx]
        cm = CHECKLIST_LINE_REGEX.match(curr_line)
        if not cm:
            return

        new_check = " " if item_dict["checked"] else "x"
        lines[line_idx] = f"{cm.group(1)}[{new_check}] {cm.group(3)}"
        new_text = "\n".join(lines) + footer

        item_dict["checked"] = not item_dict["checked"]
        task["raw_text"] = new_text
        task["done_check"] = sum(1 for c in task["checklist"] if c["checked"])

        self._send_edit_message(task["id"], new_text)

    def _move_task_column(self, task: Dict[str, Any], target_col_id: str):
        raw_text = task["raw_text"]
        match_task = KANBAN_BLOCK_REGEX.search(raw_text)
        if not match_task:
            return

        try:
            meta = json.loads(match_task.group(1))
        except Exception:
            meta = {}

        meta["col"] = target_col_id
        body = raw_text[:match_task.start()].strip()
        new_text = f"{body}\n\n---\n<!--kanban:{json.dumps(meta)}-->"

        task["col"] = target_col_id
        task["raw_text"] = new_text

        if self._sheet:
            self._open_kanban_sheet()

        self._send_edit_message(task["id"], new_text)

    def _confirm_delete_task(self, task: Dict[str, Any]):
        activity = get_activity()
        if not activity:
            return

        builder = AlertDialogBuilder(activity)
        builder.setTitle("Delete Task?")
        builder.setMessage(f"\"{task.get('title')}\"\n\nThis message will be deleted from Telegram.")

        def on_confirm():
            self._send_delete_message(task["id"], lambda success: run_on_ui_thread(lambda: self._on_task_deleted(task)))

        builder.setPositiveButton("Delete", lambda: on_confirm())
        builder.setNegativeButton("Cancel", None)
        builder.show()

    def _on_task_deleted(self, task: Dict[str, Any]):
        self._tasks = [t for t in self._tasks if t["id"] != task["id"]]
        if self._sheet:
            self._open_kanban_sheet()

    def _show_create_task_dialog(self, activity, on_created):
        builder = AlertDialogBuilder(activity)
        builder.setTitle("New Task")

        ctx = activity
        layout = LinearLayout(ctx)
        layout.setOrientation(LinearLayout.VERTICAL)
        layout.setPadding(dp(20), dp(10), dp(20), dp(10))

        title_et = EditText(ctx)
        title_et.setHint("Task Title *")
        title_et.setHintTextColor(0x77FFFFFF)
        title_et.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
        title_et.setTextSize(TypedValue.COMPLEX_UNIT_SP, 15)
        layout.addView(title_et)

        desc_et = EditText(ctx)
        desc_et.setHint("Description or checklist (- [ ] item)")
        desc_et.setHintTextColor(0x77FFFFFF)
        desc_et.setTextColor(Theme.getColor(Theme.key_dialogTextBlack))
        desc_et.setTextSize(TypedValue.COMPLEX_UNIT_SP, 13)
        desc_et.setMinLines(3)
        d_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
        d_lp.topMargin = dp(8)
        desc_et.setLayoutParams(d_lp)
        layout.addView(desc_et)

        prio_label = TextView(ctx)
        prio_label.setText("Priority:")
        prio_label.setTextColor(Theme.getColor(Theme.key_dialogTextGray))
        prio_label.setTextSize(TypedValue.COMPLEX_UNIT_SP, 12)
        p_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT)
        p_lp.topMargin = dp(10)
        prio_label.setLayoutParams(p_lp)
        layout.addView(prio_label)

        prio_layout = LinearLayout(ctx)
        prio_layout.setOrientation(LinearLayout.HORIZONTAL)
        pl_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
        pl_lp.topMargin = dp(4)
        prio_layout.setLayoutParams(pl_lp)

        selected_prio = ["medium"]
        prio_buttons = []

        for p in self._priorities:
            pid = p["id"]
            pbtn = TextView(ctx)
            pbtn.setText(p.get("label", pid).upper())
            pbtn.setTextSize(TypedValue.COMPLEX_UNIT_SP, 10)
            pbtn.setTypeface(Typeface.DEFAULT_BOLD)
            pbtn.setPadding(dp(8), dp(4), dp(8), dp(4))
            b_lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT)
            b_lp.rightMargin = dp(6)
            pbtn.setLayoutParams(b_lp)

            def update_prio_styles():
                for pb, pobj in prio_buttons:
                    c = Color.parseColor(pobj.get("color", "#fabd2f"))
                    if pobj["id"] == selected_prio[0]:
                        pb.setTextColor(Color.WHITE)
                        pb.setBackground(create_rounded_drawable(c, 6.0))
                    else:
                        pb.setTextColor(c)
                        pb.setBackground(create_rounded_drawable(0x18FFFFFF, 6.0, c, 1.0))

            def on_prio_click(v, target_p=pid):
                selected_prio[0] = target_p
                update_prio_styles()

            pr_proxy = OnClickListenerProxy(on_prio_click)
            self._keep_alive_proxies.append(pr_proxy)
            pbtn.setOnClickListener(pr_proxy)
            prio_layout.addView(pbtn)
            prio_buttons.append((pbtn, p))

        update_prio_styles()
        layout.addView(prio_layout)

        builder.setView(layout)

        def on_create_confirm():
            t_title = str(title_et.getText().toString()).strip()
            if not t_title:
                BulletinHelper.show_error("Enter task title")
                return
            t_desc = str(desc_et.getText().toString()).strip()
            self._send_new_task_message(self._active_col_id, t_title, t_desc, selected_prio[0], lambda success: run_on_ui_thread(on_created))

        builder.setPositiveButton("Create", lambda: on_create_confirm())
        builder.setNegativeButton("Cancel", None)
        builder.show()

    def create_settings(self) -> List[Any]:
        rows: List[Any] = []
        rows.append(Header("Telegram Kanban"))
        rows.append(
            Divider(
                text=(
                    "Mobile Kanban board synced with Telegram chats, forum topics, and Saved Messages.\n\n"
                    "Fully compatible with Web K Kanban UserScript: tasks and checklists are stored directly in Telegram messages."
                )
            )
        )

        rows.append(Header("Settings"))
        rows.append(
            Input(
                key="last_active_board_id",
                text="Active Board ID",
                default="",
                icon="msg_folders",
            )
        )
        return rows
