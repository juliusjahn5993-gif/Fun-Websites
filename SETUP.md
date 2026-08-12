# Einrichtung — Bewerbungsverfahren 33/2026

Diese Anleitung ist für Nicht-Entwickler geschrieben. Du brauchst keinerlei Programmierkenntnisse,
nur einen Browser. Zeitaufwand: etwa zehn Minuten.

Die Seite funktioniert **auch ohne jede Einrichtung**. Dann speichert sie alles nur auf dem Handy,
auf dem sie geöffnet wurde, und im Fuß der Seite steht dezent „Ranking derzeit nur lokal geführt."

> **Wichtig:** Das Live-Ranking über alle Gäste hinweg — also dass jeder Gast die Zeiten aller
> anderen sieht und der Beamer-Aushang sich von selbst füllt — funktioniert **nur mit
> eingerichtetem Supabase**. Ohne Supabase sieht jedes Handy ausschließlich seine eigenen Läufe.

---

## Was Supabase ist

Ein kostenloser Online-Speicher für Daten. Wir legen dort zwei Listen an: eine für die
Bestenliste, eine für die Schichtabstimmung. Das kostenlose Kontingent reicht für eine
Geburtstagsfeier um ein Vielfaches.

---

## Schritt 1 — Konto und Projekt anlegen

1. Auf <https://supabase.com> gehen und **Start your project** anklicken. Anmeldung z. B. mit GitHub
   oder E-Mail.
2. **New project** anklicken.
3. Einen Namen vergeben (z. B. `geburtstag-33`), ein Datenbank-Passwort setzen (irgendwo notieren,
   du brauchst es hier nicht wieder) und als Region **Central EU (Frankfurt)** wählen — das ist am
   nächsten an Leipzig und damit am schnellsten.
4. **Create new project** klicken und ein bis zwei Minuten warten, bis das Projekt bereitsteht.

---

## Schritt 2 — Die beiden Tabellen anlegen

1. In der linken Leiste auf **SQL Editor** klicken.
2. Auf **New query** klicken.
3. Den kompletten folgenden Block hineinkopieren und unten rechts auf **Run** klicken.

```sql
-- Tabelle 1: die Eignungsrangliste
create table public.leaderboard (
  id          bigint generated always as identity primary key,
  name        text        not null check (char_length(name) between 1 and 40),
  seconds     integer     not null check (seconds >= 0 and seconds < 100000),
  penalties   integer     not null default 0 check (penalties >= 0 and penalties < 1000),
  created_at  timestamptz not null default now()
);

-- Tabelle 2: die Schichtabstimmung
create table public.votes (
  id          bigint generated always as identity primary key,
  choice      text        not null check (choice in ('frueh','spaet')),
  comment     text        not null default '' check (char_length(comment) <= 240),
  created_at  timestamptz not null default now()
);

-- Zugriffsschutz einschalten
alter table public.leaderboard enable row level security;
alter table public.votes       enable row level security;

-- Regeln: jeder darf lesen und neue Einträge schreiben — aber nichts ändern und nichts löschen
create policy "rangliste lesen"     on public.leaderboard for select to anon, authenticated using (true);
create policy "rangliste eintragen" on public.leaderboard for insert to anon, authenticated with check (true);
create policy "stimmen lesen"       on public.votes       for select to anon, authenticated using (true);
create policy "stimmen abgeben"     on public.votes       for insert to anon, authenticated with check (true);

-- Live-Übertragung für beide Tabellen einschalten
alter publication supabase_realtime add table public.leaderboard;
alter publication supabase_realtime add table public.votes;
```

Wenn unten **Success. No rows returned** steht, hat alles geklappt.

---

## Schritt 3 — Live-Übertragung kontrollieren

Die letzten beiden SQL-Zeilen haben Realtime bereits eingeschaltet. Zur Sicherheit prüfen:

1. Linke Leiste → **Database** → **Replication** (in manchen Versionen **Publications**).
2. Bei `supabase_realtime` nachsehen, ob **leaderboard** und **votes** als aktiv markiert sind.
3. Falls nicht: die beiden Schalter dort umlegen.

Ohne diesen Schritt funktioniert die Seite trotzdem — sie fragt dann eben alle fünf Sekunden
nach neuen Einträgen, statt sie sofort zugeschickt zu bekommen. Der Unterschied fällt kaum auf.

---

## Schritt 4 — Die zwei Werte in die Datei eintragen

1. Linke Leiste → **Project Settings** (Zahnrad) → **API**.
2. Dort stehen zwei Angaben:
   * **Project URL** — sieht aus wie `https://abcdefghijklmnop.supabase.co`
   * **Project API keys → `anon` `public`** — ein sehr langer Buchstaben-Zahlen-Salat
3. Die Datei `index.html` in einem Texteditor öffnen (TextEdit, Notepad, VS Code — egal).
4. Ganz oben, in den ersten 20 Zeilen, steht dieser Block:

```js
var CONFIG = {
  SUPABASE_URL:      "",   // z. B. "https://abcdefghijklm.supabase.co"
  SUPABASE_ANON_KEY: ""    // der lange "anon public"-Schlüssel
};
```

5. Die beiden Werte **zwischen die Anführungszeichen** kopieren, speichern. Fertig. Das Ergebnis
   sieht so aus:

```js
var CONFIG = {
  SUPABASE_URL:      "https://abcdefghijklmnop.supabase.co",
  SUPABASE_ANON_KEY: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.sehr.langer.schluessel"
};
```

Der `anon`-Schlüssel darf öffentlich in der Datei stehen — dafür ist er gemacht. Die Regeln aus
Schritt 2 sorgen dafür, dass damit nur gelesen und neu eingetragen werden kann. Niemand kann
darüber Einträge ändern oder löschen. Den **`service_role`-Schlüssel** aus derselben Ansicht darfst
du dagegen **niemals** eintragen — der hat volle Rechte.

---

## Schritt 5 — Prüfen, ob es läuft

1. `index.html` doppelklicken, um sie im Browser zu öffnen.
2. Das Verfahren einmal komplett durchklicken (dauert fünf Minuten, macht aber Spaß).
3. Auf der Rangliste am Ende muss über der Tabelle stehen:
   **„Verbindung zur Personaldatenbank: aktiv"**
   Steht dort stattdessen „Datenbank: lokal geführt", stimmt etwas mit URL oder Schlüssel nicht —
   Schritt 4 noch einmal prüfen.
4. Gegenprobe: dieselbe Datei auf einem zweiten Gerät öffnen und dort ebenfalls durchlaufen. Beide
   Namen müssen auf beiden Geräten in der Liste stehen, ohne dass jemand neu lädt.

---

## Der Aushang für Fernseher oder Beamer

Dieselbe Datei, nur mit einem Zusatz hinter der Adresse:

```
…/index.html?ansicht=aushang
```

Das rendert ausschließlich die Rangliste, groß gesetzt fürs Querformat, ohne jede Bedienung. Die
Seite hält den Bildschirm wach, wo der Browser das erlaubt, und aktualisiert sich von selbst. Am
besten am Veranstaltungstag einmal einrichten und dann in Ruhe lassen.

---

## Gut zu wissen

* **Ein Lauf je Name.** Wer über „Erneut bewerben" mehrfach antritt, erscheint nur mit dem besten
  Lauf in der Rangliste. Gewertet wird die Bearbeitungszeit plus fünf Sekunden je Beanstandung.
* **Eine Stimme je Gerät.** Die Schichtabstimmung zählt pro Handy genau einmal, auch bei mehreren
  Durchläufen — damit die Auszählung belastbar bleibt.
* **Namen ändern oder Einträge löschen:** In Supabase unter **Table Editor** → `leaderboard`. Dort
  kannst du Zeilen von Hand bearbeiten oder entfernen; auf den Handys der Gäste verschwinden sie
  beim nächsten Aktualisieren.
* **Abstimmungsergebnis auslesen:** SQL Editor → `select choice, count(*) from votes group by choice;`
  Die Begründungen (die laut Formular „nicht gelesen" werden) stehen in der Spalte `comment`.
* **Fällt das Internet vor Ort aus,** schaltet die Seite selbsttätig auf lokalen Betrieb um. Niemand
  bekommt eine Fehlermeldung zu sehen, das Verfahren läuft weiter.

---

## Wenn etwas klemmt

| Symptom | Ursache | Lösung |
|---|---|---|
| „Datenbank: lokal geführt" trotz Einrichtung | URL oder Schlüssel falsch kopiert | Schritt 4 prüfen, auf Leerzeichen am Anfang/Ende achten |
| Liste bleibt leer, obwohl Gäste fertig sind | Die Policies aus Schritt 2 fehlen | SQL-Block aus Schritt 2 noch einmal ausführen |
| Neue Einträge erscheinen erst nach Sekunden | Realtime nicht aktiv | Schritt 3 |
| Nichts geht mehr | egal | Die Seite fällt automatisch auf lokalen Betrieb zurück. Die Feier ist dadurch nicht gefährdet. |
