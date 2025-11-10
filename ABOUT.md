# DSMF Vətəndaş Müraciət Botu - About

## 🤖 Bot Haqqında

**DSMF Citizen Appeal Bot** - vətəndaşlardan müraciətləri qəbul edən, qrupda icraçılar tərəfindən cavablandırılan və nəticəni vətəndaşa çatdıran Telegram botu.

## ⚡ Sürətli Başlayış

1. Bot-a `/start` yazın
2. Anketi doldurub şəkil göndərin
3. Müraciət icraçı qrupuna gedir
4. İcraçı cavab/imtina verir
5. Nəticə sizə DM-ə çatdırılır

## 📋 Xüsusiyyətlər

- **Mərhələli Anket:** Ad, telefon, FIN, şəxsiyyət şəkli, mövzu, məzmun
- **Real-Time Status:** 🟡 Gözləyir → 🔴 Vaxtı keçir → 🟢 İcra edildi / ⚫ İmtina
- **İcraçı Düymələri:** ✉️ Cavablandır / 🚫 İmtina (inline)
- **Vətəndaş Bildirişi:** Cavab və ya imtina DM-ə gedir
- **PostgreSQL Dəstəyi:** Railway-də çalışan yerli deployment
- **CSV Export:** Admin müraciətləri Excel-ə download edə bilərlər
- **Qara Siyahı:** Spam istifadəçiləri avtomatik bloklamaq

## 🎯 İstifadəçilər

- **Vətəndaşlar:** Şikayət və tələfat göndərməsi üçün
- **İcraçılar:** Qrupda müraciətləri cavablandırması üçün
- **Adminlər:** Sistemi idarə etməsi, statistika alması üçün

## 🔧 Komandalar

**Adi istifadəçilər:**
- `/start` - Botu başlat
- `/help` - Kömək
- `/ping` - Sağlamlıq yoxlaması

**Admin:**
- `/blacklist` - Qara siyahıyı göstər
- `/ban <id>` - İstifadəçini qara siyahıya sal
- `/unban <id>` - Qara siyahıdan çıxar
- `/clearall` - ⚠️ Bütün müraciətləri sil (test)
- `/export` - Müraciətləri CSV-ə indir

## 🌍 Dil & Vaxt

- 🇦🇿 Azərbaycanca
- ⏰ Bakı Vaxtı (UTC+4)
- 📅 dd.mm.yyyy HH:MM:SS formatı

## 💬 Feedback

Hər hansı sual və ya təklif üçün bot administratorunə yazın.

---

**v0.4.2** | PostgreSQL CSV Export + SQLAlchemy Session Fixes | 2025-11-10
