# karabatak.tech Sunucusu - Teknik Yol Haritasi

**Sunucu:** 187.77.110.69 (hostname: srv1778013)
**Musteri:** Karabey Marketing
**Guncel tarih:** 19.07.2026
**DNS:** Hostinger (nameserver hala atlas/hyperion.dns-parking.com, Cloudflare'e
gecis planlandi ama henuz tamamlanmadi)

---

## 1. Genel Mimari

Bu sunucuda 4 bagimsiz proje/site calisiyor. Hepsi ayni fiziksel sunucuda ama
birbirinden izole (farkli portlar, farkli process yoneticileri).

| Site/Servis | Domain | Port | Process Yoneticisi | Kod Yolu |
|---|---|---|---|---|
| Dashboard (statik) | karabatak.tech | - (nginx static) | - | /var/www/dashboard |
| HarleyOtoPost | otopost.karabatak.tech | 3000 | systemd (harley-site) | /opt/harleyotopost |
| Bot Kontrol Paneli | bot.karabatak.tech | 8002 (API) + statik | PM2 (botctl-backend) | /opt/botctl |
| SIP Otomatik Arama | (dogrudan IP:5000, nginx proxy yok) | 5000 | PM2 (autocall) | /opt/autocall |
| ~~MailFlow~~ | ~~mail.karabatak.tech~~ | - | **SILINDI (19.07.2026)** | - |

**Not:** `mail.karabatak.tech` nginx config'i ve `/opt/mailflow` klasoru 19.07.2026'da
tamamen kaldirildi - baska bir sunucuda calisan versiyonu var, burasi iptal edilmisti.

---

## 2. GitHub Repolari (hepsi 19.07.2026'da olusturuldu, Private)

| Repo | Icerik |
|---|---|
| `Hyb2779/karabatak-autocall` | SIP otomatik arama paneli (Python/Flask, 12 modul dosyasina bolundu) |
| `Hyb2779/karabatak-botctl` | Telegram bot yonetim paneli (FastAPI backend + React frontend) |
| `Hyb2779/karabatak-otopost` | HarleyOtoPost sitesi (Next.js + gomulu Telegram bot) |
| `Hyb2779/karabatak-dashboard` | Statik tek-sayfa dashboard (karabatak.tech kok domain) |

Push icin token: `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` (Hybridus'un genel
kullandigi token, bilincli olarak repo'larda url icinde tutuluyor).

---

## 3. Detaylar - Proje Bazinda

### 3.1 Dashboard (karabatak.tech)
- Tek `index.html`, hicbir backend yok
- `.gitignore` gerekmiyor, hassas veri yok

### 3.2 HarleyOtoPost (/opt/harleyotopost)
- Next.js (TypeScript) + PostgreSQL (`pg` paketi kullaniliyor)
- `bot/` alt klasorunde ayri bir Python Telegram botu var (kendi venv'i)
- systemd servisi: `harley-site.service`, `npm start` ile calisir, port 3000
- `.env` icerigi: DATABASE_URL, PANEL_USERNAME, PANEL_PASSWORD, PANEL_SECRET, PORT, NODE_ENV
- nginx: `/etc/nginx/sites-available/otopost` -> proxy_pass 127.0.0.1:3000

### 3.3 Bot Kontrol Paneli / botctl (/opt/botctl)
- Backend: FastAPI (`server.py`, `bot_manager.py`, `auth.py`) + MongoDB
- Frontend: React (Craco + Tailwind + shadcn/ui bilesenleri)
- `bot_manager.py`: coklu Telegram botunu ayri ayri subprocess olarak
  baslatip/durdurup yonetiyor (zip yukleme, dosya duzenleme, log okuma,
  cron ile zamanlanmis start/stop destekliyor)
- Onceden Supervisor ile calisiyordu (`botctl-backend` program adi),
  **19.07.2026'da PM2'ye tasindi**
- `.env` icerigi: MONGO_URL, DB_NAME, CORS_ORIGINS, JWT_SECRET,
  ADMIN_USERNAME, ADMIN_PASSWORD, BOT_STORAGE_DIR
- `bot_storage/` (420MB) - kullanicilarin yukledigi gercek bot dosyalari, git'e
  girmiyor, yedeklenmesi gerekiyorsa ayri ele alinmali
- nginx: `/etc/nginx/sites-available/botctl` -> `/api/` proxy 127.0.0.1:8002,
  geri kalani statik `/var/www/botctl` (frontend build ciktisi)
- Emergent platformunda olusturulmus proje (`.emergent/emergent.yml` var)

### 3.4 SIP Otomatik Arama Paneli (/opt/autocall)
- Python/Flask + Flask-SocketIO, Asterisk AMI ile SIP arama yapiyor
- **19.07.2026'da tamamen refactor edildi**: eskiden tek dosya (643 satir
  `app.py`), simdi 12 modul dosyasina bolundu:
  `constants.py`, `extensions.py`, `state.py`, `logger.py`,
  `settings_store.py`, `auth.py`, `asterisk_config.py`, `audio.py`,
  `ami_client.py`, `call_engine.py`, `routes.py`, `app.py` (ince giris noktasi)
- Eski `config.py` ve `autocall.py` (pyVoIP tabanli kullanilmayan prototip)
  refactor sirasinda silindi, kullanilmiyordu
- Coklu SIP hesabi (round-robin) + concurrency=60'a kadar esanli arama destegi
  (NCS deneyiminden uyarlandi)
- Onceden systemd (`autocall.service`) ile calisiyordu,
  **19.07.2026'da PM2'ye tasindi**
- Panel ayarlarindaki "Esanli Arama Sayisi" input'unun max degeri 10'dan
  60'a cikarildi (`templates/index.html`)
- `settings.json`, `results.json`, `uploads/`, `venv/` git'e girmiyor
- nginx proxy YOK, dogrudan `IP:5000` uzerinden erisiliyor (SocketIO nedeniyle
  boyle birakildi, ileride nginx arkasina alinabilir)

---

## 4. 19.07.2026 Oturumunda Yapilanlar (Ozet)

1. **Otomatik arama limit hatasi duzeltildi** - "Esanli Arama Sayisi" alaninin
   HTML `max="10"` sinirlamasi kaldirildi, 60'a cikarildi (backend'de zaten
   ust sinir yoktu, sadece frontend validasyonuydu)
2. **autocall ve botctl-backend PM2'ye tasindi** - eskiden systemd/Supervisor
   karisik yonetiliyordu, artik ikisi de PM2 altinda, `pm2 save` + `pm2 startup`
   ile kalici hale getirildi
3. **autocall tamamen refactor edildi** - 643 satirlik tek dosyadan 12 modul
   dosyasina bolundu, davranis degismedi (test edildi, dogrulandi)
4. **4 saattir cokuk olan nginx duzeltildi** - kok sebep: `karabatak.tech`
   (dashboard) icin SSL sertifikasi hic yoktu, nginx config test'i bu yuzden
   surekli basarisiz oluyordu, bu da TUM siteleri (otopost, bot, mail dahil)
   etkiliyordu. Once dashboard gecici devre disi birakilip diger siteler
   kurtarildi, sonra DNS (AAAA kaydi yanlisti) duzeltilip gercek sertifika
   uretildi, dashboard tekrar aktif edildi.
5. **DNS duzeltmeleri (Hostinger paneli uzerinden)**:
   - `karabatak.tech` A kaydi: `82.25.102.207` (yanlis) -> `187.77.110.69` (dogru)
   - `karabatak.tech` AAAA kaydi: silindi (yanlis bir IPv6'ya isaret ediyordu,
     Let's Encrypt dogrulamasini IPv6 uzerinden yapmaya calisip basarisiz oluyordu)
   - `otopost` ve `bot` A kayitlari eklendi (Cloudflare taramasinda hic yoktu)
   - `ftp.karabatak.tech` hala yanlis IP'ye (`82.25.102.207`) isaret ediyor -
     DUZELTILMEDI, dusuk oncelikli, siteleri etkilemiyor
6. **Kullanilmayan mailflow kaldirildi** - baska bir sunucuda calisan
   versiyonu oldugu icin bu sunucudaki kopya (systemd servisi, kod klasoru,
   nginx config'i) tamamen silindi
7. **4 proje GitHub'a alindi** (bkz. Bolum 2)

---

## 5. Acik Isler / Bilinen Sorunlar

1. **DNS hala Hostinger'da** - Cloudflare'e domain eklendi ama nameserver
   degisikligi (registrar tarafinda) henuz yapilmadi. Yapilirsa Cloudflare
   proxy/CDN avantajlarindan faydalanilabilir, ama su an ZORUNLU degil,
   Hostinger DNS'i dogru calisiyor.
2. **`ftp.karabatak.tech` yanlis IP'ye isaret ediyor** (`82.25.102.207`,
   sunucuya ait degil) - ne oldugu bilinmiyor, dusuk oncelik.
3. **botctl `bot_storage/` (420MB) yedeklenmiyor** - kullanicilarin yukledigi
   gercek bot kodlari git'e girmiyor (bilerek, cok buyuk), ama disk/sunucu
   kaybi durumunda bu veriler kaybolur. Ayri bir yedekleme stratejisi
   dusunulmeli (orn. gunluk tar.gz + harici depolama).
4. **`dashboard` reposu README'siz, aciklama yok** - tek `index.html`, ne
   oldugu/kime ait oldugu belgeli degil.
5. **autocall nginx arkasinda degil** - dogrudan `IP:5000` ile erisiliyor,
   SSL yok. Ileride domain baglanip nginx proxy + Certbot ile SSL eklenebilir.
