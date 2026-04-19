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

## Commit 4 Reflection notes
Simulasi respons lambat dilakukan dengan menambahkan *endpoint* `/sleep` yang akan memaksa *thread* untuk berhenti sementara selama 10 detik menggunakan `thread::sleep`. Karena arsitektur server kita saat ini masih menggunakan model *single-threaded*, mengeksekusi *endpoint* ini akan sepenuhnya memblokir eksekusi program. Akibatnya, jika ada pengguna lain yang mencoba mengakses server (misalnya ke *endpoint* `/` biasa) saat server sedang "tertidur", permintaan pengguna tersebut akan menggantung dan tidak diproses sampai proses *sleep* selesai.

Fenomena ini adalah contoh nyata dari *concurrency bottleneck* yang sangat fatal jika dibiarkan di lingkungan produksi. Operasi I/O yang berat atau *query* database yang lambat pada satu *request* dapat melumpuhkan seluruh layanan web. Hal ini menunjukkan dengan jelas mengapa arsitektur asinkron (*asynchronous*) atau *multithreading* merupakan standar mutlak yang harus diterapkan pada *backend* untuk memastikan skalabilitas dan daya tanggap (*responsiveness*) sistem ketika menghadapi lonjakan pengguna secara bersamaan.

---

## Commit 5 Reflection notes

Untuk mengatasi masalah pemblokiran di komit sebelumnya, kita mengimplementasikan `ThreadPool` untuk membuat server menjadi *multithreaded*. Daripada membuat *thread* baru untuk setiap koneksi yang masuk—yang bisa menguras memori dan menyebabkan serangan DoS (*Denial of Service*) jika ada jutaan *request*—kita membatasi jumlah *thread* pekerja (*worker*) pada kapasitas tertentu. Setiap koneksi baru akan dikirim sebagai *job* ke dalam antrean (melalui *channel* `mpsc`), dan *worker* yang sedang menganggur akan mengambil pekerjaan tersebut.

Implementasi di Rust sangat menarik karena kita dipaksa memikirkan keamanan memori (*memory safety*) saat berbagi *state* antar *thread*. Kita harus menggunakan `Arc<Mutex<Receiver>>` agar *receiver* dari *channel* dapat dimiliki oleh banyak *worker* sekaligus (`Arc`), namun memastikan hanya ada satu *worker* yang dapat menarik pekerjaan dari antrean pada satu waktu secara eksklusif (`Mutex`). Model ini mengajarkan fundamental konkurensi yang sangat solid, memastikan tidak ada *data race* sekaligus menjaga performa server tetap stabil dan responsif di bawah beban kerja yang tinggi.

---

## Commit Bonus Reflection notes
Fungsi `new` di Rust secara konvensi diharapkan selalu berhasil menginisialisasi objek atau langsung *panic* jika terjadi kegagalan fatal. Namun, untuk `ThreadPool`, pengguna bisa saja memasukkan angka `0` sebagai jumlah kapasitas *thread*, yang mana hal tersebut tidak masuk akal dan memicu *panic*. Membuat library yang secara sepihak mematikan program utama adalah praktik desain perangkat lunak yang buruk. 

Sebagai gantinya, kita membuat fungsi `build` yang mengembalikan tipe `Result<ThreadPool, PoolCreationError>`. Dengan menggunakan `Result`, kita memindahkan tanggung jawab penanganan *error* kembali ke pemanggil fungsi (*caller*). Jika inisialisasi gagal karena parameter yang tidak valid, program tidak akan langsung *crash*, melainkan memberikan kesempatan bagi developer untuk menangani kesalahan tersebut dengan lebih anggun (*graceful error handling*), misalnya dengan memberikan log peringatan atau mencoba nilai *default*. Ini adalah bentuk perbaikan fungsionalitas yang sejalan dengan filosofi *robustness* dari bahasa Rust.