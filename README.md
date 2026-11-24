# Coroutine Demo (M17 – Kotlin Coroutines & Asynchronous Programming)

## Cara memakai
1. Buat proyek **Empty Activity** bernama `CoroutineDemo`, package `com.example.coroutinedemo`.
2. Salin folder `data/`, `ui/main/`, dan `MainActivity.kt` dari zip ini ke package tersebut
   (timpa `MainActivity.kt` bawaan). Folder `ui/theme` tetap milik template.
3. Tambahkan dependency (modul Bagian 7.3), lalu **Sync Now**:

`gradle/libs.versions.toml` (paling bawah)
```toml
[libraries.androidx-lifecycle-viewmodel-compose]
group = "androidx.lifecycle"
name = "lifecycle-viewmodel-compose"
version.ref = "lifecycleRuntimeKtx"

[libraries.androidx-lifecycle-runtime-compose]
group = "androidx.lifecycle"
name = "lifecycle-runtime-compose"
version.ref = "lifecycleRuntimeKtx"
```
`app/build.gradle.kts`
```kotlin
dependencies {
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.lifecycle.runtime.compose)
}
```

## Pengembangan dari modul
- Latihan 1: `getCourseCount()` + `async` ketiga + field `courseCount`
- Latihan 2: tombol **Retry** di samping pesan error
- Latihan 6: tombol **Sort A-Z** (`withContext(Dispatchers.Default)`)
- Tambahan: 5 data tugas (agar sort terlihat), teks **"Loaded in … ms"** untuk membuktikan
  pemuatan konkuren, dan jumlah tugas di atas daftar.

## Jawaban Latihan

**1. Apakah total waktu masih ±2 detik?** Ya. Tiga `async` berjalan bersamaan, jadi total waktu
= pekerjaan terlama = max(2000, 1500, 1000) = ±2 detik. Kalau dipanggil satu per satu totalnya
2000 + 1000 + 1500 = 4,5 detik. (Lihat teks "Loaded in … ms" di layar.)

**2. Retry.** Sudah diimplementasikan di `TasksScreen.kt`. Tombol hanya tampil bila
`state.errorMessage != null` dan memanggil `loadData()`. Saat loading dimulai, `errorMessage`
direset ke `null`, sehingga tombol hilang sendiri.

**3. Mengganti `catch (e: IOException)` menjadi `catch (e: Exception)`.**
Ubah blok catch di `loadData()` menjadi:
```kotlin
} catch (e: Exception) {
    _uiState.update { it.copy(isLoading = false, errorMessage = "Failed to load: ${e.message}") }
}
```
Tekan Load lalu Cancel. Pesan "Loading cancelled" langsung tertimpa menjadi
"Failed to load: …" (isinya pesan dari exception pembatalan, mis. "StandaloneCoroutine was cancelled").
Penyebabnya: `Job.cancel()` membuat `await()` melempar `CancellationException`, yang merupakan
subclass `Exception`, sehingga ikut tertangkap dan dianggap kegagalan biasa. Perbaikan:
```kotlin
} catch (e: CancellationException) {
    throw e          // wajib dilempar ulang
} catch (e: Exception) {
    _uiState.update { it.copy(isLoading = false, errorMessage = "Failed to load: ${e.message}") }
}
```
Cara paling aman tetap menangkap exception yang spesifik (`IOException`).

**4. Di mana macetnya?** Ubah `getTasks()` pada percobaan:
```kotlin
// (a) Thread.sleep di dalam withContext(IO)
withContext(Dispatchers.IO) { Thread.sleep(2000L); ... }   // layar TIDAK macet
// (b) tanpa withContext
suspend fun getTasks(...) { Thread.sleep(2000L); ... }       // layar MACET ±2 detik
```
(a) `Thread.sleep` memblokir thread IO, bukan main thread, jadi "Tap me" tetap responsif.
(b) Tanpa `withContext`, coroutine berjalan di `Dispatchers.Main` (bawaan `viewModelScope`),
sehingga `Thread.sleep` memblokir main thread: tombol tidak bereaksi dan spinner tidak berputar.
Itulah gunanya fungsi main-safe: repository yang bertanggung jawab memindahkan kerja berat ke IO.

**5. Konsep.**
(a) `launch` menghasilkan `Job` dan tidak mengembalikan nilai (cocok untuk "jalankan saja").
`async` menghasilkan `Deferred<T>`; nilainya diambil dengan `await()` (cocok untuk beberapa
pekerjaan paralel yang hasilnya dibutuhkan).
(b) `GlobalScope` tidak terikat lifecycle apa pun dan tidak pernah berhenti selama aplikasi
hidup. Coroutine tetap berjalan walau layar/ViewModel sudah hilang, sehingga boros resource,
bisa memperbarui UI yang sudah tidak ada, dan memicu kebocoran. Pakai `viewModelScope` dll.
(c) Karena `getAnnouncement()` adalah sibling dari `getTasks()` di dalam `coroutineScope`,
ia ikut **dibatalkan** saat `getTasks()` gagal (aturan ke-3 structured concurrency). Errornya
diteruskan ke `try/catch` di luar `coroutineScope`.

**6. Sort.** `sortTasks()` di ViewModel mengurutkan dengan `withContext(Dispatchers.Default)`,
lalu memperbarui state di Main. Tombolnya ada di atas daftar tugas.
