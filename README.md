# Rust Web Server Reflection Notes

## Commit 1 Reflection notes
Pada tahap ini, fungsi `handle_connection` ditambahkan untuk membaca dan mencetak *request* HTTP yang dikirimkan oleh *browser* melalui protokol TCP. Kita menggunakan `BufReader` untuk membungkus `TcpStream` agar pembacaan data menjadi lebih efisien dengan sistem *buffering*, alih-alih membaca byte demi byte secara langsung. Request HTTP diurai baris demi baris menggunakan kombinasi *method* `.lines()`, `.map()`, dan `.take_while()` hingga menemukan baris kosong yang menandakan akhir dari metadata HTTP *header*. 

Melihat *raw output* dari request ini memberikan pemahaman visual yang sangat jelas tentang bagaimana protokol web bekerja di bawah kapur. Berbeda dengan *framework backend* modern yang langsung menyajikan data dalam bentuk objek atau *routing* yang sudah rapi, di sini kita benar-benar membedah *string* mentah yang berisi metode HTTP (`GET`), *path* (`/`), versi HTTP, hingga informasi *User-Agent*. Proses ini sangat mirip dengan bagaimana kita menangani *file descriptor* atau *socket* mentah di tingkat sistem operasi, di mana kita bertanggung jawab penuh atas setiap *byte* yang masuk dan keluar dari server.


## Commit 2 Reflection notes
![Commit 2 screen capture](/assets/images/commit2.png) 

Mengembalikan file HTML ke *browser* mengharuskan kita merakit HTTP *response* secara manual sesuai dengan spesifikasi protokol yang ketat. Di dalam `handle_connection`, kita tidak hanya membaca file `hello.html` menggunakan `fs::read_to_string`, tetapi juga harus menyusun *status line* (`HTTP/1.1 200 OK`) dan *header* `Content-Length`. Pemformatan ini sangat sensitif; kita wajib menggunakan *carriage return* dan *line feed* (`\r\n\r\n`) yang tepat untuk memisahkan antara HTTP *header* dan bagian *body* (konten HTML itu sendiri).

Jika `Content-Length` tidak disertakan atau ukurannya tidak akurat, *browser* klien bisa mengalami kebingungan saat melakukan *parsing*, yang berpotensi memutus koneksi sebelum seluruh file selesai diunduh. Pengalaman menulis balasan HTTP secara eksplisit seperti ini memberikan perspektif baru tentang seberapa banyak pekerjaan repetitif yang biasanya disembunyikan oleh *web server*. Kita jadi lebih menghargai pentingnya abstraksi, namun juga memiliki bekal teknis yang kuat tentang cara kerja dasar pertukaran data web di tingkat jaringan.

---

## Commit 3 Reflection notes
![Commit 3 screen capture](/assets/images/commit3.png) 

Pada komit ini, kita mengimplementasikan *routing* sederhana untuk memvalidasi *request* dan memberikan respons yang berbeda (halaman sukses atau halaman 404 *Not Found*). Kita memisahkan antara blok *response* sukses dan blok *error*, namun pada awalnya hal ini menimbulkan banyak duplikasi kode, terutama pada bagian pembacaan file dan penulisan ke *stream*. Oleh karena itu, *refactoring* dilakukan untuk membuat kode menjadi lebih DRY (*Don't Repeat Yourself*).

Dengan *refactoring*, kita memanfaatkan `match` pada Rust untuk sekadar menentukan *status line* dan nama file HTML yang akan dieksekusi, lalu menyimpannya dalam variabel. Pemrosesan akhir—seperti membaca file, menghitung *length*, dan memformat *string response*—hanya ditulis satu kali di bagian akhir fungsi. Pola desain ini tidak hanya membuat kode lebih bersih dan mudah dibaca, tetapi juga sangat memudahkan pengembangan ke depannya jika kita ingin menambahkan ratusan *endpoint* baru, karena logika utama pengiriman balasan sudah terpusat di satu tempat.

---