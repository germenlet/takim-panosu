# Supabase Kurulumu — Takım Panosu için

## 1) Proje oluştur (2 dk)
1. https://supabase.com → "Start your project" → GitHub/Google ile giriş
2. "New project" → isim: `takim-panosu` (istediğin isim), bölge: Frankfurt/Europe (Türkiye'ye en yakın), database şifresi belirle (not al)
3. Proje kurulana kadar ~1-2 dk bekle

## 2) Tabloyu oluştur
Sol menüden **SQL Editor** → "New query" → aşağıdakini yapıştır → **Run**:

```sql
create table public.tp_kv (
  k text primary key,
  v text not null,
  updated_at timestamptz not null default now()
);

alter table public.tp_kv enable row level security;

create policy "public read" on public.tp_kv for select using (true);
create policy "public write" on public.tp_kv for insert with check (true);
create policy "public update" on public.tp_kv for update using (true);
create policy "public delete" on public.tp_kv for delete using (true);

alter publication supabase_realtime add table public.tp_kv;
```

⚠️ **Önemli güvenlik notu:** Bu policy'ler tabloyu tamamen açık yapıyor (herkes okuyup yazabilir). Bu şu anki `window.storage` modeliyle aynı güven seviyesi — güvenlik "oda kodunu bilenler görür" mantığına dayanıyor, gerçek kullanıcı girişi/yetkilendirme yok. Şirket içi kullanım için makul ama halka açık, hassas veri girilecek bir ürün olsaydı bunu kabul etmezdim. İstersen ileride gerçek kullanıcı girişi (auth) ekleyip bu policy'leri sıkılaştırabiliriz.

## 3) Anahtarları al
Sol menüden **Project Settings → API**:
- **Project URL** (örn. `https://abcdefgh.supabase.co`)
- **anon public** key (uzun bir JWT string)

Bu ikisini bana yapıştır, dosyalara gömüp sana hazır halde geri vereceğim. (anon key herkese açık/client-side kullanım için tasarlanmıştır, gizli tutman gereken `service_role` key'i **asla** paylaşma.)

## 4) Güvenlik sıkılaştırması (public kullanım için)

Adım 2'deki policy'ler tabloyu tamamen açık bırakıyor — herkes her satırı silebiliyordu, bu da script'le "tüm odaları sil" saldırısına açık demekti. **SQL Editor**'de yeni bir query açıp aşağıdakini çalıştır:

```sql
-- Silme iznini tamamen kaldır — uygulama zaten hiçbir zaman gerçek silme yapmıyor
-- (öğeler "del:true" ile işaretlenip update ile kaydediliyor), bu yüzden bu güvenli.
drop policy if exists "public delete" on public.tp_kv;

-- Anahtar formatını ("tp:ODAKODU:oyun") ve satır boyutunu sınırla —
-- rastgele/şişirilmiş verilerle tabloyu spam'lemeyi/kotayı tüketmeyi zorlaştırır.
alter table public.tp_kv
  add constraint tp_kv_key_format check (k ~ '^tp:[A-Z0-9]{1,8}:[a-z]{2,24}$'),
  add constraint tp_kv_value_size check (char_length(v) <= 200000);
```

Bunu çalıştırdıktan sonra:
- Kimse (anon key'i bilse bile) artık bir satırı **silemez** — sadece okuma/yazma/güncelleme kalır, uygulamanın ihtiyacı zaten bu.
- Uygulamanın kullanmadığı formatta bir anahtara veya 200KB'tan büyük bir değere yazma girişimi veritabanı seviyesinde reddedilir.

⚠️ Hâlâ kalan gerçek risk: oda kodunu bilen (veya 4 karakterlik kodu deneyerek bulan) herkes o odanın verisini okuyup normal şekilde güncelleyebilir — gerçek kullanıcı girişi (auth) olmadan bunun tam çözümü yok. "Oda kodunu bilen herkes görebilir, gizli bilgi yazma" uyarısı bu yüzden arayüzde duruyor. Daha ileri gitmek istersen (örn. oda başına parola, gerçek auth) ayrıca konuşabiliriz.
