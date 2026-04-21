# Module 6

Refleksi 1

1. **Penjelasan Alur `handle_connection` dan Modifikasi Kode:**
   Menurut saya, method `handle_connection` dirancang dengan sangat spesifik untuk menangani koneksi protokol TCP dari browser. 
   Di dalam blok kodenya, `BufReader` bertugas membungkus referensi mutable yang masuk menuju `TcpStream`. Mengapa kita butuh `BufReader`? Karena `BufReader` akan melakukan *buffering* pada jalur stream tersebut, sehingga dapat meminimalkan pemanggilan iteratif pada *system calls* secara langsung serta mendongkrak efisiensi operasi I/O. Berkat implementasi ini, kita dapat membaca aliran *stream* sebaris demi sebaris (*line by line*).

2. **Pembentukan Representasi HTTP Request:**
   Dalam fungsi ini, kita juga mencetak variabel `http_request` yang bertipe `Vec<String>`. Logika ini dikonfigurasikan lewat sejumlah *method chaining*:
   - Pertama, `buf_reader.lines()` menginisiasikan *iterator* secara sekuensial pada sekumpulan request tersebut.
   - Kemudian, kita menggunakan `.map(|result| result.unwrap())` untuk mengekstrak tipe data `Result` menjadi nilai *raw* `String` murni untuk masing-masing baris *request*.
   - Selanjutnya, terdapat perintah `.take_while(|line| !line.is_empty())`. Method ini memerintahkan agar sistem hanya akan mengambil baris bilamana string-nya tidak kosong. Hal ini diterapkan karena pada struktur protokol HTTP, baris yang kosong merupakan penanda *end-of-headers* dari bagian *headers* pada HTTP request.
   - Pada akhirnya, `.collect()` bertugas untuk mengonsumsi keseluruhan elemen di dalam *iterator* hingga batas `.take_while` tadi tercapai, dan menyimpannya ke tipe `Vec<_>`.
   
   Pada baris terakhir, program menggunakan `println!` untuk mem-print `http_request` ke layar terminal console. Hasil tampilannya mempermudah kita saat melakukan inspeksi metadaata *request* dari sisi browser klien.

---

Refleksi 2

1. **Pengembalian HTML dan Peran `Content-Length`:**
   Pada commit ini, saya melakukan modifikasi pada method `handle_connection` agar server mampu merespons dengan me-return file HTML sederhana (`hello.html`) ke browser pengguna.
   Pertama, fungsionalitas membaca file diatur melalui `fs::read_to_string` yang mengubah HTML mentah menjadi tipe data `String`. 
   Lalu ditambahkan *header* `Content-Length`. Atribut ini berperan wajib dalam protokol transmisi untuk memberi tahu *client browser* mengenai ukuran aktual (dalam bytes via `contents.len()`) dari *response body* yang dikirimkan. Berkat *header* presisi ini, browser klien dapat menjamin kepastian bahwa keseluruhan data *response* sudah diterima dengan paripurna sebelum memutus aliran datanya.
   Langkah terakhir dijalankan dengan merangkai *status line*, *header* `Content-Length`, dan *body* ke dalam `format!()`, lalu ditransmisikan kembali lewat `stream.write_all()` dengan mengonversinya menjadi *byte array* mentah (`as_bytes()`).
   ![Commit 2 screen capture](/assets/images/commit2.png)
