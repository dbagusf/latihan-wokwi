---
layout: default
title: "IoT ESP32 dengan Wokwi Cheatsheet"
description: "Panduan ringkas belajar IoT: simulasi ESP32 di Wokwi, sensor, WiFi, MQTT, dan dasar keamanan IoT."
---

# IoT ESP32 + Wokwi Cheatsheet

Sumber belajar singkat untuk praktik **IoT tanpa perangkat fisik**. Berisi langkah kerja, kode siap salin, tabel pin, MQTT, troubleshooting, dan dasar keamanan IoT menggunakan simulator [Wokwi](https://wokwi.com/).

* TOC
{:toc}

---

## Mulai Cepat

**Wokwi** adalah simulator online untuk ESP32, Arduino, dan Raspberry Pi Pico. Rangkaian dan kode dijalankan langsung di browser, jadi aman untuk pemula: tidak ada risiko komponen terbakar.

### Alur kerja 5 langkah

1. **Buka** [wokwi.com](https://wokwi.com/) lalu login (gratis) agar proyek tersimpan.
2. **New Project** lalu pilih **ESP32** lalu pilih template **Arduino** atau **MicroPython**.
3. **Rangkai** komponen di panel diagram (tombol biru **+**).
4. **Tulis kode** di `sketch.ino`.
5. **Klik Start Simulation** (tombol hijau), lalu amati **Serial Monitor**.

### File dalam proyek Wokwi

| File | Fungsi |
| --- | --- |
| `sketch.ino` | Kode program utama |
| `diagram.json` | Daftar komponen dan kabel (bisa diedit sebagai teks) |
| `libraries.txt` | Daftar library yang dipasang otomatis |
| `wokwi.toml` | Konfigurasi bila dipakai di VS Code / CLI |

---

## Komponen Populer

| Komponen | Kegunaan | Pin tipikal ESP32 |
| --- | --- | --- |
| LED + resistor 220 ohm | Output digital | GPIO 2, 4, 5 |
| Pushbutton | Input digital | GPIO 18, 19 |
| Potensiometer | Input analog | GPIO 34 |
| DHT22 | Suhu dan kelembapan | GPIO 15 |
| HC-SR04 | Jarak ultrasonik | TRIG 5, ECHO 18 |
| LDR (photoresistor) | Intensitas cahaya | GPIO 35 |
| Servo | Aktuator gerak | GPIO 13 |
| Buzzer | Alarm | GPIO 12 |
| OLED SSD1306 | Layar I2C | SDA 21, SCL 22 |
| PIR | Deteksi gerak | GPIO 27 |

### Aturan pin ESP32 yang wajib diingat

* **GPIO 34 sampai 39** hanya untuk **input** (tanpa pull-up internal).
* **GPIO 6 sampai 11** terhubung ke flash internal, **jangan dipakai**.
* Saat WiFi aktif, gunakan **ADC1** (GPIO 32 sampai 39) untuk `analogRead`. ADC2 bentrok dengan WiFi.
* Pin strapping (0, 2, 12, 15) memengaruhi proses boot, hati-hati bila dipasang beban.
* Tegangan logika ESP32 adalah **3,3 V**, bukan 5 V.

---

## Kode Dasar

### 1. Blink LED

```cpp
const int LED_PIN = 2;

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(500);
  digitalWrite(LED_PIN, LOW);
  delay(500);
}
```

### 2. Tombol menyalakan LED

```cpp
const int BTN_PIN = 18;
const int LED_PIN = 2;

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);  // tombol aktif LOW
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  bool ditekan = (digitalRead(BTN_PIN) == LOW);
  digitalWrite(LED_PIN, ditekan);
}
```

### 3. Serial Monitor dan input analog

```cpp
void setup() {
  Serial.begin(115200);
}

void loop() {
  int nilai = analogRead(34);              // 0 - 4095
  float volt = nilai * 3.3 / 4095.0;
  Serial.printf("ADC=%d  Tegangan=%.2f V\n", nilai, volt);
  delay(500);
}
```

### 4. Sensor DHT22

Tambahkan ke `libraries.txt`:

```
DHT sensor library
```

```cpp
#include "DHT.h"
#define DHTPIN 15
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  dht.begin();
}

void loop() {
  float suhu = dht.readTemperature();
  float lembap = dht.readHumidity();
  if (isnan(suhu) || isnan(lembap)) {
    Serial.println("Gagal membaca DHT22");
  } else {
    Serial.printf("Suhu: %.1f C | Lembap: %.1f %%\n", suhu, lembap);
  }
  delay(2000);
}
```

---

## WiFi di Wokwi

Wokwi menyediakan jaringan virtual bernama **Wokwi-GUEST** tanpa password, sehingga ESP32 bisa terhubung ke internet dari simulasi.

```cpp
#include <WiFi.h>

void setup() {
  Serial.begin(115200);
  WiFi.begin("Wokwi-GUEST", "", 6);   // SSID, password kosong, channel 6
  Serial.print("Menghubungkan");
  while (WiFi.status() != WL_CONNECTED) {
    delay(250);
    Serial.print(".");
  }
  Serial.println("\nTerhubung! IP: " + WiFi.localIP().toString());
}

void loop() {}
```

**Catatan:** yang disimulasikan hanya WiFi. Bluetooth/BLE tidak tersedia di Wokwi.

---

## MQTT: Kirim Data Sensor

**MQTT** adalah protokol ringan berbasis publish/subscribe, standar umum pada IoT. Untuk latihan gunakan broker publik `broker.hivemq.com` port `1883`. Jangan kirim data sensitif ke broker publik.

`libraries.txt`:

```
PubSubClient
DHT sensor library
```

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include "DHT.h"

DHT dht(15, DHT22);
WiFiClient wifiClient;
PubSubClient mqtt(wifiClient);

const char* BROKER = "broker.hivemq.com";
const char* TOPIK  = "smk-demo/iot/suhu";   // ganti dengan topik unik Anda

void sambungMqtt() {
  while (!mqtt.connected()) {
    String id = "esp32-" + String(random(0xffff), HEX);
    if (!mqtt.connect(id.c_str())) delay(2000);
  }
}

void setup() {
  Serial.begin(115200);
  dht.begin();
  WiFi.begin("Wokwi-GUEST", "", 6);
  while (WiFi.status() != WL_CONNECTED) delay(250);
  mqtt.setServer(BROKER, 1883);
}

void loop() {
  if (!mqtt.connected()) sambungMqtt();
  mqtt.loop();

  float suhu = dht.readTemperature();
  if (!isnan(suhu)) {
    char pesan[16];
    dtostrf(suhu, 4, 1, pesan);
    mqtt.publish(TOPIK, pesan);
    Serial.printf("Publish %s = %s\n", TOPIK, pesan);
  }
  delay(5000);
}
```

### Memantau hasilnya

* Buka klien web seperti [HiveMQ Web Client](https://www.hivemq.com/demos/websocket-client/), lalu **Subscribe** ke topik yang sama.
* Atau gunakan aplikasi MQTT Explorer / MQTTX di komputer.

### Tips

* Gunakan **topik unik** (misal awali dengan nama sekolah) agar tidak tercampur data orang lain.
* Struktur topik yang rapi: `lokasi/perangkat/sensor`.
* Kirim data secukupnya (misal tiap 5 detik), bukan tanpa jeda.

---

## Keamanan IoT (Dasar)

Perangkat IoT sering menjadi titik lemah jaringan. Biasakan praktik aman sejak simulasi.

| Risiko | Contoh kesalahan | Praktik yang benar |
| --- | --- | --- |
| Kredensial default | Password `admin` / `1234` | Ganti password, gunakan kredensial unik per perangkat |
| Kredensial tertanam | SSID dan password ditulis di kode yang diunggah publik | Pisahkan ke file `secrets.h` dan masukkan ke `.gitignore` |
| Komunikasi tanpa enkripsi | MQTT port 1883, HTTP | Gunakan **MQTT over TLS (port 8883)** dan HTTPS |
| Tanpa autentikasi | Broker terbuka untuk siapa saja | Aktifkan username/password atau sertifikat klien |
| Firmware tidak diperbarui | Tidak ada mekanisme update | Rencanakan update firmware yang tertandatangani (OTA aman) |
| Data berlebihan | Mengirim data pribadi tanpa perlu | Kumpulkan data seminimal mungkin |

### Memisahkan kredensial

```cpp
// secrets.h  (jangan di-commit ke GitHub!)
#define WIFI_SSID "NamaWiFi"
#define WIFI_PASS "passwordwifi"
```

```cpp
#include "secrets.h"
WiFi.begin(WIFI_SSID, WIFI_PASS);
```

### Referensi keamanan

* [OWASP IoT Top 10](https://owasp.org/www-project-internet-of-things/) - daftar risiko utama IoT.
* [OWASP IoT Security Testing Guide](https://owasp.org/www-project-iot-security-testing-guide/) - panduan pengujian.

**Etika:** latihan hanya pada simulasi atau perangkat milik sendiri. Jangan menguji perangkat atau broker milik orang lain tanpa izin tertulis.

---

## Troubleshooting

| Masalah | Kemungkinan penyebab | Solusi |
| --- | --- | --- |
| Serial Monitor kosong | Baud rate tidak sama | Samakan `Serial.begin(115200)` |
| LED tidak menyala | Polaritas terbalik / pin salah | Anoda ke pin GPIO (lewat resistor), katoda ke GND |
| Tombol selalu terbaca LOW/HIGH | Lupa pull-up | Gunakan `INPUT_PULLUP`, sambungkan sisi lain ke GND |
| DHT22 hasil `nan` | Pin data atau library salah | Cek pin, pastikan library ada di `libraries.txt` |
| WiFi tidak tersambung | SSID/channel salah | Gunakan `WiFi.begin("Wokwi-GUEST", "", 6)` |
| MQTT gagal connect | Broker penuh atau ID klien sama | Gunakan client ID acak, coba lagi |
| `analogRead` tidak stabil saat WiFi aktif | Memakai pin ADC2 | Pindah ke pin ADC1 (32 sampai 39) |
| Library tidak ditemukan | Belum terdaftar | Tambahkan nama library di `libraries.txt` atau lewat tombol Library Manager |
| Simulasi lambat | Terlalu banyak `delay` / komponen | Kurangi komponen, gunakan `millis()` |

---

## Latihan Mandiri

Latihan lengkap dengan peta pin, kode, dan rubrik ada di halaman [Latihan Rangkaian ESP32](latihan.html).

1. **Lampu lalu lintas:** 3 LED (merah, kuning, hijau) berganti dengan pola waktu.
2. **Alarm suhu:** buzzer berbunyi bila suhu DHT22 lebih dari 35 C.
3. **Lampu otomatis:** LDR menyalakan LED saat gelap.
4. **Dasbor MQTT:** kirim suhu dan kelembapan ke dua topik terpisah, tampilkan di klien MQTT.
5. **Versi aman:** ubah proyek nomor 4 agar kredensial tidak tertanam di kode dan bahas risiko broker publik.

---

## Sumber Belajar

* **[Wokwi](https://wokwi.com/)** - Simulator online.
* **[Dokumentasi Wokwi](https://docs.wokwi.com/)** - Panduan komponen, WiFi, dan VS Code.
* **[Arduino-ESP32](https://docs.espressif.com/projects/arduino-esp32/en/latest/)** - Dokumentasi resmi Espressif.
* **[PubSubClient](https://pubsubclient.knolleary.net/)** - Library MQTT untuk Arduino.
* **[HiveMQ](https://www.hivemq.com/mqtt/)** - Pengantar MQTT.
* **[OWASP IoT](https://owasp.org/www-project-internet-of-things/)** - Keamanan IoT.

---

[Kembali ke atas](#top)
