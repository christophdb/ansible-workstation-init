# Ruhemodus: Instant-Wake auf dem B550I AORUS PRO AX

Analysiert und behoben am 03.09.2026 auf `pop-os`.

## Symptom

Der Rechner ging nicht in den Ruhemodus. Dahinter steckten **zwei unabhängige
Ursachen**, die zusammen den Eindruck erzeugten, Suspend sei generell kaputt:

1. **Automatischer Suspend war gar nicht aktiv.** In COSMIC stand
   `suspend_on_ac_time` auf `None` — nur der Bildschirm ging nach 30 Minuten aus.
   Der Rechner hat es also nie von selbst versucht. Passend dazu fand sich im
   Journal in 14 Tagen genau *ein* Suspend-Versuch.
2. **Manuell ausgelöster Suspend endete nach ~1 Sekunde.** Klassisches
   Instant-Wake.

## Diagnose

Im Kernel-Log lagen Ein- und Ausstieg in derselben Sekunde — zu schnell für einen
Nutzer, der gerade auf „Suspend" geklickt hat:

```
16:40:23  ACPI: PM: Preparing to enter system sleep state S3
16:40:23  Disabling non-boot CPUs ...
16:40:23  ACPI: PM: Low-level resume complete      <- selbe Sekunde
```

Der entscheidende Hinweis kam aus `/sys/firmware/acpi/interrupts/`: Seit dem Boot
hatte genau **ein** ACPI-Interrupt gefeuert, `gpe10`. Neun Weckquellen standen in
`/proc/acpi/wakeup` auf `*enabled`:

```
GP17 XHC0 GP18 GPP8 SWUS SWDS PT20 PT24 PTXH
```

`GPP0` war bereits abgeschaltet (siehe „Vorgeschichte" unten) — das Problem war
also schon einmal aufgetreten und nur unvollständig gelöst worden.

### Vorgehen

Getestet wurde mit `rtcwake -m mem -s 20`. Das umgeht logind und weckt per
RTC-Alarm zuverlässig wieder auf. Messgröße ist die verstrichene Zeit: ~20 s
bedeutet durchgeschlafen, 1–7 s bedeutet Instant-Wake. Parallel wurde der
`gpe10`-Zähler vorher/nachher verglichen.

| Runde | Konfiguration | Ergebnis |
|---|---|---|
| A | unverändert, alle neun scharf | **7 s** — `gpe10` feuerte |
| B | alle neun aus | **24 s** — durchgeschlafen |
| C | nur `GP17` + `XHC0` wieder scharf | **24 s** — durchgeschlafen |

Runde C ist die eigentlich wertvolle: Die USB-Tastatur hängt an
`0000:0b:00.3` (`XHC0`) hinter der Bridge `0000:00:07.1` (`GP17`). Beide sind
nachweislich unschuldig und dürfen scharf bleiben — **damit funktioniert das
Aufwecken per Tastendruck weiterhin.** Der Störer sitzt in den übrigen sieben.

Abschließend über den echten Pfad geprüft (`systemctl suspend`, also inklusive
logind und Sperrbildschirm): 28 Sekunden geschlafen, per Tastendruck geweckt,
`gpe10`-Zähler unverändert.

## Was installiert wurde

Am 03.09.2026 zunächst von Hand, seit 04.09.2026 von der Rolle `fix-suspend`
(Tag `suspend`) verwaltet — ein Playbook-Lauf stellt denselben Stand her.

### 1. `/usr/local/sbin/disable-wakeup-sources.sh`

Schaltet beim Boot die acht problematischen Quellen ab (die sieben plus `GPP0`).

**Wichtig:** Ein Schreibvorgang auf `/proc/acpi/wakeup` *toggelt* den Zustand.
Blindes `echo NAME > /proc/acpi/wakeup` schaltet eine bereits deaktivierte Quelle
also wieder ein. Das Skript prüft darum erst den Ist-Zustand:

```sh
for s in GPP0 GP18 GPP8 SWUS SWDS PT20 PT24 PTXH; do
    if grep -q "^$s[[:space:]].*\*enabled" /proc/acpi/wakeup; then
        echo "$s" > /proc/acpi/wakeup
    fi
done
```

### 2. `/etc/systemd/system/disable-wakeup-sources.service`

`Type=oneshot`, `RemainAfterExit=yes`, `WantedBy=multi-user.target`.

### 3. Entfernt: `disable-gpp0-wakeup.service`

Der alte Service deckte nur `GPP0` ab und schrieb blind. Er wurde deaktiviert und
gelöscht, `GPP0` ist jetzt im neuen Skript enthalten. Zwei parallel laufende
Services hätten sich beim Toggeln gegenseitig aufgehoben.

### 4. COSMIC-Auto-Suspend aktiviert

Gesteuert über `cosmic_suspend_on_ac_ms` in `group_vars/all.yml`.

```
~/.config/cosmic/com.system76.CosmicIdle/v1/suspend_on_ac_time   Some(1800000)
~/.config/cosmic/com.system76.CosmicIdle/v1/screen_off_time      Some(1800000)
```

Einheit ist Millisekunden, `1800000` = 30 Minuten.

## Nachprüfen

```bash
# Soll: nur GP17 und XHC0
grep '\*enabled' /proc/acpi/wakeup

systemctl status disable-wakeup-sources.service

# Kontrollierter Suspend-Test, weckt sich nach 20s selbst
sudo rtcwake -m mem -s 20

# Wer hat geweckt? Zaehler vor/nach dem Suspend vergleichen
grep -H . /sys/firmware/acpi/interrupts/gpe* | grep -v ' 0 '
```

## Rolle ausführen

```bash
ANSIBLE_RUN_TAGS=suspend ansible-playbook local.yml
```

**Nicht** `--tags suspend` verwenden, wenn wirklich nur diese Rolle laufen soll.
`ansible.cfg` setzt

```ini
[tags]
run = untagged
```

als Vorgabe, und `--tags` von der Kommandozeile wird **dazugehängt** statt zu
ersetzen. `--tags suspend` führt also zusätzlich alles Ungetaggte aus — inklusive
der `teleport`-Rolle. Nachgewiesen am 04.09.2026 mit einem Testplaybook; die
Umgebungsvariable `ANSIBLE_RUN_TAGS` überschreibt die Vorgabe dagegen sauber.

Verwirrend dabei: `--list-tasks` filtert in ansible-core 2.16 nur rollenweise und
listet ungetaggte Tasks trotzdem mit auf. Die Liste taugt hier nicht als Beleg.

Ein Lauf ist idempotent — der zweite meldet `changed=0`.

## Ist das universell?

Nein — die **Methode** ist universell, die **Namen** sind es nicht.

Instant-Wake durch zu scharf gestellte ACPI-Weckquellen ist ein verbreitetes
Problem, besonders auf AMD-Desktop-Boards. Auch das Vorgehen (Zähler in
`/sys/firmware/acpi/interrupts/` vergleichen, mit `rtcwake` eingrenzen, per
oneshot-Service festschreiben) lässt sich überall anwenden.

Die konkreten Bezeichner `GP17`, `PTXH`, `SWDS` usw. stammen dagegen aus den
ACPI-Tabellen (DSDT) genau dieses Mainboards. Auf anderer Hardware existieren
sie nicht oder bezeichnen andere Geräte — eine fest verdrahtete Liste wäre dort
wirkungslos oder schädlich.

Die Rolle `fix-suspend` ist deshalb board-abhängig aufgebaut: Die Zuordnung steht
in `group_vars/all.yml` unter `wakeup_fix_boards`, adressiert über
`ansible_board_name`. Für ein Board ohne Eintrag gibt die Rolle einen Hinweis aus
und ändert nichts.

### Ein neues Board aufnehmen

1. `grep '\*enabled' /proc/acpi/wakeup` — Ausgangslage notieren.
2. Reproduzieren: `sudo rtcwake -m mem -s 20`. Kommt er nach 1–7 s zurück statt
   nach 20 s, liegt Instant-Wake vor.
3. Alle aktiven Quellen abschalten, erneut testen. Schläft er jetzt durch, sitzt
   der Störer in dieser Gruppe.
4. Den Pfad zur Tastatur wieder scharf schalten und nochmal testen — so bleibt
   das Aufwecken per Tastendruck erhalten. Den Pfad findet man über
   `readlink -f /sys/bus/usb/devices/<gerät>` und die Sysfs-Spalte in
   `/proc/acpi/wakeup`.
5. Ergebnis als neuen Eintrag unter `wakeup_fix_boards` eintragen.

Ein Skript, das genau diese Stufen abarbeitet, steckt in der Historie dieser
Analyse; die Logik steht oben unter „Vorgehen".

## Hinweise zur Hardware

- Board: Gigabyte B550I AORUS PRO AX, **BIOS F15 vom 04.01.2022**. Für das Board
  gibt es deutlich neuere AGESA-Versionen; ein Update wäre der nächste Hebel,
  falls das Thema wiederkommt.
- Beim Resume meldet der xHCI-Controller `0000:02:00.0` regelmäßig
  `xHC error in resume, USBSTS 0x401, Reinit`, und der Creative Pebble V3 (USB
  `3-2.2`) kommt mit `error -5` zurück. Kosmetisch, stört den Suspend nicht.
- Als Rückfallebene bleibt der Power-Button (`PNP0C0C:00`) immer eine gültige
  Weckquelle.
