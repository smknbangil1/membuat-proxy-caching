Menggabungkan **Squid** dan **Nginx** untuk membuat CDN lokal yang handal memungkinkan Anda memanfaatkan kelebihan masing-masing tool: **Squid** untuk caching granular dan analisis trafik, serta **Nginx** untuk kecepatan tinggi dalam melayani file statis dan konten besar. Berikut panduan langkah demi langkah:

---

## **1. Konsep Kombinasi Squid dan Nginx**

1. **Squid sebagai Reverse Proxy dan Cache Utama:**
   - Squid menangani permintaan pertama dari klien.
   - Squid menyimpan cache untuk konten web (HTTP/HTTPS) dan file kecil atau menengah.

2. **Nginx sebagai Server Statis atau Frontend:**
   - Nginx digunakan untuk melayani konten besar (video, file binary, dll.).
   - Nginx memanfaatkan cache Squid atau bertindak sebagai server CDN tambahan untuk konten tertentu.

3. **Flow Request:**
   - **Client → Squid → Nginx (untuk konten besar) → Origin Server.**
   - **Client → Squid → Cache Lokal (untuk konten kecil/berulang).**

---

## **2. Instalasi Squid dan Nginx**

### **a. Instal Squid**
1. Instal Squid di server lokal:
   ```bash
   sudo apt update
   sudo apt install squid
   ```

2. Konfigurasi cache Squid:
   - File konfigurasi: `/etc/squid/squid.conf`
   - Tambahkan pengaturan dasar untuk caching:
     ```squid
     cache_mem 512 MB
     maximum_object_size_in_memory 512 KB
     maximum_object_size 4 GB
     cache_dir ufs /var/spool/squid 5000 16 256
     acl localnet src 192.168.0.0/16
     http_access allow localnet
     ```

3. Pastikan Squid dapat menangani file besar:
   ```squid
   range_offset_limit -1
   quick_abort_min -1
   ```

4. Restart Squid:
   ```bash
   sudo systemctl restart squid
   ```

---

### **b. Instal Nginx**
1. Instal Nginx:
   ```bash
   sudo apt install nginx
   ```

2. Tambahkan konfigurasi caching pada Nginx:
   - File konfigurasi: `/etc/nginx/sites-available/cdn.conf`
   ```nginx
   proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cdn_cache:10m max_size=100g inactive=7d use_temp_path=off;

   server {
       listen 80;
       server_name cdn.local;

       location / {
           proxy_pass http://origin-server; # Ganti dengan server asli
           proxy_cache cdn_cache;
           proxy_cache_key "$scheme$request_method$host$request_uri";
           proxy_cache_valid 200 302 7d;
           proxy_cache_valid 404 1m;
           add_header X-Cache-Status $upstream_cache_status;
       }
   }
   ```

3. Aktivasi konfigurasi:
   ```bash
   sudo ln -s /etc/nginx/sites-available/cdn.conf /etc/nginx/sites-enabled/
   sudo systemctl reload nginx
   ```

---

## **3. Mengatur Alur Kombinasi**

### **a. Squid Mengarahkan Permintaan ke Nginx**
1. Modifikasi konfigurasi Squid agar meneruskan permintaan tertentu ke Nginx:
   - Tambahkan di `/etc/squid/squid.conf`:
     ```squid
     cache_peer 127.0.0.1 parent 80 0 no-query originserver name=nginx
     acl large_files urlpath_regex -i \.(mp4|mkv|exe|zip|tar|iso|avi|mp3)$
     cache_peer_access nginx allow large_files
     cache_peer_access nginx deny all
     ```

2. Dengan konfigurasi ini:
   - File besar (video, file binary) akan diteruskan ke Nginx.
   - File kecil akan tetap ditangani oleh Squid.

3. Restart Squid:
   ```bash
   sudo systemctl restart squid
   ```

---

### **b. Nginx Melayani Konten Statis dan Streaming**
1. Jika Nginx bertindak sebagai server frontend, arahkan klien langsung ke Nginx untuk file tertentu.
2. Gunakan DNS atau MikroTik untuk memetakan domain tertentu (misalnya `cdn.local`) ke IP Nginx:
   - MikroTik CLI:
     ```bash
     /ip dns static add name="cdn.local" address=192.168.1.10
     ```

---

## **4. Optimasi untuk Performa**

### **a. Squid Optimasi untuk HTTPS**
Jika Anda ingin caching HTTPS, Squid membutuhkan sertifikat lokal:
1. Aktifkan mode SSL bumping:
   ```squid
   http_port 3128 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/cert.pem
   ssl_bump server-first all
   ```
2. Hasilkan sertifikat CA lokal dan tambahkan ke klien:
   ```bash
   openssl req -new -x509 -days 365 -keyout /etc/squid/cert.pem -out /etc/squid/cert.pem
   ```

3. Tambahkan sertifikat ke browser klien agar tidak ada peringatan SSL.

---

### **b. Nginx Optimasi untuk File Besar**
1. Konfigurasi Nginx untuk menangani file besar dengan efisien:
   ```nginx
   client_body_buffer_size 128k;
   client_max_body_size 10G;
   sendfile on;
   tcp_nopush on;
   tcp_nodelay on;
   ```

2. Aktifkan caching berdasarkan MIME type:
   ```nginx
   location ~* \.(mp4|mkv|exe|zip|tar|iso|avi|mp3)$ {
       expires 30d;
       add_header Cache-Control "public";
   }
   ```

---

## **5. Monitoring dan Debugging**
### **a. Monitoring Squid:**
- Gunakan log untuk memantau cache hits/misses:
  ```bash
  tail -f /var/log/squid/access.log
  ```

### **b. Monitoring Nginx:**
- Lihat cache status dengan header `X-Cache-Status`:
  ```bash
  curl -I http://cdn.local/somefile.mp4
  ```

### **c. Tools untuk Monitoring:**
- **Grafana + Prometheus** untuk visualisasi trafik dan performa server.
- Gunakan **iftop** atau **htop** untuk memantau konsumsi bandwidth.

---

## **6. Kesimpulan**
Kombinasi Squid dan Nginx menciptakan solusi CDN lokal yang handal:
- **Squid:** Fokus pada caching granular untuk file kecil dan analisis trafik.
- **Nginx:** Melayani file besar dan konten statis dengan performa tinggi.

### **Keunggulan Kombinasi:**
- Menghemat bandwidth internet.
- Memberikan pengalaman pengguna yang cepat dengan konten lokal.
- Fleksibel untuk kebutuhan caching beragam.

Jika Anda memiliki pertanyaan lebih lanjut tentang konfigurasi tertentu, jangan ragu untuk bertanya! 😊# membuat-proxy-caching
