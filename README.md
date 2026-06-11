import tkinter as tk
from tkinter import messagebox
import requests
import threading
import random
import string
import time
import json
import os
from datetime import datetime

BASE = "https://api.mail.tm"
SAVE_FILE = "voidmail_accounts.json"

# ── helpers ────────────────────────────────────────────────────────────
def rand(n, c=string.ascii_lowercase + string.digits):
    return ''.join(random.choices(c, k=n))

def gen_password():
    return rand(8) + rand(3, string.digits) + random.choice("!@#$%")

def time_ago(iso):
    try:
        diff = int(time.time() - datetime.fromisoformat(
            iso.replace("Z", "+00:00")).timestamp())
        if diff < 60:   return "just now"
        if diff < 3600: return f"{diff//60}m ago"
        if diff < 86400: return f"{diff//3600}h ago"
        return f"{diff//86400}d ago"
    except:
        return ""

def fmt_date(iso):
    try:
        return datetime.fromisoformat(
            iso.replace("Z", "+00:00")).strftime("%b %d, %Y - %H:%M")
    except:
        return ""

def load_accounts():
    if os.path.exists(SAVE_FILE):
        with open(SAVE_FILE) as f:
            return json.load(f)
    return []

def save_accounts(accs):
    with open(SAVE_FILE, "w") as f:
        json.dump(accs, f, indent=2)

def add_account(email, pw):
    accs = load_accounts()
    accs.append({"email": email, "password": pw})
    save_accounts(accs)

def remove_account(email):
    accs = load_accounts()
    accs = [a for a in accs if a["email"] != email]
    save_accounts(accs)


# ── Color Palette ───────────────────────────────────────────────────────
C = {
    "bg": "#0a0a0f",
    "bg2": "#111118",
    "bg3": "#161622",
    "card": "#1a1a28",
    "border": "#252538",
    "accent": "#7289ff",
    "accent2": "#4a5fd1",
    "accent_bg": "#1a1f3a",
    "text": "#e8eaf0",
    "muted": "#9b9db5",
    "dim": "#5e6078",
    "ghost": "#363650",
    "green": "#5ee8a0",
    "green_bg": "#1a2e22",
    "red": "#ff6b7b",
    "red_bg": "#3d1a1f",
    "hover": "#1f1f30",
    "selected": "#1e2440",
}


class VoidMail:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("VoidMail - Temporary Email")
        self.root.geometry("1000x650")
        self.root.configure(bg=C["bg"])
        self.root.minsize(800, 500)
        self.root.update_idletasks()
        w = self.root.winfo_screenwidth()
        h = self.root.winfo_screenheight()
        x = (w - 1000) // 2
        y = (h - 650) // 2
        self.root.geometry(f"1000x650+{x}+{y}")

        self.account = None
        self.token = None
        self.messages = []
        self.selected = None
        self.polling = False
        self.pw_show = False
        self._seen_ids = set()
        self._poll_thread = None

        accs = load_accounts()
        if accs:
            self._screen_accounts(accs)
        else:
            self._screen_generate()

        self.root.mainloop()

    # ── utility ──────────────────────────────────────────────────────────
    def _clear(self):
        self.polling = False
        for w in self.root.winfo_children():
            w.destroy()

    def _toast(self, msg):
        toast = tk.Toplevel(self.root)
        toast.overrideredirect(True)
        toast.attributes("-topmost", True)
        frame = tk.Frame(toast, bg=C["card"], highlightthickness=1,
                         highlightbackground=C["border"])
        frame.pack()
        tk.Label(frame, text=f"  {msg}  ", font=("Segoe UI", 9),
                 bg=C["card"], fg=C["text"], padx=16, pady=10).pack()
        self.root.update_idletasks()
        rx = self.root.winfo_x() + self.root.winfo_width()
        ry = self.root.winfo_y() + self.root.winfo_height()
        toast.geometry(f"+{rx-250}+{ry-70}")
        toast.after(2000, toast.destroy)

    def _make_btn(self, parent, text, cmd, accent=False, danger=False, small=False):
        """Simple styled button."""
        bg = C["accent"] if accent else (C["red_bg"] if danger else C["bg3"])
        fg = "#fff" if accent else (C["red"] if danger else C["muted"])
        active_bg = C["accent2"] if accent else ("#4a2020" if danger else C["hover"])
        font_size = 8 if small else 10
        b = tk.Button(parent, text=text, font=("Segoe UI", font_size),
                      bg=bg, fg=fg, bd=0, relief="flat",
                      activebackground=active_bg, activeforeground="#fff",
                      cursor="hand2", command=cmd, padx=10 if small else 14, pady=4 if small else 7)
        return b

    # ══════════════════════════════════════════════════════════════════════
    #  ACCOUNT LIST SCREEN
    # ══════════════════════════════════════════════════════════════════════
    def _screen_accounts(self, accounts):
        self._clear()
        outer = tk.Frame(self.root, bg=C["bg"])
        outer.place(relx=0.5, rely=0.5, anchor="center")

        # Logo header
        logo_row = tk.Frame(outer, bg=C["bg"])
        logo_row.pack(pady=(0, 24))
        tk.Label(logo_row, text="✦", font=("Segoe UI", 22), bg=C["bg"],
                 fg=C["accent"]).pack(side="left", padx=(0, 10))
        tk.Label(logo_row, text="VoidMail", font=("Segoe UI", 22, "bold"),
                 bg=C["bg"], fg=C["text"]).pack(side="left")

        card = tk.Frame(outer, bg=C["card"], highlightthickness=1,
                        highlightbackground=C["border"])
        card.pack(ipadx=24, ipady=20)

        tk.Label(card, text="Your Inboxes", font=("Segoe UI", 14, "bold"),
                 bg=C["card"], fg=C["text"]).pack(anchor="w", pady=(0, 4))
        tk.Label(card, text=f"{len(accounts)} saved",
                 font=("Segoe UI", 9), bg=C["card"], fg=C["dim"]).pack(anchor="w", pady=(0, 18))

        for acc in accounts:
            row = tk.Frame(card, bg=C["bg3"], cursor="hand2",
                           highlightthickness=1, highlightbackground=C["border"])
            row.pack(fill="x", pady=3)
            inner = tk.Frame(row, bg=C["bg3"], padx=14, pady=10)
            inner.pack(fill="x")

            # avatar
            av = tk.Frame(inner, bg=C["accent_bg"], width=32, height=32)
            av.pack_propagate(False)
            av.pack(side="left", padx=(0, 10))
            tk.Label(av, text=acc["email"][0].upper(), font=("Segoe UI", 10, "bold"),
                     bg=C["accent_bg"], fg=C["accent"]).place(relx=0.5, rely=0.5, anchor="center")

            tk.Label(inner, text=acc["email"], font=("Cascadia Code", 9),
                     bg=C["bg3"], fg=C["muted"]).pack(side="left", fill="x", expand=True)

            self._make_btn(inner, "Open", lambda a=acc: self._login(a), accent=True, small=True).pack(side="left", padx=4)
            self._make_btn(inner, "Del", lambda a=acc: self._delete_account(a), danger=True, small=True).pack(side="left")

            # hover
            def on_enter(e, r=row): r.configure(bg=C["hover"])
            def on_leave(e, r=row): r.configure(bg=C["bg3"])
            for w in (row, inner, av):
                w.bind("<Enter>", on_enter)
                w.bind("<Leave>", on_leave)

        self._make_btn(card, "+ New Address", self._screen_generate).pack(fill="x", pady=(16, 0))

    def _login(self, acc):
        self._clear()
        self._show_loading("Logging in...")
        threading.Thread(target=self._auth_login, args=(acc,), daemon=True).start()

    def _auth_login(self, acc):
        try:
            r = requests.post(BASE + "/token",
                              json={"address": acc["email"], "password": acc["password"]}, timeout=10)
            if r.ok:
                self.token = r.json()["token"]
                self.account = acc
                self.root.after(0, self._screen_app)
            else:
                self.root.after(0, lambda: [self._screen_accounts(load_accounts()),
                                            self._toast("Account expired")])
        except Exception as e:
            self.root.after(0, lambda: [self._screen_accounts(load_accounts()),
                                        self._toast(str(e))])

    def _delete_account(self, acc):
        if messagebox.askyesno("Delete", f"Delete {acc['email']}?"):
            remove_account(acc["email"])
            rem = load_accounts()
            if rem:
                self._screen_accounts(rem)
            else:
                self._screen_generate()
            self._toast("Deleted")

    def _show_loading(self, msg):
        f = tk.Frame(self.root, bg=C["bg"])
        f.place(relx=0.5, rely=0.5, anchor="center")
        tk.Label(f, text=msg, font=("Segoe UI", 13), bg=C["bg"], fg=C["muted"]).pack()
        self.root.update()

    # ══════════════════════════════════════════════════════════════════════
    #  GENERATE SCREEN
    # ══════════════════════════════════════════════════════════════════════
    def _screen_generate(self):
        self._clear()
        outer = tk.Frame(self.root, bg=C["bg"])
        outer.place(relx=0.5, rely=0.5, anchor="center")

        logo_row = tk.Frame(outer, bg=C["bg"])
        logo_row.pack(pady=(0, 24))
        tk.Label(logo_row, text="✦", font=("Segoe UI", 22), bg=C["bg"],
                 fg=C["accent"]).pack(side="left", padx=(0, 10))
        tk.Label(logo_row, text="VoidMail", font=("Segoe UI", 22, "bold"),
                 bg=C["bg"], fg=C["text"]).pack(side="left")

        card = tk.Frame(outer, bg=C["card"], highlightthickness=1,
                        highlightbackground=C["border"], padx=32, pady=28)
        card.pack()

        tk.Label(card, text="✦", font=("Segoe UI", 28), bg=C["card"],
                 fg=C["accent"]).pack(pady=(0, 12))
        tk.Label(card, text="Disposable Email", font=("Segoe UI", 14, "bold"),
                 bg=C["card"], fg=C["text"]).pack()
        tk.Label(card, text="One click, no signup.", font=("Segoe UI", 10),
                 bg=C["card"], fg=C["dim"]).pack(pady=(4, 20))

        self.gen_btn = self._make_btn(card, "Generate Inbox", self._start_generate, accent=True)
        self.gen_btn.pack(fill="x", ipady=4)

        tk.Label(outer, text="mail.tm", font=("Segoe UI", 8),
                 bg=C["bg"], fg=C["ghost"]).pack(pady=(12, 0))

    def _start_generate(self):
        self.gen_btn.config(text="Generating...", state="disabled")
        threading.Thread(target=self._generate_account, daemon=True).start()

    def _generate_account(self):
        try:
            r = requests.get(BASE + "/domains", timeout=10)
            domain = r.json()["hydra:member"][0]["domain"]
            email = f"{rand(10)}@{domain}"
            pw = gen_password()
            res = requests.post(BASE + "/accounts", json={"address": email, "password": pw}, timeout=10)
            if not res.ok:
                raise Exception("Creation failed")
            tok = requests.post(BASE + "/token", json={"address": email, "password": pw}, timeout=10)
            self.token = tok.json()["token"]
            self.account = {"email": email, "password": pw}
            add_account(email, pw)
            self.root.after(0, self._screen_app)
        except Exception as e:
            self.root.after(0, lambda: [self._toast(str(e)), self._screen_generate()])

    # ══════════════════════════════════════════════════════════════════════
    #  MAIN APP SCREEN
    # ══════════════════════════════════════════════════════════════════════
    def _screen_app(self):
        self._clear()
        self.messages = []
        self._seen_ids = set()
        self.selected = None
        self.pw_show = False
        self.polling = True

        # Navbar
        nav = tk.Frame(self.root, bg=C["bg2"], height=52)
        nav.pack(fill="x")
        nav.pack_propagate(False)

        left = tk.Frame(nav, bg=C["bg2"])
        left.place(x=16, rely=0.5, anchor="w")
        tk.Label(left, text="✦ VoidMail", font=("Segoe UI", 13, "bold"),
                 bg=C["bg2"], fg=C["accent"]).pack(side="left")
        self.badge = tk.Label(left, text="", font=("Segoe UI", 8, "bold"),
                              bg=C["accent"], fg="#fff", padx=6, pady=2)

        right = tk.Frame(nav, bg=C["bg2"])
        right.place(relx=1, x=-16, rely=0.5, anchor="e")
        tk.Label(right, text=self.account["email"], font=("Cascadia Code", 9),
                 bg=C["bg2"], fg=C["dim"]).pack(side="left", padx=(0, 10))
        self._make_btn(right, "Switch", self._switch_account, small=True).pack(side="left", padx=4)
        self._make_btn(right, "Delete", self._delete_current, danger=True, small=True).pack(side="left")

        tk.Frame(self.root, bg=C["border"], height=1).pack(fill="x")

        # Body
        body = tk.Frame(self.root, bg=C["bg"])
        body.pack(fill="both", expand=True)

        # Sidebar
        sidebar = tk.Frame(body, bg=C["bg2"], width=320)
        sidebar.pack(side="left", fill="y")
        sidebar.pack_propagate(False)

        # Account card
        ac = tk.Frame(sidebar, bg=C["bg3"], padx=14, pady=12)
        ac.pack(fill="x", padx=10, pady=(10, 0))

        hdr = tk.Frame(ac, bg=C["bg3"])
        hdr.pack(fill="x")
        tk.Label(hdr, text="ACTIVE", font=("Segoe UI", 8, "bold"),
                 bg=C["bg3"], fg=C["dim"]).pack(side="left")
        tk.Label(hdr, text="● Live", font=("Segoe UI", 8, "bold"),
                 bg=C["green_bg"], fg=C["green"], padx=6, pady=1).pack(side="right")

        tk.Label(ac, text=self.account["email"], font=("Cascadia Code", 9),
                 bg=C["bg3"], fg=C["muted"], anchor="w").pack(fill="x", pady=(6, 8))

        btns = tk.Frame(ac, bg=C["bg3"])
        btns.pack(fill="x")
        self._make_btn(btns, "Copy", self._copy_email, small=True).pack(side="left", padx=(0, 4))
        self._make_btn(btns, "Show PW", self._toggle_pw, small=True).pack(side="left")

        self.pw_frame = tk.Frame(ac, bg=C["bg3"])

        # Timer
        self.timer = tk.Canvas(ac, height=3, bg=C["border"], highlightthickness=0)
        self.timer.pack(fill="x", pady=(8, 2))
        self.timer_lbl = tk.Label(ac, text="polling...", font=("Segoe UI", 7),
                                  bg=C["bg3"], fg=C["ghost"])
        self.timer_lbl.pack(anchor="w")

        # Inbox header
        ih = tk.Frame(sidebar, bg=C["bg2"])
        ih.pack(fill="x", padx=14, pady=(14, 6))
        tk.Label(ih, text="Inbox", font=("Segoe UI", 11, "bold"),
                 bg=C["bg2"], fg=C["muted"]).pack(side="left")
        self.count_lbl = tk.Label(ih, text="0", font=("Segoe UI", 8),
                                  bg=C["bg3"], fg=C["dim"], padx=7, pady=2)
        self.count_lbl.pack(side="left", padx=6)
        self._make_btn(ih, "↻", self._manual_refresh, small=True).pack(side="right")

        tk.Frame(sidebar, bg=C["border"], height=1).pack(fill="x")

        # Message list
        lf = tk.Frame(sidebar, bg=C["bg2"])
        lf.pack(fill="both", expand=True)
        self.canvas = tk.Canvas(lf, bg=C["bg2"], highlightthickness=0)
        sb = tk.Scrollbar(lf, command=self.canvas.yview)
        self.canvas.configure(yscrollcommand=sb.set)
        sb.pack(side="right", fill="y")
        self.canvas.pack(fill="both", expand=True)
        self.inner = tk.Frame(self.canvas, bg=C["bg2"])
        self.canvas.create_window((0, 0), window=self.inner, anchor="nw")
        self.inner.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))

        # mouse wheel
        def mw(event):
            self.canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")
        self.canvas.bind("<Enter>", lambda e: self.canvas.bind_all("<MouseWheel>", mw))
        self.canvas.bind("<Leave>", lambda e: self.canvas.unbind_all("<MouseWheel>"))

        # Divider
        tk.Frame(body, bg=C["border"], width=1).pack(side="left", fill="y")

        # Viewer
        self.viewer = tk.Frame(body, bg=C["bg"])
        self.viewer.pack(side="left", fill="both", expand=True)
        self._empty_viewer()

        # Start polling
        self._poll_thread = threading.Thread(target=self._poll_loop, daemon=True)
        self._poll_thread.start()

    def _empty_viewer(self):
        for w in self.viewer.winfo_children():
            w.destroy()
        f = tk.Frame(self.viewer, bg=C["bg"])
        f.place(relx=0.5, rely=0.5, anchor="center")
        tk.Label(f, text="✉", font=("Segoe UI", 32), bg=C["bg"], fg=C["ghost"]).pack()
        tk.Label(f, text="Select a message", font=("Segoe UI", 10),
                 bg=C["bg"], fg=C["dim"]).pack(pady=(4, 0))

    # ── inbox rendering ──────────────────────────────────────────────────
    def _render_inbox(self):
        for w in self.inner.winfo_children():
            w.destroy()
        total = len(self.messages)
        self.count_lbl.config(text=str(total))
        unread = sum(1 for m in self.messages if not m.get("_read"))
        if unread:
            self.badge.config(text=f" {unread} ")
            self.badge.pack(side="left", padx=6)
        else:
            self.badge.pack_forget()
        if not self.messages:
            f = tk.Frame(self.inner, bg=C["bg2"])
            f.pack(pady=40)
            tk.Label(f, text="📭 waiting...", font=("Segoe UI", 10),
                     bg=C["bg2"], fg=C["ghost"]).pack()
            return
        for m in self.messages:
            self._msg_row(m)

    def _msg_row(self, msg):
        is_new = not msg.get("_read")
        sel = self.selected and self.selected.get("id") == msg.get("id")
        bg = C["selected"] if sel else C["bg2"]
        bar_color = C["accent"] if sel else (C["accent2"] if is_new else bg)

        row = tk.Frame(self.inner, bg=bg, cursor="hand2")
        row.pack(fill="x", pady=1)
        tk.Frame(row, bg=bar_color, width=3).pack(side="left", fill="y")
        inner = tk.Frame(row, bg=bg, padx=12, pady=10)
        inner.pack(fill="x", expand=True)

        sender = msg.get("from", {}).get("address", "unknown")
        subj = msg.get("subject", "(no subject)")
        ago = time_ago(msg.get("createdAt", ""))

        tk.Label(inner, text=sender[:28], font=("Segoe UI", 10, "bold" if is_new else "normal"),
                 bg=bg, fg=C["text"] if is_new else C["muted"], anchor="w").pack(fill="x")
        bot = tk.Frame(inner, bg=bg)
        bot.pack(fill="x")
        tk.Label(bot, text=subj[:35], font=("Segoe UI", 9), bg=bg,
                 fg=C["muted"] if is_new else C["dim"], anchor="w").pack(side="left")
        tk.Label(bot, text=ago, font=("Segoe UI", 8), bg=bg, fg=C["ghost"]).pack(side="right")

        tk.Frame(self.inner, bg=C["border"], height=1).pack(fill="x")

        for w in (row, inner, bot):
            w.bind("<Button-1>", lambda e, m=msg: self._open_msg(m))

    def _open_msg(self, msg):
        self.selected = msg
        msg["_read"] = True
        threading.Thread(target=self._load_msg, args=(msg,), daemon=True).start()
        self._render_inbox()

    def _load_msg(self, msg):
        body = msg.get("_body", "")
        if not msg.get("_loaded") and self.token:
            try:
                r = requests.get(f"{BASE}/messages/{msg['id']}",
                                 headers={"Authorization": f"Bearer {self.token}"}, timeout=10)
                d = r.json()
                body = d.get("text", d.get("intro", ""))
                msg["_body"] = body
                msg["_loaded"] = True
            except:
                pass
        self.root.after(0, lambda: self._show_msg(msg, body))

    def _show_msg(self, msg, body):
        for w in self.viewer.winfo_children():
            w.destroy()

        h = tk.Frame(self.viewer, bg=C["bg"], padx=20, pady=16)
        h.pack(fill="x")
        tk.Label(h, text=msg.get("subject", "(no subject)"),
                 font=("Segoe UI", 14, "bold"), bg=C["bg"], fg=C["text"],
                 wraplength=500, anchor="w", justify="left").pack(fill="x", pady=(0, 10))

        meta = tk.Frame(h, bg=C["bg"])
        meta.pack(fill="x")
        for lab, val in [("From", msg.get("from", {}).get("address", "?")),
                         ("To", self.account["email"]),
                         ("Date", fmt_date(msg.get("createdAt", "")))]:
            col = tk.Frame(meta, bg=C["bg"])
            col.pack(side="left", padx=(0, 20))
            tk.Label(col, text=lab, font=("Segoe UI", 7, "bold"),
                     bg=C["bg"], fg=C["dim"]).pack(anchor="w")
            tk.Label(col, text=val, font=("Segoe UI", 9),
                     bg=C["bg"], fg=C["muted"]).pack(anchor="w")

        tk.Frame(self.viewer, bg=C["border"], height=1).pack(fill="x", padx=20)

        bf = tk.Frame(self.viewer, bg=C["bg"])
        bf.pack(fill="both", expand=True, padx=20, pady=12)
        txt = tk.Text(bf, bg=C["bg"], fg=C["muted"], font=("Segoe UI", 11),
                      wrap="word", bd=0, highlightthickness=0, padx=4, pady=4)
        sb2 = tk.Scrollbar(bf, command=txt.yview)
        txt.configure(yscrollcommand=sb2.set)
        sb2.pack(side="right", fill="y")
        txt.pack(fill="both", expand=True)
        txt.insert("end", body or "(empty)")
        txt.configure(state="disabled")

    # ── polling ───────────────────────────────────────────────────────────
    def _poll_loop(self):
        total = 80
        step = 0
        self._fetch()
        while self.polling:
            time.sleep(0.1)
            step += 1
            pct = step / total
            self.root.after(0, self._update_timer, pct)
            if step >= total:
                step = 0
                self._fetch()

    def _fetch(self):
        try:
            r = requests.get(BASE + "/messages",
                             headers={"Authorization": f"Bearer {self.token}"}, timeout=10)
            if r.ok:
                data = r.json().get("hydra:member", [])
                new = [m for m in data if m["id"] not in self._seen_ids]
                if new:
                    for m in new:
                        self._seen_ids.add(m["id"])
                        self.messages.insert(0, m)
                    self.root.after(0, self._render_inbox)
                    self.root.after(0, lambda: self._toast(f"{len(new)} new message{'s' if len(new)>1 else ''}"))
        except:
            pass

    def _manual_refresh(self):
        threading.Thread(target=self._fetch, daemon=True).start()

    def _update_timer(self, pct):
        w = self.timer.winfo_width()
        if w > 2:
            self.timer.delete("all")
            self.timer.create_rectangle(0, 0, int(w * pct), 3, fill=C["accent"], outline="")

    # ── actions ───────────────────────────────────────────────────────────
    def _copy_email(self):
        self.root.clipboard_clear()
        self.root.clipboard_append(self.account["email"])
        self._toast("Copied!")

    def _toggle_pw(self):
        self.pw_show = not self.pw_show
        if self.pw_show:
            self.pw_frame.pack(fill="x", pady=(8, 0))
            for w in self.pw_frame.winfo_children():
                w.destroy()
            inner = tk.Frame(self.pw_frame, bg="#0a0a0f")
            inner.pack(fill="x")
            tk.Label(inner, text=self.account["password"], font=("Cascadia Code", 9),
                     bg="#0a0a0f", fg=C["dim"], anchor="w", padx=8, pady=6).pack(side="left", fill="x", expand=True)
            self._make_btn(inner, "Copy", self._copy_pw, small=True).pack(side="right", padx=4)
        else:
            self.pw_frame.pack_forget()

    def _copy_pw(self):
        self.root.clipboard_clear()
        self.root.clipboard_append(self.account["password"])
        self._toast("Password copied!")

    def _switch_account(self):
        self.polling = False
        self._screen_accounts(load_accounts())

    def _delete_current(self):
        if messagebox.askyesno("Delete", "Delete this inbox?"):
            remove_account(self.account["email"])
            self.polling = False
            rem = load_accounts()
            if rem:
                self._screen_accounts(rem)
            else:
                self._screen_generate()
            self._toast("Deleted")


if __name__ == "__main__":
    VoidMail()
