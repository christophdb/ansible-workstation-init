# Brother HL-3152CDW unter Linux – treiberlos einrichten

Kurzanleitung für treiberloses Drucken über **CUPS + IPP Everywhere** (AirPrint).
Keine Brother-Treiberpakete nötig.

## Voraussetzungen

- Drucker hängt im WLAN/LAN und hat eine feste IP – hier: **192.168.5.150**
- CUPS und Avahi laufen:

```bash
sudo systemctl enable --now cups avahi-daemon
```

## Einrichtung (ein Befehl)

CUPS baut die Warteschlange selbst aus den echten Druckerdaten – das ist der entscheidende Schritt:

```bash
lpadmin -p BrotherHL-3152CDW -E -v ipp://192.168.5.150/ipp/print -m everywhere
lpadmin -d BrotherHL-3152CDW
```

- `-m everywhere` → CUPS fragt die Fähigkeiten live beim Drucker ab und erzeugt einen passgenauen PPD (treiberlos).
- `-d` → setzt den Drucker als Standard.
- Wichtig: IPP-Pfad ist **`/ipp/print`**, nicht `/ipp`.

## Test

```bash
printf "Testdruck\n%s\n" "$(date)" | lp -d BrotherHL-3152CDW
lpstat -o BrotherHL-3152CDW   # leer = Job durchgelaufen
```

## Stolperfallen (die hier zuerst auftraten)

1. **Falscher Treiber im GUI:** Der Eintrag *„Generic → IPP Everywhere"* aus der
   foomatic-Datenbank passt **nicht** – der Drucker startet und bricht sofort ab.
   Grund: Das Gerät kann **kein PDF** (`pdf-versions-supported = none`), nur
   URF/PWG-Raster, srgb_8/sgray_8, fest 600 dpi.
   → Lösung: stattdessen `lpadmin … -m everywhere` (siehe oben).

2. **Geister-Queue von cups-browsed:** Es entstand automatisch eine zweite
   Warteschlange `Brother_HL_3152CDW_series` (Backend `implicitclass://`) ohne
   Zieladresse und wurde Standard → Jobs blieben auf „Ausführen" hängen.
   Falls das wieder passiert:

   ```bash
   sudo systemctl disable --now cups-browsed
   ```

## Diagnose-Befehle (zum Nachschauen)

```bash
lpstat -t                          # Gesamtstatus aller Queues/Jobs
lpstat -p BrotherHL-3152CDW        # Status dieser Queue
tail -n 40 /var/log/cups/error_log # Fehlerprotokoll (ggf. mit sudo)

# Welche Formate kann der Drucker wirklich?
ipptool -tv ipp://192.168.5.150/ipp/print get-printer-attributes.test \
  | grep -iE "document-format-supported|urf-supported|pdf"
```

## Ergebnis

Treiberloses Drucken in Farbe + Duplex über CUPS – ohne ein einziges Brother-Paket.
```bash
lpstat -d
# systemvoreingestelltes Ziel: BrotherHL-3152CDW
```
