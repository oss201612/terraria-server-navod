# Terraria dedikovaný server — instalace na Linuxu

Testováno na Ubuntu 22.04 / Debian. **Verze serveru: 1.4.5.6.**

Co se změnilo oproti minulé verzi návodu:
- Nová URL balíčku: `terraria-server-1456.zip`
- Rozbalená složka je teď `1456/` (dřív `1449`)
- Vypuštěno `lib32gcc-s1` — pro 64bitovou binárku není potřeba

---

## Část 1 — Vanilla server

### 1. Závislosti
64bitová binárka je samostatná. Potřebuješ jen `unzip` a `screen`.

```bash
sudo apt update
sudo apt install -y unzip screen
```

### 2. Vyhrazený uživatel
Server spouštěj pod vlastním neprivilegovaným uživatelem, ne pod rootem.

```bash
sudo useradd -m -d /opt/terraria -s /bin/bash terraria
```

> Pokud dostaneš `command not found`, `/usr/sbin` není v PATH. Spusť nejdřív `sudo -i`, nebo příkaz volej jako `/usr/sbin/useradd ...`.

### 3. Stažení a rozbalení
Přepni se na uživatele terraria a stáhni 1.4.5.6:

```bash
sudo -i -u terraria
cd ~
wget https://terraria.org/api/download/pc-dedicated-server/terraria-server-1456.zip
unzip terraria-server-1456.zip
cd 1456/Linux
chmod +x TerrariaServer.bin.x86_64
```

> Složka `1456` se mění s každou verzí (1449, 1412 atd.). Při aktualizaci ji vždy uprav.

### 4. serverconfig.txt
Vytvoř `serverconfig.txt` vedle binárky (ve složce `1456/Linux`):

```ini
world=/opt/terraria/.local/share/Terraria/Worlds/myworld.wld
autocreate=3
worldname=MyWorld
difficulty=0
maxplayers=8
port=7777
password=changeme
motd=Welcome!
secure=1
language=en-US
```

- `autocreate`: 1=malý, 2=střední, 3=velký (použije se jen když svět ještě neexistuje)
- `difficulty`: 0=klasický, 1=expert, 2=master, 3=journey
- Světy se ukládají do `~/.local/share/Terraria/Worlds/`
- Pro otevřený server řádek `password` smaž

### 5. Spuštění (screen)
Ze složky `1456/Linux`:

```bash
screen -dmS terraria ./TerrariaServer.bin.x86_64 -config serverconfig.txt
```

- Připojení ke konzoli: `screen -r terraria`
- Odpojení (server běží dál): `Ctrl+A` a pak `D`
- Čisté vypnutí: připoj se a napiš `exit`

### 6. Firewall (ufw)
```bash
sudo ufw allow 7777/tcp
```
(Některé cloudové hostingy chtějí i UDP: `sudo ufw allow 7777/udp`)

### 7. Automatické spouštění (systemd)
Vytvoř `/etc/systemd/system/terraria.service`:

```ini
[Unit]
Description=Terraria Dedicated Server
After=network.target

[Service]
Type=forking
User=terraria
Group=terraria
WorkingDirectory=/opt/terraria/1456/Linux
ExecStart=/usr/bin/screen -dmS terraria ./TerrariaServer.bin.x86_64 -config serverconfig.txt
ExecStop=/usr/bin/screen -S terraria -X stuff "exit\n"
TimeoutStopSec=60
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now terraria
systemctl status terraria
```

> `WorkingDirectory` a `ExecStart` obsahují složku verze `1456` — při aktualizaci uprav obojí.

---

## Část 2 — tModLoader server

tModLoader se spravuje přes samostatný skript `manage-tModLoaderServer.sh`. Stáhni ho rovnou na server; žádné další soubory z repozitáře nepotřebuješ.

```bash
wget https://raw.githubusercontent.com/tModLoader/tModLoader/1.4/patches/tModLoader/Terraria/release_extras/DedicatedServerUtils/manage-tModLoaderServer.sh
chmod +x manage-tModLoaderServer.sh
```

### Příprava modpacku (na tvém herním PC)
Ze složky Mods tvého klienta tModLoaderu zkopíruj tyhle soubory vedle skriptu (nebo do složky, kterou zadáš přes `--folder`):
- `enabled.json` — které mody jsou zapnuté (povinné, jinak se nic nenačte)
- `install.txt` — ID modů z Workshopu pro automatické stažení (potřeba pro `install-mods`)

Složka Mods na klientovi:
- Linux: `~/.local/share/Terraria/tModLoader/Mods/`
- Windows: `%UserProfile%\Documents\My Games\Terraria\tModLoader\Mods\`

### Instalace tModLoaderu — vyber jednu možnost
**SteamCMD (doporučeno)** — nejdřív nainstaluj SteamCMD a ujisti se, že je v PATH:
```bash
sudo apt install -y steamcmd      # Ubuntu/Debian (zapni multiverse/non-free, odsouhlas Steam EULA)
# Arch: AUR 'steamcmd'   |   Gentoo: games-util/steamcmd
```
Potom:
```bash
./manage-tModLoaderServer.sh install-tml --username your_steam_username
```
(Potřebuješ Steam účet, který vlastní tModLoader — je zdarma, stačí si ho jednou přidat do knihovny.)

**GitHub (bez přihlášení do Steamu):**
```bash
./manage-tModLoaderServer.sh install-tml --github
```

### Instalace modů a spuštění
```bash
./manage-tModLoaderServer.sh install-mods     # načte install.txt + enabled.json
./manage-tModLoaderServer.sh start
```
- `install` udělá `install-tml` i `install-mods` najednou
- Přes `--folder /plna/cesta` udržíš data serveru (Mods, Worlds, serverconfig.txt) na jednom místě; uváděj ho u každého příkazu
- Pro spouštění bez vstupu / přes systemd zkopíruj vzorový `serverconfig.txt` a vyplň ho (stejné parametry jako v Části 1)

### Aktualizace
- Instalace přes SteamCMD: spusť znovu `./manage-tModLoaderServer.sh install-tml`
- Instalace přes GitHub: spusť znovu `./manage-tModLoaderServer.sh install-tml --github`
- Jen mody: `./manage-tModLoaderServer.sh install-mods`

### Přesměrování portu
Stejné jako u vanilla: otevři TCP 7777 v ufw a přesměruj ho na routeru, pokud jsi za NATem.
