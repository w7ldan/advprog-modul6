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
