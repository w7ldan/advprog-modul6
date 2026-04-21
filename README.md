# Module 6

Refleksi 1

1. **Penjelasan Alur `handle_connection` dan Modifikasi Kode:**
   Menurut saya, method `handle_connection` dirancang dengan sangat spesifik untuk menangani koneksi protokol TCP dari browser. 
   Di dalam blok kodenya, `BufReader` bertugas membungkus referensi mutable yang masuk menuju `TcpStream`. Mengapa saya butuh `BufReader`? Karena `BufReader` akan melakukan *buffering* pada jalur stream tersebut, sehingga dapat meminimalkan pemanggilan iteratif pada *system calls* secara langsung serta mendongkrak efisiensi operasi I/O. Berkat implementasi ini, saya dapat membaca aliran *stream* sebaris demi sebaris (*line by line*).

2. **Pembentukan Representasi HTTP Request:**
   Dalam fungsi ini, saya juga mencetak variabel `http_request` yang bertipe `Vec<String>`. Logika ini dikonfigurasikan lewat sejumlah *method chaining*:
   - Pertama, `buf_reader.lines()` menginisiasikan *iterator* secara sekuensial pada sekumpulan request tersebut.
   - Kemudian, saya menggunakan `.map(|result| result.unwrap())` untuk mengekstrak tipe data `Result` menjadi nilai *raw* `String` murni untuk masing-masing baris *request*.
   - Selanjutnya, terdapat perintah `.take_while(|line| !line.is_empty())`. Method ini memerintahkan agar sistem hanya akan mengambil baris bilamana string-nya tidak kosong. Hal ini diterapkan karena pada struktur protokol HTTP, baris yang kosong merupakan penanda *end-of-headers* dari bagian *headers* pada HTTP request.
   - Pada akhirnya, `.collect()` bertugas untuk mengonsumsi keseluruhan elemen di dalam *iterator* hingga batas `.take_while` tadi tercapai, dan menyimpannya ke tipe `Vec<_>`.
   
   Pada baris terakhir, program menggunakan `println!` untuk mem-print `http_request` ke layar terminal console. Hasil tampilannya mempermudah saya saat melakukan inspeksi metadaata *request* dari sisi browser klien.

Refleksi 2

1. **Pengembalian HTML dan Peran `Content-Length`:**
   Pada commit ini, saya melakukan modifikasi pada method `handle_connection` agar server mampu merespons dengan me-return file HTML sederhana (`hello.html`) ke browser pengguna.
   Pertama, fungsionalitas membaca file diatur melalui `fs::read_to_string` yang mengubah HTML mentah menjadi tipe data `String`. 
   Lalu ditambahkan *header* `Content-Length`. Atribut ini berperan wajib dalam protokol transmisi untuk memberi tahu *client browser* mengenai ukuran aktual (dalam bytes via `contents.len()`) dari *response body* yang dikirimkan. Berkat *header* presisi ini, browser klien dapat menjamin kepastian bahwa keseluruhan data *response* sudah diterima dengan paripurna sebelum memutus aliran datanya.
   Langkah terakhir dijalankan dengan merangkai *status line*, *header* `Content-Length`, dan *body* ke dalam `format!()`, lalu ditransmisikan kembali lewat `stream.write_all()` dengan mengonversinya menjadi *byte array* mentah (`as_bytes()`).
   ![Commit 2 screen capture](/assets/images/commit2.png)

Refleksi 3

1. **Pemisahan Response dan Validasi Request:**
   Pada pencapaian commit ini, saya mengembangkan sebuah logika *validation* yang sederhana untuk mengarahkan ke mana *request* harus diproses. Langkah utama diaplikasikan dengan memilah baris pertama dari HTTP request (`request_line`) sebagai acuan. Jika pengunjung mengarah ke root `/` lewat `"GET / HTTP/1.1"`, maka aplikasi akan mengalokasikannya dengan isi file `"hello.html"` dengan *status code* bernilai `"HTTP/1.1 200 OK"`. Sebaliknya, apabila ia mencari URL yang tidak valid (semisal ke `/bad`), muatannya secara adaptif akan mendarat di blok eksekusi `else`, dan direkomendasikan menuju file `"404.html"` dengan sisipan *status code* pengembalian `"HTTP/1.1 404 NOT FOUND"`.

2. **Dampak Signifikan Refactoring pada Basis Kode:**
   Sebelum refactoring ini dilakukan, desain mentah program memiliki duplikasi *boilerplate code* lantaran fungsionalitas `fs::read_to_string`, mekanisme ukurannya (`.len()`), *macro* `format!()`, serta metode pengiriman datanya (`stream.write_all()`) terpaksa ditulis berulang di dalam blok `if` maupun `else`. Ini sangat tidak efisien dan melanggar praktik arsitektur program yang baik.
   Oleh karena itu, dijalankan iterasi *refactoring*. *Refactoring* dicapai di mana perbedaan variabel tersebut difokuskan dan diubah jadi tipe Tuple `(status_line, filename)`. Nilai evaluasi blok `if` dipakai sepenuhnya untuk mengisi konfigurasi Tuple ini. Lalu, logika utama untuk membaca isi HTML dan menulis *response* akhirnya berhasil diekstraksi ke luar blok pemisah `if-else` menjadi kode mandiri yang hanya ditulis satu kali. Hal ini merupakan perwujudan asas pengembangan dari paham DRY (Don't Repeat Yourself), menjadikan kualitas kode jauh lebih bersih meramping, tangguh dan terpusat (*maintainable*).

   ![Commit 3 screen capture](/assets/images/commit3.png)

Refleksi 4

1. **Simulasi Respons Lambat (*Slow Response Simulation*):**
   Pada simulasi ini, saya mengintegrasikan modul `thread` dan `Duration` yang merupakan fitur *core library* bawaan bahasa Rust. Tujuannya adalah untuk meniru *endpoint* lambat yang memakan banyak waktu pemrosesan (*blocking processing*). Saya membuat simulasi melalui *endpoint* `/sleep` yang di dalamnya terpasang kode `thread::sleep(Duration::from_secs(10))`. Jika rute ini dikunjungi, eksekusi alur aplikasi di *thread* utama akan diinterupsi dan ditahan paksa selama 10 detik penuh sebelum akhirnya *server* membalas pengembalian `"hello.html"`.

2. **Problem Arsitektur *Single-Threaded*:**
   Efek yang langsung terasa akibat *endpoint* `/sleep` ini rupanya berdampak sangat masif pada visibilitas proyek! Karena *server* saya masih berjalan pada arus tunggal (*single-threaded*), segala komputasi *request* membonceng antrean komputasi secara linier. Saat browser *(tab spesifik)* memuat rute simulasi `/sleep`, selurh arsitektur *server* seketika divonis "tersandera" (*blocked*/*stuck*). Berapa pun jumlah jendela atau rute tab normal lain yang memuat halaman `/` secara bersamaan, mereka semua terpaksa menggantung terus (*loading state*) hingga 10 detiknya ditutup.
   Inilah kelemahan dan cacat absolut sistem web *single-thread*. Di level iterasi dan sistem sesungguhnya (*production*), metode konvensional ini sungguh rentan, *bottle-neck* yang parah pada *throughput*, dan *non-scalable* bila dieksploitasi untuk pengaksesan data secara serentak (*concurrent execution*).

Refleksi 5

1. **Pemahaman tentang `ThreadPool`:**
   Saya mengembangkan struktur `ThreadPool` *(thread pool)* untuk memecahkan kelemahan dari *single-threaded server*. Idenya adalah dibandingkan dengan menciptakan *thread* secara bar-bar *unlimited* setiap kali server menerima *request* baru (yang rentan diserang dan menghabiskan *resource* OS seperti pada kasus *DDoS attack*), lebih baik saya membatasi alokasi ketersediaan *threads*. Di sini, `ThreadPool` diinstansiasi dengan kapasitas konstan `size` bernilai 4. 
   Setiap entitas `Worker` yang dibuat akan menjalankan *infinite loop* untuk mendengarkan pesanan di dalam jalurnya secara konstan. Begitu ada permintaan (*loop incoming connection* melakukan `execute`), *closure* logika penanganan (*Job*) akan diforward via *channel sender*. Antrean aliran pesan (*channel receiver*) tersebut direngkuh menggunakan payung `Arc` (*Atomic Reference Counting* untuk me-*share ownership*) dikombinasikan dengan perlindungan `Mutex` agar serangkaian siklus pergantian antar *multiple worker* ini terlindungi dan tidak bertabrakan (*race conditions*). 
   Saat ini server mampu mendelegasikan beban pada jalur `/sleep` ke satu *thread worker* saja, sambil membebaskan 3 *thread* sisanya bagi interaksi pengguna lain yang masuk tanpa halangan. *Throughput* program saya juga menjadi stabil dan kebal terhadap *bottleneck*.
