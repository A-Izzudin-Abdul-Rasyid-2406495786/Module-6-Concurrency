# Rust Web Server Reflection Notes

## Commit 1 Reflection notes
Pada tahap ini, fungsi `handle_connection` ditambahkan untuk membaca dan mencetak *request* HTTP yang dikirimkan oleh *browser* melalui protokol TCP. Kita menggunakan `BufReader` untuk membungkus `TcpStream` agar pembacaan data menjadi lebih efisien dengan sistem *buffering*, alih-alih membaca byte demi byte secara langsung. Request HTTP diurai baris demi baris menggunakan kombinasi *method* `.lines()`, `.map()`, dan `.take_while()` hingga menemukan baris kosong yang menandakan akhir dari metadata HTTP *header*. 

Melihat *raw output* dari request ini memberikan pemahaman visual yang sangat jelas tentang bagaimana protokol web bekerja di bawah kapur. Berbeda dengan *framework backend* modern yang langsung menyajikan data dalam bentuk objek atau *routing* yang sudah rapi, di sini kita benar-benar membedah *string* mentah yang berisi metode HTTP (`GET`), *path* (`/`), versi HTTP, hingga informasi *User-Agent*. Proses ini sangat mirip dengan bagaimana kita menangani *file descriptor* atau *socket* mentah di tingkat sistem operasi, di mana kita bertanggung jawab penuh atas setiap *byte* yang masuk dan keluar dari server.

---