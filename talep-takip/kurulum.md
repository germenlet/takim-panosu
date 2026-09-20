# Kurulum — Talep Takip

Bu, "Takım Panosu"ndan tamamen ayrı, yeni bir proje. Aynı Supabase hesabını (ve aynı projeyi) kullanıyor ama kendi tablolarıyla — yani yeni bir Supabase projesi açmana gerek yok, sadece aşağıdaki iki tabloyu ekleyeceğiz.

## 1) Veritabanı tablolarını oluştur

Supabase panelinde (takim-panosu projesinin olduğu yer) → **SQL Editor** → "New query" → aşağıdakini yapıştır → **Run**:

```sql
create table public.tt_fis (
  id uuid primary key default gen_random_uuid(),
  para_birimi text not null,
  toplam_tl numeric not null check (toplam_tl >= 0),
  aciklama text,
  created_at timestamptz not null default now()
);

create table public.tt_talep (
  id uuid primary key default gen_random_uuid(),
  kisi_adi text not null check (char_length(kisi_adi) <= 200),
  urun_adi text not null check (char_length(urun_adi) <= 300),
  urun_link text check (char_length(urun_link) <= 500),
  durum text not null default 'bekliyor' check (durum in ('bekliyor','alindi','odendi','iptal')),
  fis_id uuid references public.tt_fis(id),
  tutar_yabanci numeric check (tutar_yabanci >= 0),
  para_birimi text check (char_length(para_birimi) <= 10),
  tl_payi numeric check (tl_payi >= 0),
  created_at timestamptz not null default now(),
  alindi_at timestamptz,
  odendi_at timestamptz
);

alter table public.tt_fis enable row level security;
alter table public.tt_talep enable row level security;

create policy "public read" on public.tt_fis for select using (true);
create policy "public write" on public.tt_fis for insert with check (true);
create policy "public update" on public.tt_fis for update using (true);

create policy "public read" on public.tt_talep for select using (true);
create policy "public write" on public.tt_talep for insert with check (true);
create policy "public update" on public.tt_talep for update using (true);

alter publication supabase_realtime add table public.tt_fis;
alter publication supabase_realtime add table public.tt_talep;
```

⚠️ **Güvenlik notu (Takım Panosu'ndakiyle aynı mantık):** Bu tablolar, kişi girişinde gerçek şifre/hesap doğrulaması olmadan açık okuma/yazmaya izin veriyor — yani linki bilen herkes teorik olarak tüm talepleri görebilir/güncelleyebilir (silme kapalı). Bunu, "yönetici şifresi bilinmeden fiş oluşturulamaz, kişi girişinde de sadece isim yazılır" seviyesinde bir güven modeli olarak düşün — aile/arkadaş çevresi kullanımı için makul, halka açık hassas veri için değil.

## 2) E-posta bildirimi için EmailJS (ücretsiz)

Birisi "Ödedim" dediğinde sana otomatik mail gitmesi için:

1. https://www.emailjs.com → ücretsiz kayıt ol (Google/Gmail ile giriş yapabilirsin — billozcan@gmail.com)
2. Sol menüden **Email Services** → "Add New Service" → Gmail'i seç → hesabını bağla. Oluşan **Service ID**'yi not al (örn. `service_abc123`).
3. Sol menüden **Email Templates** → "Create New Template". İçeriği istediğin gibi yazabilirsin, örnek:
   - **To Email** alanına: `billozcan@gmail.com` yaz (sabit alıcı — bu şekilde her zaman sana gider)
   - **Subject**: `💸 Ödeme bildirimi: {{kisi_adi}}`
   - **Content**:
     ```
     {{kisi_adi}} adlı kişi ödeme yaptığını bildirdi.

     Ürün: {{urun_adi}}
     Tutar: {{tutar}}
     Senin payın: {{tl_payi}}
     Tarih: {{tarih}}
     ```
   - Kaydet, **Template ID**'yi not al (örn. `template_xyz456`).
4. Sol menüden **Account → General** → **Public Key**'i not al.
5. Bu üç değeri (Service ID, Template ID, Public Key) bana yapıştır, dosyaya gömüp sana hazır halde geri vereceğim. (Public key client-side kullanım için tasarlanmıştır, paylaşman güvenlik sorunu yaratmaz.)

Ücretsiz plan ayda 200 e-posta gönderimine izin veriyor — bu kullanım için fazlasıyla yeterli.

Bu adımı atlarsan uygulama yine çalışır, sadece e-posta gitmez — admin panelini açık tuttuğunda ödemeleri anlık (canlı) olarak yine görürsün.

## 3) Yönetici şifresi

Varsayılan yönetici şifresi: **`talep2026`**

Değiştirmek istersen bana yeni şifreni söyle, `index.html` içindeki `ADMIN_PASS_HASH` değerini güncelleyip sana geri veririm (şifrenin kendisi kod içinde açık yazmıyor, SHA-256 hash'i tutuluyor — yine de bunu gerçek bir güvenlik önlemi değil, "tesadüfen bulunmasın" seviyesinde düşün).

## 4) Yayınlama

`talep-takip/index.html` dosyasını, `takim-panosu` projesini nasıl yayında tutuyorsan aynı şekilde (GitHub Pages, Netlify, vs.) yayınlayabilirsin — build adımı gerektirmiyor, tek dosya.

Kişiler için paylaşacağın link doğrudan `.../talep-takip/index.html` olabilir; istersen bir kişiye özel link de verebilirsin: `.../talep-takip/index.html?ben=Ahmet%20Yılmaz` — bu, o kişinin adını otomatik doldurup girişini kolaylaştırır.
