# Issue tracker: GitHub

Spec dan tiket repo ini disimpan di GitHub Issues.
Gunakan gh CLI dengan repo rafsidqi/BRUTAL-raf.

## Operasi

- Baca: gh issue view <nomor> --repo rafsidqi/BRUTAL-raf --comments
- Daftar: gh issue list --repo rafsidqi/BRUTAL-raf --state open
- Buat: gh issue create --repo rafsidqi/BRUTAL-raf --title "..." --body-file <file>
- Komentar: gh issue comment <nomor> --repo rafsidqi/BRUTAL-raf --body-file <file>
- Label: gh issue edit <nomor> --repo rafsidqi/BRUTAL-raf --add-label "..."
- Selesaikan: gh issue close <nomor> --repo rafsidqi/BRUTAL-raf --comment "..."

Untuk isi multiline, simpan teks ke file UTF-8 dan gunakan --body-file.

## Spec, tiket, dan dependensi

- Publikasikan spec sebagai issue.
- Buat satu issue per tiket dan tautkan ke issue spec.
- Catat hubungan blocking dengan native issue dependencies.
  Jika tidak tersedia, gunakan baris "Blocked by: #<nomor>".
- Tiket siap dikerjakan setelah semua blocker selesai.
- Tutup tiket setelah kriteria penerimaan terpenuhi.
  Sertakan bukti validasi dan referensi commit atau PR.
- Tutup spec setelah semua tiketnya selesai.

## Pull requests as a triage surface

PRs as a request surface: no.

## Ketersediaan

Operasi tiket memerlukan Issues aktif dan izin GitHub yang sesuai.
Jika operasi ditolak, laporkan kendalanya; jangan mengklaim tiket
berhasil dibuat atau ditutup.
