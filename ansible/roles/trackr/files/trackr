#!/usr/bin/env python3
"""TrackR - le suivi des expéditions de LogiNord."""

import base64
import hashlib
import hmac
import html
import json
import logging
import os
import secrets
import sqlite3
import sys
import time
from datetime import date, datetime
from http.cookies import SimpleCookie
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import parse_qs, urlparse

VERSION = "1.0"
log = logging.getLogger("trackr")


def load_config(path):
    if not path or not os.path.isfile(path):
        return
    with open(path, encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith("#") or "=" not in line:
                continue
            key, value = line.split("=", 1)
            os.environ.setdefault(key.strip(), value.strip().strip('"').strip("'"))


def conf(key, default=""):
    return os.environ.get("TRACKR_" + key, default).strip()


class Database:
    def __init__(self, url):
        self.url = url
        if url.startswith("postgresql://") or url.startswith("postgres://"):
            import psycopg2
            self.driver = psycopg2
            self.kind = "postgresql"
        else:
            self.driver = sqlite3
            self.kind = "sqlite"
            self.path = url.removeprefix("sqlite://") or "trackr.db"
            folder = os.path.dirname(self.path)
            if folder:
                os.makedirs(folder, exist_ok=True)

    def connect(self):
        if self.kind == "postgresql":
            return self.driver.connect(self.url)
        cnx = sqlite3.connect(self.path, timeout=10)
        cnx.execute("PRAGMA journal_mode=WAL")
        return cnx

    def run(self, sql, params=(), fetch=False):
        if self.kind == "postgresql":
            sql = sql.replace("?", "%s")
        cnx = self.connect()
        try:
            cur = cnx.cursor()
            cur.execute(sql, params)
            rows = cur.fetchall() if fetch else None
            cnx.commit()
            return rows
        finally:
            cnx.close()

    def init(self):
        auto_id = "SERIAL PRIMARY KEY" if self.kind == "postgresql" else "INTEGER PRIMARY KEY AUTOINCREMENT"
        self.run(f"""CREATE TABLE IF NOT EXISTS expeditions (
            id {auto_id},
            reference TEXT NOT NULL,
            client TEXT NOT NULL,
            origine TEXT NOT NULL,
            destination TEXT NOT NULL,
            quai TEXT NOT NULL DEFAULT '',
            prevue TEXT NOT NULL,
            arrivee TEXT,
            cree_par TEXT NOT NULL,
            cree_le TEXT NOT NULL)""")

    def demo(self):
        if self.run("SELECT COUNT(*) FROM expeditions", fetch=True)[0][0]:
            return
        today = date.today().toordinal()
        rows = [
            ("LN-1001", "Brasserie du Port", "Dunkerque", "Lille", "Q2", today - 2, today - 2),
            ("LN-1002", "Textiles Roubaix", "Lille", "Dunkerque", "Q1", today - 1, None),
            ("LN-1003", "Sucrerie Flandre", "Dunkerque", "Arras", "Q4", today, None),
            ("LN-1004", "Acier Nord", "Dunkerque", "Valenciennes", "Q3", today + 1, None),
            ("LN-1005", "Papeterie Opale", "Calais", "Dunkerque", "Q2", today + 3, None),
        ]
        for ref, client, orig, dest, quai, prevue, arrivee in rows:
            self.run(
                "INSERT INTO expeditions (reference, client, origine, destination, quai, prevue, arrivee, cree_par, cree_le)"
                " VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
                (ref, client, orig, dest, quai, date.fromordinal(prevue).isoformat(),
                 date.fromordinal(arrivee).isoformat() if arrivee else None, "demo", now()))
        log.info("Données de démonstration ajoutées")


def now():
    return datetime.now().strftime("%Y-%m-%d %H:%M")


class LocalAuth:
    name = "local"

    def __init__(self, users):
        self.users = {}
        for item in users.split(","):
            if ":" in item:
                user, password = item.split(":", 1)
                self.users[user.strip()] = password.strip()
        if not self.users:
            raise SystemExit("TRACKR_AUTH=local mais TRACKR_LOCAL_USERS est vide (format : alice:motdepasse,bob:motdepasse)")

    def login(self, user, password):
        expected = self.users.get(user)
        if expected and hmac.compare_digest(expected, password):
            return user, None
        return None, "identifiant ou mot de passe incorrect"


AD_ERRORS = {
    "525": "compte inconnu",
    "52e": "mauvais mot de passe",
    "530": "connexion interdite à cette heure",
    "531": "connexion interdite depuis ce poste",
    "532": "mot de passe expiré",
    "533": "compte désactivé",
    "701": "compte expiré",
    "773": "mot de passe à changer",
    "775": "compte verrouillé",
}


class ADAuth:
    name = "ad"

    def __init__(self):
        try:
            import ldap3
        except ImportError:
            raise SystemExit("TRACKR_AUTH=ad demande le paquet python3-ldap3 (apt install python3-ldap3)")
        self.ldap3 = ldap3
        self.domain = conf("AD_DOMAIN")
        if not self.domain:
            raise SystemExit("TRACKR_AUTH=ad mais TRACKR_AD_DOMAIN est vide (exemple : loginord.internal)")
        self.base_dn = conf("AD_BASE_DN") or ",".join("DC=" + part for part in self.domain.split("."))
        self.group = conf("AD_GROUP")
        self.timeout = int(conf("AD_TIMEOUT", "5"))
        self.tls_mode = conf("AD_TLS", "no").lower()
        tls = None
        if self.tls_mode in ("starttls", "ldaps"):
            import ssl
            ca = conf("AD_CA_FILE")
            if not ca:
                log.warning("TRACKR_AD_CA_FILE vide : le certificat de l'annuaire n'est pas vérifié")
            tls = ldap3.Tls(validate=ssl.CERT_REQUIRED if ca else ssl.CERT_NONE, ca_certs_file=ca or None)
        servers = [s.strip() for s in (conf("AD_SERVERS") or self.domain).split(",") if s.strip()]
        self.pool = ldap3.ServerPool(
            [ldap3.Server(s, use_ssl=self.tls_mode == "ldaps", tls=tls, connect_timeout=self.timeout) for s in servers],
            ldap3.FIRST, active=1, exhaust=60)

    def login(self, user, password):
        ldap3 = self.ldap3
        escape = ldap3.utils.conv.escape_filter_chars
        user = user.split("@")[0].split("\\")[-1]
        if not user or not password or not all(c.isalnum() or c in "._-" for c in user):
            return None, "identifiant ou mot de passe incorrect"
        try:
            cnx = ldap3.Connection(self.pool, user=f"{user}@{self.domain}", password=password,
                                   receive_timeout=self.timeout, raise_exceptions=False)
            if self.tls_mode == "starttls":
                cnx.open()
                cnx.start_tls()
            if not cnx.bind():
                detail = cnx.result.get("message") or cnx.result.get("description") or ""
                reason = next((v for k, v in AD_ERRORS.items() if f"data {k}," in detail), detail)
                log.info("Connexion refusée pour %s : %s", user, reason)
                return None, "identifiant ou mot de passe incorrect"
            ldap_filter = f"(&(objectCategory=person)(sAMAccountName={user}))"
            if self.group:
                cnx.search(self.base_dn, f"(&(objectCategory=group)(cn={escape(self.group)}))", attributes=["cn"])
                if not cnx.entries:
                    log.warning("Le groupe %s n'existe pas dans %s", self.group, self.base_dn)
                    return None, "accès à TrackR non configuré, prévenez l'informatique"
                group_dn = escape(cnx.entries[0].entry_dn)
                ldap_filter = f"(&(objectCategory=person)(sAMAccountName={user})(memberOf:1.2.840.113556.1.4.1941:={group_dn}))"
            cnx.search(self.base_dn, ldap_filter, attributes=["displayName"])
            found = cnx.entries[0] if cnx.entries else None
            cnx.unbind()
        except ldap3.core.exceptions.LDAPException as e:
            log.error("Annuaire injoignable : %s", e)
            return None, "l'annuaire ne répond pas, réessayez dans un instant"
        if not found:
            log.info("Connexion refusée pour %s : hors du groupe %s", user, self.group)
            return None, "votre compte n'a pas accès à TrackR"
        return found.displayName.value or user, None


class Sessions:
    def __init__(self, secret, hours):
        self.key = secret.encode()
        self.ttl = hours * 3600

    def make(self, user):
        payload = base64.urlsafe_b64encode(json.dumps([user, int(time.time()) + self.ttl]).encode()).decode()
        sig = hmac.new(self.key, payload.encode(), hashlib.sha256).hexdigest()
        return f"{payload}.{sig}"

    def check(self, token):
        try:
            payload, sig = token.rsplit(".", 1)
            if not hmac.compare_digest(sig, hmac.new(self.key, payload.encode(), hashlib.sha256).hexdigest()):
                return None
            user, expires = json.loads(base64.urlsafe_b64decode(payload))
            return user if expires > time.time() else None
        except Exception:
            return None


STATUTS = {"encours": "En cours", "retard": "En retard", "arrivee": "Arrivée"}

STYLE = """
body{font-family:system-ui,sans-serif;margin:0;background:#f4f5f7;color:#1d2433}
header{background:#14365d;color:#fff;padding:12px 24px;display:flex;justify-content:space-between;align-items:center}
header a{color:#fff}
main{max-width:1100px;margin:24px auto;padding:0 16px}
table{width:100%;border-collapse:collapse;background:#fff}
th,td{padding:8px 10px;border-bottom:1px solid #e3e6eb;text-align:left}
th{background:#eef1f5}
.retard{color:#b42318;font-weight:600}.arrivee{color:#067647}.encours{color:#175cd3}
form.inline{display:inline}
.box{background:#fff;padding:16px;margin:16px 0;border:1px solid #e3e6eb}
input,button{padding:6px 8px;margin:2px}
.erreur{background:#fef3f2;border:1px solid #fda29b;padding:8px;margin-bottom:12px}
nav a{margin-right:12px}
"""


def page(title, body, user=None):
    who = f'{html.escape(user)} · <a href="/logout">se déconnecter</a>' if user else ""
    return f"""<!doctype html><html lang="fr"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"><title>{html.escape(title)} - TrackR</title>
<style>{STYLE}</style></head><body><header><strong>TrackR</strong><span>{who}</span></header>
<main>{body}</main></body></html>"""


class App:
    def __init__(self):
        self.db = Database(conf("DB", "sqlite:///var/lib/trackr/trackr.db"))
        self.db.init()
        if conf("DEMO", "no").lower() in ("1", "yes", "oui", "true"):
            self.db.demo()
        mode = conf("AUTH", "local").lower()
        self.auth = ADAuth() if mode == "ad" else LocalAuth(conf("LOCAL_USERS"))
        secret = conf("SECRET")
        if not secret:
            secret = secrets.token_hex(32)
            log.warning("TRACKR_SECRET vide : les sessions seront perdues à chaque redémarrage")
        self.sessions = Sessions(secret, int(conf("SESSION_HOURS", "10")))
        self.secure_cookie = conf("COOKIE_SECURE", "no").lower() in ("1", "yes", "oui", "true")


class Handler(BaseHTTPRequestHandler):
    server_version = "TrackR/" + VERSION
    app = None

    def log_message(self, fmt, *args):
        log.debug("%s %s", self.client_address[0], fmt % args)

    def client_ip(self):
        return self.headers.get("X-Real-IP") or self.client_address[0]

    def send(self, code, body, content_type="text/html; charset=utf-8", headers=None):
        data = body.encode()
        self.send_response(code)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(data)))
        for k, v in (headers or {}).items():
            self.send_header(k, v)
        self.end_headers()
        self.wfile.write(data)

    def redirect(self, where, cookie=None):
        self.send_response(303)
        self.send_header("Location", where)
        if cookie is not None:
            self.send_header("Set-Cookie", cookie)
        self.send_header("Content-Length", "0")
        self.end_headers()

    def user(self):
        cookie = SimpleCookie(self.headers.get("Cookie", ""))
        token = cookie.get("trackr")
        return self.app.sessions.check(token.value) if token else None

    def form(self):
        length = int(self.headers.get("Content-Length") or 0)
        data = parse_qs(self.rfile.read(min(length, 65536)).decode(errors="replace"))
        return {k: v[0].strip() for k, v in data.items()}

    def do_GET(self):
        url = urlparse(self.path)
        if url.path == "/health":
            return self.health()
        if url.path == "/login":
            return self.login_page()
        if url.path == "/logout":
            return self.redirect("/login", "trackr=; Path=/; Max-Age=0")
        user = self.user()
        if not user:
            return self.redirect("/login")
        if url.path == "/":
            return self.index(user, parse_qs(url.query).get("vue", ["toutes"])[0])
        self.send(404, page("Introuvable", "<p>Cette page n'existe pas.</p>", user))

    def do_POST(self):
        url = urlparse(self.path)
        if url.path == "/login":
            return self.login_post()
        user = self.user()
        if not user:
            return self.redirect("/login")
        if url.path == "/expeditions":
            return self.create(user)
        parts = url.path.strip("/").split("/")
        if len(parts) == 3 and parts[0] == "expeditions" and parts[1].isdigit() and parts[2] == "arrivee":
            self.app.db.run("UPDATE expeditions SET arrivee = ? WHERE id = ? AND arrivee IS NULL",
                            (date.today().isoformat(), int(parts[1])))
            log.info("%s a marqué l'expédition %s comme arrivée", user, parts[1])
            return self.redirect("/")
        self.send(404, page("Introuvable", "<p>Cette page n'existe pas.</p>", user))

    def health(self):
        try:
            self.app.db.run("SELECT 1", fetch=True)
            self.send(200, json.dumps({"status": "ok", "version": VERSION, "auth": self.app.auth.name}),
                      "application/json")
        except Exception as e:
            log.error("Base de données injoignable : %s", e)
            self.send(503, json.dumps({"status": "erreur", "detail": "base de données"}), "application/json")

    def login_page(self, error=None):
        msg = f'<div class="erreur">{html.escape(error)}</div>' if error else ""
        hint = "votre compte Windows" if self.app.auth.name == "ad" else "votre compte TrackR"
        body = f"""<div class="box" style="max-width:360px;margin:60px auto"><h2>Connexion</h2>{msg}
<form method="post" action="/login"><p>Identifiant ({hint})<br><input name="user" autofocus required></p>
<p>Mot de passe<br><input name="password" type="password" required></p><button>Se connecter</button></form></div>"""
        self.send(200 if not error else 401, page("Connexion", body))

    def login_post(self):
        f = self.form()
        name, error = self.app.auth.login(f.get("user", ""), f.get("password", ""))
        if not name:
            log.info("Échec de connexion : %s depuis %s", f.get("user", ""), self.client_ip())
            return self.login_page(error)
        log.info("Connexion réussie : %s depuis %s", name, self.client_ip())
        cookie = f"trackr={self.app.sessions.make(name)}; Path=/; HttpOnly; SameSite=Strict"
        if self.app.secure_cookie:
            cookie += "; Secure"
        self.redirect("/", cookie)

    def index(self, user, vue):
        rows = self.app.db.run(
            "SELECT id, reference, client, origine, destination, quai, prevue, arrivee FROM expeditions"
            " ORDER BY arrivee IS NOT NULL, prevue, id", fetch=True)
        today = date.today().isoformat()

        def statut(r):
            if r[7]:
                return "arrivee"
            return "retard" if r[6] < today else "encours"

        counts = {"toutes": len(rows)}
        for r in rows:
            counts[statut(r)] = counts.get(statut(r), 0) + 1
        if vue in ("encours", "retard", "arrivee"):
            rows = [r for r in rows if statut(r) == vue]
        labels = {"toutes": "Toutes", "encours": "En cours", "retard": "En retard", "arrivee": "Arrivées"}
        nav = " ".join(
            f'<a href="/?vue={k}">{"<b>" if k == vue else ""}{v} ({counts.get(k, 0)}){"</b>" if k == vue else ""}</a>'
            for k, v in labels.items())
        lines = []
        for r in rows:
            s = statut(r)
            action = r[7] or (f'<form class="inline" method="post" action="/expeditions/{r[0]}/arrivee">'
                              f'<button>Arrivée</button></form>')
            cells = "".join(f"<td>{html.escape(str(v))}</td>" for v in r[1:7])
            lines.append(f'<tr>{cells}<td class="{s}">{STATUTS[s]}</td><td>{action}</td></tr>')
        table = "".join(lines) or '<tr><td colspan="8">Aucune expédition.</td></tr>'
        body = f"""<nav>{nav}</nav>
<table><tr><th>Référence</th><th>Client</th><th>Origine</th><th>Destination</th><th>Quai</th>
<th>Prévue le</th><th>Statut</th><th></th></tr>{table}</table>
<div class="box"><h3>Nouvelle expédition</h3><form method="post" action="/expeditions">
<input name="reference" placeholder="Référence" required><input name="client" placeholder="Client" required>
<input name="origine" placeholder="Origine" required><input name="destination" placeholder="Destination" required>
<input name="quai" placeholder="Quai"><input name="prevue" type="date" value="{today}" required>
<button>Ajouter</button></form></div>"""
        self.send(200, page("Expéditions", body, user))

    def create(self, user):
        f = self.form()
        fields = [f.get(k, "")[:100] for k in ("reference", "client", "origine", "destination", "quai")]
        try:
            prevue = date.fromisoformat(f.get("prevue", "")).isoformat()
        except ValueError:
            prevue = ""
        if not all(fields[:4]) or not prevue:
            return self.send(400, page("Erreur", '<div class="erreur">Champs manquants.</div><a href="/">Retour</a>', user))
        self.app.db.run(
            "INSERT INTO expeditions (reference, client, origine, destination, quai, prevue, cree_par, cree_le)"
            " VALUES (?, ?, ?, ?, ?, ?, ?, ?)", (*fields, prevue, user, now()))
        log.info("%s a créé l'expédition %s", user, fields[0])
        self.redirect("/")


def main():
    args = sys.argv[1:]
    if args and args[0] in ("-h", "--help"):
        print("usage : trackr [fichier.env]   (par défaut /etc/trackr/trackr.env)")
        return
    load_config(args[0] if args else os.environ.get("TRACKR_CONFIG", "/etc/trackr/trackr.env"))
    logging.basicConfig(level=conf("LOG_LEVEL", "INFO").upper(), format="%(levelname)s %(message)s")
    Handler.app = App()
    listen, port = conf("LISTEN", "0.0.0.0"), int(conf("PORT", "8080"))
    server = ThreadingHTTPServer((listen, port), Handler)
    log.info("TrackR %s écoute sur http://%s:%s (base %s, connexion %s)",
             VERSION, listen, port, Handler.app.db.kind, Handler.app.auth.name)
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        pass


if __name__ == "__main__":
    main()
