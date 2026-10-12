# Tugas untuk Claude Code: tombol mati & layar lag setelah upload `novaband_display`

Kamu bekerja di repo Nova-Band, branch `fitur/sensor-max30102-gy50-mlx90614`, commit `165a8c3`
("Jam LilyGo membaca sensor ESP32-C3 dari RTDB dan sinkron waktu NTP otomatis").
Pemilik repo tidak bisa compile di komputernya, jadi **kamu yang compile & uji di laptop ini**.

## Gejala
Setelah sketch `firmware/novaband_display` (LilyGo T-Display-S3) di-upload dari commit itu:
1. Tombol (termasuk tombol on/off, GPIO 0 = PREV, GPIO 14 = NEXT) tidak berfungsi / tidak responsif.
2. Layar sangat patah-patah (frame rate jatuh).

## Apa yang berubah di commit itu (hanya 3 file)
- `firmware/novaband_display/cloud.h` dan salinannya di `firmware/novaband_c3_sensors/cloud.h`
  (harus tetap identik):
  - `cloud::watch(role)` + `cloud::remote` (struct telemetri perangkat lain).
  - `getRemote()`: HTTP GET `/devices/esp32c3/live.json` tiap 2 dtk di task `cloud`
    (FreeRTOS, core 0, prioritas 1), memakai satu `HTTPClient http` global yang juga dipakai `put()`.
  - `HTTPClient http` dulunya `static` lokal di `put()`, sekarang variabel namespace bersama.
- `firmware/novaband_display/novaband_display.ino`:
  - `applySensors()`: kalau sensor lokal tidak ada/basi, pakai `cloud::remote` (HR/SpO2/suhu dari C3).
  - `ntpClockTick()` dipanggil di `loop()`: set `model.epochOffset` dari NTP (WIB) sekali per jam.
  - `cloud::watch("esp32c3");` dipanggil sebelum `cloud::begin(devName);`.

## Konteks arsitektur (penting)
- Render layar dibagi dua core: task `render0` (core 0, prioritas 3) menggambar band atas,
  `loop()` (core 1) menggambar band bawah, lalu keduanya dikirim lewat bus i80.
  `loop()` juga yang membaca tombol (`bNext.poll()`, `bPrev.poll()`). Jadi kalau core 0 sibuk/terblokir,
  frame dan tombol sama-sama terganggu.
- Task `cloud` juga di core 0 (WiFi + TLS mbedtls). Dugaan utama: GET tambahan tiap 2 dtk ditambah PUT
  tiap 2 dtk membuat core 0 jenuh (mis. koneksi TLS tidak bertahan sehingga tiap request handshake
  penuh, atau GET gagal/timeout 8 dtk). Dugaan ini BELUM terbukti.
- Perubahan NTP hanya hitungan kecil, kecil kemungkinan jadi penyebab.

## Yang harus kamu lakukan
1. **Jangan langsung mengubah kode.** Buktikan dulu penyebabnya.
2. Pastikan `firmware/novaband_display/secrets.h` ada (salin dari `secrets.example.h` dan isi) —
   file itu tidak ada di git.
3. Compile & upload (lihat `firmware/flash.ps1` untuk opsi board; FQBN:
   `esp32:esp32:esp32s3:CDCOnBoot=cdc,FlashSize=16M,PSRAM=opi,PartitionScheme=custom,UploadSpeed=921600`).
   Pastikan partisi + `assets.bin` ikut ter-flash (`flash.ps1` melakukannya); sketch saja tanpa
   partisi/assets yang benar juga bisa membuat layar bermasalah. Bandingkan dengan `firmware/prebuilt/`.
4. Buka Serial Monitor (115200) dan catat: baris `# mem: ... internal bebas ... blok terbesar ...`,
   `# cloud: ...` (terutama `GET esp32c3 -> <kode>` yang berulang atau negatif), `# ble ...`,
   dan `# waktu: sinkron NTP (WIB)`.
5. **Bisect:** komentari baris `cloud::watch("esp32c3");` di `novaband_display.ino`, upload ulang.
   - Kalau lag & tombol normal → penyebabnya `getRemote()`. Perbaiki: perlambat interval GET (mis. 5 dtk),
     pastikan koneksi TLS keep-alive bertahan (jangan handshake tiap request), jangan sampai request
     memblokir lama (timeout lebih kecil), dan/atau pastikan task `cloud` tidak mengganggu `render0`.
   - Kalau masih lag → bukan dari fitur baca C3. Cek `ntpClockTick()` (komentari panggilannya di `loop()`),
     lalu bandingkan dengan commit sebelumnya `55362ee`:
     `git stash; git checkout 55362ee -- firmware/novaband_display` dan uji lagi.
6. Setelah penyebab ketemu dan diperbaiki, **pertahankan kedua fitur**:
   - jam LilyGo membaca HR/SpO2/suhu dari `/devices/esp32c3/live` bila sensor lokal tidak ada/basi,
   - jam layar sinkron NTP (WIB) otomatis saat WiFi tersambung (waktu dari HP via BLE tetap menang).
   Salinan `cloud.h` di kedua folder harus tetap identik.
7. Laporkan: penyebab pasti, bukti (log serial), perubahan yang dibuat, dan hasil uji
   (tombol berfungsi, frame mulus, data C3 tampil di jam, jam layar benar).

## Catatan
- Aturan RTDB (`database.rules.json`) tidak perlu diubah: `/devices` sudah `.read: true`.
- ESP32-C3 (`novaband_c3_sensors`) tidak perlu di-upload ulang; ia hanya harus menyala dan mengirim
  ke `/devices/esp32c3/live` (sumber data).
- Jangan commit `secrets.h`.
