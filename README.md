# JokiHub — Marketplace Jasa Joki Game

Aplikasi marketplace full-stack berbasis Next.js App Router, TypeScript, PostgreSQL, Prisma, dan autentikasi JWT cookie.

## Fitur
- Role `USER`, `JOKER`, `ASSISTANT`, `ADMIN`.
- Pendaftaran user dan login; user mengajukan diri menjadi penjoki dan menunggu persetujuan admin.
- Admin membuat asisten dan mengatur izin deposit, withdrawal, sengketa, dan order.
- Penjoki memasang jasa per game, harga, kategori, deskripsi, dan estimasi waktu.
- User mencari jasa, memesan, dan saldo masuk escrow.
- Penjoki menerima order, menyatakan selesai; user mengonfirmasi untuk melepaskan dana.
- Sengketa mempertahankan escrow sampai admin/asisten berwenang memutuskan refund atau release.
- Deposit manual bank/QRIS dengan bukti URL dan verifikasi admin.
- Withdrawal mengunci saldo sampai admin menandai dibayar; penolakan mengembalikan saldo.
- Fee platform dapat diatur admin (0–50%).
- Ledger untuk jejak mutasi saldo.

## Menjalankan lokal
1. Install Node.js 20+ dan PostgreSQL 14+.
2. Buat database PostgreSQL bernama `jokigame`.
3. Salin `.env.example` menjadi `.env`, lalu sesuaikan `DATABASE_URL` dan `AUTH_SECRET` (minimal 32 karakter acak).
4. Jalankan:

```bash
npm install
npx prisma generate
npx prisma migrate dev --name init
npm run db:seed
npm run dev
```

Buka `http://localhost:3000`.

Akun admin seed mengikuti `SEED_ADMIN_EMAIL` dan `SEED_ADMIN_PASSWORD` di `.env`. Ganti password default sebelum deployment.

## Pembayaran gateway
Deposit bank/QRIS manual dapat digunakan dengan URL bukti transfer dan persetujuan admin. Adapter gateway berada di `src/lib/payments.ts`. Untuk mengaktifkan Midtrans atau Xendit, integrasikan SDK resmi, buat endpoint webhook, verifikasi signature provider, validasi nominal dan status transaksi, serta lakukan update deposit secara idempotent dalam transaksi database. Saat ini mode gateway sengaja fail-closed dan tidak mengkredit saldo otomatis.

## Catatan sebelum produksi (wajib)
- Tambahkan rate limiting, CAPTCHA, verifikasi email/OTP, reset password, dan audit log admin.
- Upload bukti transfer/evidence sebaiknya melalui object storage privat; jangan menerima URL arbitrer tanpa validasi.
- Implementasikan webhook gateway terverifikasi sebelum menerima pembayaran otomatis.
- Buat proses payout bank nyata sebelum tombol admin “Tandai dibayar” digunakan sebagai bukti pembayaran.
- Pastikan job scheduler timeout aktif dan termonitor pada environment produksi.
- Terapkan CSRF/origin checks untuk deployment cookie-auth, CSP, monitoring, backup, dan uji penetrasi.
- Tambahkan pagination dan pencarian server-side untuk katalog serta dashboard.
- Hindari menyimpan password atau data rekening dalam log.

## Arsitektur saldo
`Wallet.availableBalance` adalah saldo yang dapat digunakan. `Wallet.heldBalance` adalah dana escrow. Saat order dibuat, saldo available dipindah ke held. Saat user konfirmasi, held buyer dikurangi dan saldo bersih penjoki ditambah. Saat banding dimenangkan user, held dikembalikan. Semua operasi keuangan dilakukan dalam transaksi Prisma dengan isolation serializable untuk pembuatan order.

## Fitur tambahan: timeout, chat, notifikasi, rating
- Timeout penjoki dapat diatur admin pada rentang 15–30 menit (default 20 menit). Scheduler memanggil `GET /api/cron/order-timeouts` setiap menit dengan header `Authorization: Bearer $CRON_SECRET`. Order yang melewati batas tanpa respons dibatalkan dan escrow dikembalikan secara atomik/idempotent.
- Vercel Cron sudah didefinisikan pada `vercel.json`. Untuk server sendiri, buat cron tiap menit, misalnya `* * * * * curl -fsS -H "Authorization: Bearer $CRON_SECRET" https://domainanda.com/api/cron/order-timeouts`.
- Chat tersedia per order, mendukung pesan teks dan lampiran bukti transfer (JPG/PNG/WEBP/PDF, maksimal 5 MB). Status banding tampil di detail order.
- Rating 1–5 dan komentar hanya dapat dikirim user setelah order berstatus `COMPLETED`, satu ulasan per order. Rata-rata rating tampil pada katalog jasa.
- Notifikasi in-app disimpan di database. Email dikirim melalui Resend jika `RESEND_API_KEY` dan `EMAIL_FROM` diatur. WhatsApp dikirim melalui WhatsApp Cloud API jika token, phone number ID, dan nomor WhatsApp user tersedia.

### Pengujian sebelum produksi
Jalankan migrasi pada database uji khusus, lalu:

```bash
npm run db:generate
npm run db:migrate
npm run test:integration
npm run build
```

Integration test timeout membuat data fixture sementara, memanggil proses refund dua kali secara bersamaan, lalu memastikan escrow kembali tepat sekali dan ledger hanya memiliki satu refund. Jalankan hanya pada database pengujian, bukan database produksi.

### Penting untuk deployment publik
- Cron harus aktif dan `CRON_SECRET` wajib dirahasiakan. Pastikan platform hosting mendukung jadwal per menit.
- Upload saat ini disimpan di `public/uploads/orders` untuk pengembangan lokal/self-host sederhana. Untuk serverless/multi-instance production, ganti ke object storage privat (S3/R2/Cloudinary) dengan signed upload/download URL dan antivirus scanning. Bukti pembayaran bersifat sensitif.
- Lengkapi rate limiting, verifikasi email/nomor, CSP, audit log, backup, pemantauan, dan review keamanan independen sebelum menerima uang sungguhan.
- WhatsApp Cloud API memerlukan nomor yang valid dan kebijakan/template pesan Meta yang sesuai. Notifikasi eksternal best-effort; notifikasi in-app tetap disimpan meskipun provider eksternal gagal.
