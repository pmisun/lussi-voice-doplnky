# Lussi Voice – doplňky pro Home Assistant

Doplněk **Lussi agent** spojí Home Assistant ve tvém domě s Aplikací Lussi Voice: telefony doma mluví
s centrálou přímo (brána na portu 8444), na dálku přes šifrované spojení Lussi cloudu. Nic dalšího
(Nabu Casa, NGINX, přesměrování portů) nepotřebuje.

## Instalace

1. V Home Assistantu: **Nastavení → Doplňky → Obchod doplňků → ⋮ → Repozitáře**.
2. Přidej `https://github.com/pmisun/lussi-voice-doplnky` a potvrď.
3. V obchodě najdi **Lussi agent** → **Instalovat** → **Spustit**. **Lussi agent (pilot)** je kanál pro
   zkušební domy: dostává každou novou verzi hned. Nainstaluj jen jeden z nich.
4. Otevři panel **Lussi agent** v bočním menu. Párovací kód `LUS-…` z panelu zadej v Aplikaci Lussi Voice.

Doplněk běží na Home Assistant OS na `aarch64` (např. Home Assistant Green, Raspberry Pi 4 a 5 v 64 bitech)
a `amd64`. Volby doplňku a jejich význam popisuje karta **Dokumentace** u doplňku.

## Co tu je

Jen soubory pro Supervisor: `repository.yaml`, `lussi_agent/` (stabilní kanál) a `lussi_agent_pilot/`
(pilot). Obraz doplňku je hotový v GitHub Container Registry (`ghcr.io/pmisun/{arch}-addon-lussi-agent`);
Supervisor ho stáhne, nic se nestaví. Stabilní verze je přesně ten obraz, který předtím obstál v pilotu.
Změny jsou v `CHANGELOG.md` u každého doplňku. Aktualizaci nabídne Home Assistant v Nastavení → Aktualizace.
