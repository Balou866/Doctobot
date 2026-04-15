# Doctolib Checker — Notifications Telegram

Script de surveillance Doctolib qui vérifie périodiquement la disponibilité de créneaux médicaux et envoie une notification Telegram à chaque changement (nouveau créneau apparu ou disparu).

**Fonctionnalités :**
- Surveillance de **plusieurs médecins** simultanément
- Notification uniquement sur **changement** (pas de spam si le même créneau reste disponible)
- Détail complet des créneaux disponibles (date + heure) dans la notification
- Gestion des **remplaçants** : choix d'inclure ou non leurs créneaux, avec indication du nom dans la notification
- Déploiement 100% via Portainer (docker-compose), aucune ligne de commande

---

## Prérequis

- Docker / Portainer installé sur ton serveur
- Un compte Telegram

---

## Configuration

### 1. Créer un bot Telegram et récupérer les credentials

**Token du bot :**
1. Ouvre Telegram et cherche **@BotFather**
2. Envoie `/newbot` et suis les instructions
3. BotFather te donne un token : `123456789:ABCdefGHIjklMNOPQRSTuvwxyz` → c'est ton `TELEGRAM_BOT_TOKEN`

**Chat ID :**
1. Envoie n'importe quel message à ton bot
2. Ouvre cette URL dans ton navigateur (remplace `TON_TOKEN`) :
   ```
   https://api.telegram.org/botTON_TOKEN/getUpdates
   ```
3. Dans la réponse JSON, cherche `"chat":{"id":` → le nombre qui suit est ton `TELEGRAM_CHAT_ID`

---

### 2. Récupérer les URLs Doctolib

#### URL API (obligatoire — `DOCTOLIB_URLS`)

1. Va sur la page du médecin sur Doctolib et connecte-toi. Va sur la page de réservation (page "Choisissez la date de consultation").
2. Ouvre les **Outils de développement** (`F12`) > onglet **Réseau**
3. Filtre sur `availabilities`
4. Clique sur le **dernier GET** `availabilities.json` qui apparaît
5. **Copie l'URL complète** et colle-la dans `DOCTOLIB_URLS`

> **Note :** Si l'URL contient `master_patient_signed_id`, il est automatiquement supprimé — les appels API se font sans identifiant de compte.

#### URL de réservation (optionnelle — `DOCTOLIB_BOOKING_URLS`)

Si tu veux un **lien cliquable** dans les notifications Telegram pour accéder directement à la page de prise de rendez-vous :

1. Depuis la même page de réservation Doctolib, **copie l'URL de la barre d'adresse du navigateur**
2. Colle-la dans `DOCTOLIB_BOOKING_URLS` (même ordre que `DOCTOLIB_URLS`)

> Les paramètres `masterPatientSignedId` et `master_patient_id` sont automatiquement supprimés de ce lien.

---

### 3. Déployer dans Portainer

1. **Stacks → Add stack → Web editor**
2. Colle le contenu du docker-compose ci-dessous
3. Remplis les variables (URLs, noms, Telegram, date limite)
4. **Deploy the stack**

---

## Docker Compose

```yaml
version: "3.8"

services:
  doctolib-checker:
    image: python:3.11-slim
    restart: unless-stopped
    environment:

      # ── MÉDECINS À SURVEILLER ────────────────────────────────────────────
      # URL API (onglet Réseau > dernier GET availabilities.json) — une par ligne
      DOCTOLIB_URLS: |
        https://www.doctolib.fr/availabilities.json?visit_motive_ids=XXX&agenda_ids=XXX&practice_ids=XXX&telehealth=false&start_date=2026-04-15&limit=5

      # Nom affiché dans les notifications (un nom par ligne, même ordre que les URLs)
      DOCTOLIB_NAMES: |
        Dr. Nom Prénom

      # URL de réservation pour lien cliquable dans les notifs (optionnel)
      # URL de la barre d'adresse Doctolib — une par ligne, même ordre que DOCTOLIB_URLS
      # Les paramètres masterPatientSignedId / master_patient_id sont supprimés automatiquement
      # Laisser vide = pas de lien dans les notifications
      DOCTOLIB_BOOKING_URLS: |
        https://www.doctolib.fr/[specialite]/[ville]/[medecin]/booking/availabilities?...

      # ── DATES ET FENÊTRE ─────────────────────────────────────────────────
      # Ne notifier que si un créneau est disponible AVANT cette date
      LIMIT_DATE: "2026-06-10"
      # Fenêtre de recherche en jours par appel API (max 15, imposé par Doctolib)
      LIMIT: "15"

      # ── VÉRIFICATION ─────────────────────────────────────────────────────
      # Intervalle entre chaque vérification en secondes (300 = 5 minutes)
      INTERVAL_SECONDS: "300"
      # Notification quotidienne "le script tourne toujours"
      ALIVE_CHECK: "true"
      ALIVE_CHECK_HOUR: "9"

      # ── FILTRE CRÉNEAUX (optionnel) ───────────────────────────────────────
      # Laisser vide = tous les créneaux notifiés (comportement par défaut)
      # Format : jour:HH:MM-HH:MM  (une plage par ligne)
      # Jours : lundi mardi mercredi jeudi vendredi samedi dimanche
      SLOT_FILTER: |
        mercredi:13:00-19:00

      # ── REMPLAÇANTS ───────────────────────────────────────────────────────
      # true  = notifier aussi les créneaux des remplaçants (nom indiqué dans la notif)
      # false = uniquement les créneaux du praticien original
      INCLUDE_SUBSTITUTES: "true"

      # ── TELEGRAM ─────────────────────────────────────────────────────────
      TELEGRAM_BOT_TOKEN: "VOTRE_BOT_TOKEN"
      TELEGRAM_CHAT_ID: "VOTRE_CHAT_ID"

      PYTHONUNBUFFERED: "1"

      # ── SCRIPTS (ne pas modifier) ─────────────────────────────────────────
      CONFIG_SCRIPT: |
        import os, yaml, datetime, urllib.parse
        today = datetime.date.today().strftime("%Y-%m-%d")
        def normalize_api_url(url):
            url = url.strip()
            if '{start_date}' in url:
                return url
            parsed = urllib.parse.urlparse(url)
            params = urllib.parse.parse_qs(parsed.query, keep_blank_values=True)
            clean = {k: v[0] for k, v in params.items() if k not in ('master_patient_signed_id', 'start_date', 'limit')}
            base = urllib.parse.urlunparse((parsed.scheme, parsed.netloc, parsed.path, '', urllib.parse.urlencode(clean), ''))
            return base + '&start_date={start_date}&limit={limit}'
        def clean_booking_url(url):
            url = url.strip()
            parsed = urllib.parse.urlparse(url)
            params = urllib.parse.parse_qs(parsed.query, keep_blank_values=True)
            clean = {k: v[0] for k, v in params.items() if k not in ('masterPatientSignedId', 'master_patient_id', 'master_patient_signed_id')}
            new_query = urllib.parse.urlencode(clean) if clean else ''
            return urllib.parse.urlunparse((parsed.scheme, parsed.netloc, parsed.path, '', new_query, ''))
        raw_urls = [u for u in os.environ.get("DOCTOLIB_URLS", "").strip().splitlines() if u.strip() and not u.strip().startswith('#')]
        names = [n.strip() for n in os.environ.get("DOCTOLIB_NAMES", "").strip().splitlines() if n.strip() and not n.strip().startswith('#')]
        booking_raw = [u for u in os.environ.get("DOCTOLIB_BOOKING_URLS", "").strip().splitlines() if u.strip() and not u.strip().startswith('#')]
        while len(names) < len(raw_urls):
            names.append("Medecin " + str(len(names) + 1))
        entries = []
        for i, raw_url in enumerate(raw_urls):
            booking_url = clean_booking_url(booking_raw[i]) if i < len(booking_raw) else None
            entries.append({"name": names[i], "url": normalize_api_url(raw_url), "booking_url": booking_url})
        slot_filter_raw = os.environ.get("SLOT_FILTER", "").strip()
        slot_filter = []
        days_fr = {"lundi":0,"mardi":1,"mercredi":2,"jeudi":3,"vendredi":4,"samedi":5,"dimanche":6}
        for line in slot_filter_raw.splitlines():
            line = line.strip()
            if not line:
                continue
            day_part, time_part = line.split(":", 1)
            start_str, end_str = time_part.split("-")
            sh, sm = map(int, start_str.split(":"))
            eh, em = map(int, end_str.split(":"))
            slot_filter.append({"day": days_fr.get(day_part.lower(), -1), "start": sh*60+sm, "end": eh*60+em})
        cfg = {
            "run_in_loop": True,
            "interval_in_seconds": int(os.environ.get("INTERVAL_SECONDS", 300)),
            "alive_check": os.environ.get("ALIVE_CHECK", "true").lower() == "true",
            "hour_of_alive_check": int(os.environ.get("ALIVE_CHECK_HOUR", 9)),
            "limit_date": os.environ.get("LIMIT_DATE"),
            "start_date": today,
            "limit": int(os.environ.get("LIMIT", 15)),
            "entries": entries,
            "slot_filter": slot_filter,
            "include_substitutes": os.environ.get("INCLUDE_SUBSTITUTES", "true").lower() == "true",
            "telegram": {
                "bot_token": os.environ.get("TELEGRAM_BOT_TOKEN"),
                "chat_id": os.environ.get("TELEGRAM_CHAT_ID"),
            }
        }
        yaml.dump(cfg, open("/app/config.yaml", "w"))
        print("Config OK - " + str(len(entries)) + " medecin(s), " + str(len(slot_filter)) + " filtre(s) horaire(s), remplacants: " + str(cfg["include_substitutes"]))

      CHECKER_SCRIPT: |
        import datetime, json, pathlib, time, urllib.parse, urllib.request, yaml

        with open(str(pathlib.Path(__file__).parent.resolve()) + "/config.yaml", "r") as f:
            config = yaml.safe_load(f)

        limit_date = config["limit_date"]
        limit = config["limit"]
        limit_dt = datetime.datetime.strptime(limit_date, "%Y-%m-%d")
        slot_filter = config.get("slot_filter", [])
        include_substitutes = config.get("include_substitutes", True)

        def now():
            return datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

        def slot_matches_filter(slot_str):
            if not slot_filter:
                return True
            dt = datetime.datetime.strptime(slot_str[:-10], "%Y-%m-%dT%H:%M:%S")
            wd = dt.weekday()
            minutes = dt.hour * 60 + dt.minute
            for f in slot_filter:
                if f["day"] == wd and f["start"] <= minutes <= f["end"]:
                    return True
            return False

        def send_telegram(message):
            token = config["telegram"]["bot_token"]
            chat_id = str(config["telegram"]["chat_id"])
            api_url = "https://api.telegram.org/bot" + token + "/sendMessage"
            data = urllib.parse.urlencode({"chat_id": chat_id, "text": message, "parse_mode": "HTML"}).encode("utf-8")
            try:
                urllib.request.urlopen(urllib.request.Request(api_url, data=data))
            except Exception as e:
                print(now() + " - Erreur Telegram : " + str(e))

        def fetch(url_template, start_date):
            u = url_template.format_map({"start_date": start_date, "limit": limit})
            req = urllib.request.Request(u, headers={"User-Agent": "Magic Browser"})
            return json.loads(urllib.request.urlopen(req).read())

        def parse_availability_slots(availability):
            slots = availability.get("slots", [])
            substitution = availability.get("substitution") or {}
            substitute_map = {}
            for sub_name, sub_info in substitution.items():
                for s in sub_info.get("slots", []):
                    substitute_map[s] = sub_name
            result = []
            for slot in slots:
                sub_name = substitute_map.get(slot)
                if sub_name is None:
                    result.append((slot, None))
                elif include_substitutes:
                    result.append((slot, sub_name))
            return result

        def get_all_slots(url_template):
            data = fetch(url_template, config["start_date"])
            slots = []
            if data["total"] > 0:
                for availability in data["availabilities"]:
                    if datetime.datetime.strptime(availability["date"][:10], "%Y-%m-%d") > limit_dt:
                        break
                    slots.extend(parse_availability_slots(availability))
            elif data.get("next_slot") and datetime.datetime.strptime(data["next_slot"][:10], "%Y-%m-%d") <= limit_dt:
                data2 = fetch(url_template, data["next_slot"][:10])
                for availability in data2["availabilities"]:
                    if datetime.datetime.strptime(availability["date"][:10], "%Y-%m-%d") > limit_dt:
                        break
                    slots.extend(parse_availability_slots(availability))
            filtered = [(s, sub) for s, sub in slots if slot_matches_filter(s)]
            return filtered, data

        def format_slots_message(slots):
            by_date = {}
            for s, sub in slots:
                dt = datetime.datetime.strptime(s[:-10], "%Y-%m-%dT%H:%M:%S")
                d = dt.strftime("%d/%m/%Y")
                label = dt.strftime("%H:%M")
                if sub:
                    label += " (rempl.: " + sub + ")"
                by_date.setdefault(d, []).append(label)
            return "\n".join(d + " : " + ", ".join(times) for d, times in sorted(by_date.items()))

        def main():
            entries = config["entries"]
            days_names = {0:"lundi",1:"mardi",2:"mercredi",3:"jeudi",4:"vendredi",5:"samedi",6:"dimanche"}
            names_str = "\n  - ".join(e["name"] for e in entries)
            if slot_filter:
                filter_lines = []
                for f in slot_filter:
                    sh, sm = divmod(f["start"], 60)
                    eh, em = divmod(f["end"], 60)
                    filter_lines.append(days_names.get(f["day"], "?") + " " + str(sh).zfill(2) + ":" + str(sm).zfill(2) + "-" + str(eh).zfill(2) + ":" + str(em).zfill(2))
                filter_str = "\n  - ".join(filter_lines)
            else:
                filter_str = "tous creneaux"
            sub_str = "inclus" if include_substitutes else "exclus"
            print(now() + " - Demarrage\n  Medecins :\n  - " + names_str + "\n  Filtres  :\n  - " + filter_str + "\n  Remplacants : " + sub_str)
            send_telegram(
                "<b>Doctolib Checker demarre</b>\n" +
                "<b>Medecins :</b>\n- " + "\n- ".join(e["name"] for e in entries) + "\n" +
                "<b>Filtres :</b>\n- " + filter_str.replace("\n  - ", "\n- ") + "\n" +
                "<b>Remplacants :</b> " + sub_str + "\n" +
                "Jusqu au " + limit_date + " - toutes les " + str(config["interval_in_seconds"]) + "s"
            )
            last_slots = {entry["name"]: None for entry in entries}

            while True:
                try:
                    for entry in entries:
                        name = entry["name"]
                        try:
                            current_slots, data = get_all_slots(entry["url"])
                        except Exception as e:
                            print(now() + " - Erreur fetch [" + name + "] : " + str(e))
                            continue

                        if current_slots != last_slots[name]:
                            if current_slots:
                                msg = format_slots_message(current_slots)
                                booking_url = entry.get("booking_url")
                                link = ('\n<a href="' + booking_url + '">Prendre rendez-vous</a>') if booking_url else ""
                                send_telegram("<b>" + str(len(current_slots)) + " creneau(x) - " + name + "</b>\n" + msg + link)
                                print(now() + " - ALERTE [" + name + "] : " + str(len(current_slots)) + " creneau(x)\n" + msg)
                            else:
                                if last_slots[name] is not None:
                                    send_telegram("<b>Plus de creneaux - " + name + "</b>\nProchain : " + str(data.get("next_slot", "?")))
                                    print(now() + " - [" + name + "] Tous les creneaux ont disparu")
                            last_slots[name] = current_slots
                        else:
                            if not current_slots:
                                print(now() + " - [" + name + "] Pas de creneau avant " + limit_date)
                            else:
                                print(now() + " - [" + name + "] " + str(len(current_slots)) + " creneau(x) inchange(s)")

                    if config["alive_check"] and datetime.datetime.now().hour == config["hour_of_alive_check"] and datetime.datetime.now().minute == 0:
                        send_telegram("<b>Doctolib Checker actif</b>\nSurveillance jusqu au " + limit_date)

                    if config["run_in_loop"]:
                        time.sleep(config["interval_in_seconds"])
                    else:
                        break
                except Exception as e:
                    print(now() + " - Erreur globale : " + str(e))
                    send_telegram("<b>Erreur Doctolib Checker</b>\n" + str(e))
                    if config["run_in_loop"]:
                        time.sleep(config["interval_in_seconds"])
                    else:
                        break

        if __name__ == "__main__":
            main()

    command:
      - /bin/bash
      - -c
      - |
        echo ">>> [1/4] Installation pyyaml..."
        pip install pyyaml --quiet --no-warn-script-location 2>&1 | grep -v "WARNING\|notice"
        echo ">>> [2/4] Ecriture des scripts..."
        mkdir -p /app
        python3 -c "import os; open('/tmp/gen_config.py','w').write(os.environ['CONFIG_SCRIPT'])"
        python3 -c "import os; open('/app/doctolib_checker.py','w').write(os.environ['CHECKER_SCRIPT'])"
        echo ">>> [3/4] Generation de la configuration..."
        python3 /tmp/gen_config.py
        echo ">>> [4/4] Demarrage du checker..."
        python3 /app/doctolib_checker.py
```

---

## Variables de configuration

| Variable | Obligatoire | Description |
|---|---|---|
| `DOCTOLIB_URLS` | ✅ | URLs API Doctolib (`availabilities.json`), une par ligne (onglet Réseau) |
| `DOCTOLIB_NAMES` | ✅ | Noms des médecins, un par ligne (même ordre que les URLs) |
| `DOCTOLIB_BOOKING_URLS` | ❌ | URLs de réservation (barre d'adresse), une par ligne — active le lien cliquable dans les notifs |
| `LIMIT_DATE` | ✅ | Date limite — ne notifie que si le créneau est avant cette date (`YYYY-MM-DD`) |
| `LIMIT` | ✅ | Fenêtre de recherche en jours (max `15`, imposé par Doctolib) |
| `INTERVAL_SECONDS` | ✅ | Intervalle entre chaque vérification en secondes (`300` = 5 min) |
| `TELEGRAM_BOT_TOKEN` | ✅ | Token du bot Telegram (obtenu via @BotFather) |
| `TELEGRAM_CHAT_ID` | ✅ | Identifiant de ta conversation Telegram |
| `ALIVE_CHECK` | ❌ | Envoie une notification quotidienne "le script tourne" (`true`/`false`) |
| `ALIVE_CHECK_HOUR` | ❌ | Heure de la notification "alive" (format 24h, ex: `9`) |
| `SLOT_FILTER` | ❌ | Plages horaires à surveiller — vide = tous les créneaux |
| `INCLUDE_SUBSTITUTES` | ❌ | `true` = inclure les créneaux des remplaçants (défaut) / `false` = praticien original uniquement |

---

## Comportement des notifications

| Situation | Notification envoyée |
|---|---|
| Nouveaux créneaux détectés | ✅ Liste complète des créneaux par date |
| Créneau avec un remplaçant | ✅ Heure suivie de `(rempl.: Dr. Nom)` si `INCLUDE_SUBSTITUTES=true` |
| Créneaux disponibles (méthode A) | ✅ Lien cliquable "Prendre rendez-vous" inclus dans la notification |
| Mêmes créneaux qu'au check précédent | ❌ Pas de spam |
| Un ou plusieurs créneaux changent | ✅ Nouvelle liste complète |
| Tous les créneaux disparaissent | ✅ "Plus de créneaux" |

---

## Ajouter un second médecin

Dans `DOCTOLIB_URLS`, ajouter une URL API par ligne. Même chose pour `DOCTOLIB_BOOKING_URLS` si tu veux les liens (même ordre) :

```yaml
DOCTOLIB_URLS: |
  https://www.doctolib.fr/availabilities.json?visit_motive_ids=AAA&agenda_ids=AAA&practice_ids=AAA&telehealth=false&start_date=2026-04-15&limit=5
  https://www.doctolib.fr/availabilities.json?visit_motive_ids=BBB&agenda_ids=BBB&practice_ids=BBB&telehealth=false&start_date=2026-04-15&limit=5

DOCTOLIB_NAMES: |
  Dr. Premier Médecin
  Dr. Second Médecin

DOCTOLIB_BOOKING_URLS: |
  https://www.doctolib.fr/[specialite]/[ville]/[medecin-1]/booking/availabilities?...
  https://www.doctolib.fr/[specialite]/[ville]/[medecin-2]/booking/availabilities?...
```

Chaque médecin est surveillé indépendamment — la notification indique le nom du médecin concerné.
