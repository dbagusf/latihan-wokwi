---
layout: default
title: "Latihan Rangkaian ESP32 di Wokwi"
description: "6 latihan bertingkat: LED, pushbutton, potensiometer, HC-SR04, MQ2, DHT22, buzzer, servo, dan relay."
---

# Latihan Rangkaian ESP32 di Wokwi

Enam latihan bertingkat, dari dasar sampai proyek terpadu. Setiap latihan berisi **tujuan, rangkaian, kode, dan tantangan**. Kembali ke [Cheatsheet utama](index.html).

* TOC
{:toc}

---

## Peta Pin (dipakai di semua latihan)

Gunakan pin yang sama agar rangkaian bisa dikembangkan tanpa bongkar ulang.

| Komponen | Pin komponen | Pin ESP32 | Catatan |
| --- | --- | --- | --- |
| LED merah + resistor 220 ohm | Anoda (lewat resistor) | GPIO 4 | Katoda ke GND |
| LED hijau + resistor 220 ohm | Anoda (lewat resistor) | GPIO 5 | Katoda ke GND |
| Pushbutton | Kaki 1 | GPIO 18 | Kaki 2 ke GND, pakai `INPUT_PULLUP` |
| Potensiometer | SIG | GPIO 34 | Ujung ke 3V3 dan GND |
| HC-SR04 | TRIG / ECHO | GPIO 26 / 27 | VCC ke **VIN (5V)**, GND ke GND |
| MQ2 (gas sensor) | AOUT | GPIO 35 | VCC ke 3V3 (simulasi), GND ke GND |
| DHT22 | SDA/DATA | GPIO 15 | VCC ke 3V3, GND ke GND |
| Buzzer | Pin 1 / Pin 2 | GPIO 25 / GND | |
| Servo | PWM / V+ / GND | GPIO 13 / VIN / GND | |
| Relay module | IN / VCC / GND | GPIO 14 / VIN / GND | Beban dipasang di COM dan NO |

**Peringatan perangkat nyata:** pin ECHO HC-SR04 mengeluarkan 5 V, sedangkan ESP32 hanya toleran 3,3 V. Pada rangkaian fisik gunakan **pembagi tegangan** (misal 1 kohm dan 2 kohm). Banyak modul relay bersifat **aktif LOW**, jadi cek datasheet modul Anda.

---

## Latihan 1: Saklar Toggle (LED + Pushbutton)

**Tujuan:** memahami input digital, `INPUT_PULLUP`, dan deteksi perubahan tombol.

**Komponen:** ESP32, LED merah, resistor 220 ohm, pushbutton.

```cpp
const int LED = 4, BTN = 18;
bool nyala = false, terakhir = HIGH;

void setup() {
  pinMode(LED, OUTPUT);
  pinMode(BTN, INPUT_PULLUP);
}

void loop() {
  bool sekarang = digitalRead(BTN);
  if (terakhir == HIGH && sekarang == LOW) {   // tepi turun = tombol baru ditekan
    nyala = !nyala;
    digitalWrite(LED, nyala);
    delay(30);                                 // debounce sederhana
  }
  terakhir = sekarang;
}
```

**Tantangan:** tambahkan LED hijau yang menyala saat LED merah mati.

---

## Latihan 2: Kendali Servo dengan Potensiometer

**Tujuan:** membaca ADC dan memetakan nilainya ke sudut servo.

**Komponen:** ESP32, potensiometer, servo.

`libraries.txt`:

```
ESP32Servo
```

```cpp
#include <ESP32Servo.h>
Servo servo;

void setup() {
  Serial.begin(115200);
  servo.attach(13);
}

void loop() {
  int adc = analogRead(34);              // 0 - 4095
  int sudut = map(adc, 0, 4095, 0, 180);
  servo.write(sudut);
  Serial.printf("ADC=%d  Sudut=%d\n", adc, sudut);
  delay(50);
}
```

**Tantangan:** batasi gerak servo hanya 30 sampai 150 derajat, lalu nyalakan LED merah bila di luar rentang 60 sampai 120.

---

## Latihan 3: Sensor Parkir (HC-SR04 + Buzzer + LED)

**Tujuan:** mengukur jarak dan memberi peringatan bertingkat.

**Komponen:** ESP32, HC-SR04, buzzer, LED merah, LED hijau, 2 resistor 220 ohm.

```cpp
const int TRIG = 26, ECHO = 27, BUZ = 25, LED_M = 4, LED_H = 5;

float bacaJarak() {
  digitalWrite(TRIG, LOW);  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH); delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long t = pulseIn(ECHO, HIGH, 30000);   // timeout 30 ms
  if (t == 0) return 999;                // tidak ada pantulan
  return t * 0.0343 / 2;                 // cm
}

void beep(int ms) {                      // nada sekitar 2 kHz tanpa library
  for (long i = 0; i < ms * 2L; i++) {
    digitalWrite(BUZ, HIGH); delayMicroseconds(250);
    digitalWrite(BUZ, LOW);  delayMicroseconds(250);
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(TRIG, OUTPUT); pinMode(ECHO, INPUT);
  pinMode(BUZ, OUTPUT);
  pinMode(LED_M, OUTPUT); pinMode(LED_H, OUTPUT);
}

void loop() {
  float d = bacaJarak();
  Serial.printf("Jarak: %.1f cm\n", d);
  digitalWrite(LED_H, d >= 50);
  digitalWrite(LED_M, d < 50);
  if (d < 20)      { beep(100); delay(50); }    // bahaya: bunyi cepat
  else if (d < 50) { beep(100); delay(400); }   // waspada: bunyi lambat
  else             { delay(200); }
}
```

**Tantangan:** buat jeda bunyi berbanding lurus dengan jarak, sehingga makin dekat makin cepat.

---

## Latihan 4: Termostat Otomatis (DHT22 + Relay)

**Tujuan:** kontrol on/off dengan **histeresis** agar relay tidak bergetar.

**Komponen:** ESP32, DHT22, relay module, LED merah, LED hijau (beban relay bisa diwakili LED di sisi COM/NO).

`libraries.txt`:

```
DHT sensor library
```

```cpp
#include "DHT.h"
DHT dht(15, DHT22);
const int RELAY = 14, LED_M = 4, LED_H = 5;
const float BATAS_ATAS = 30.0, BATAS_BAWAH = 28.0;
bool kipas = false;

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(RELAY, OUTPUT); pinMode(LED_M, OUTPUT); pinMode(LED_H, OUTPUT);
}

void loop() {
  float suhu = dht.readTemperature();
  if (!isnan(suhu)) {
    if (suhu > BATAS_ATAS) kipas = true;
    else if (suhu < BATAS_BAWAH) kipas = false;
    digitalWrite(RELAY, kipas);
    digitalWrite(LED_M, kipas);
    digitalWrite(LED_H, !kipas);
    Serial.printf("Suhu %.1f C | Kipas %s\n", suhu, kipas ? "ON" : "OFF");
  }
  delay(2000);
}
```

**Tantangan:** tambahkan kelembapan, dan nyalakan kipas juga bila kelembapan lebih dari 80 persen.

---

## Latihan 5: Alarm Gas (MQ2 + Buzzer + Relay)

**Tujuan:** membaca sensor gas analog dan memicu alarm serta exhaust fan.

**Komponen:** ESP32, MQ2, buzzer, relay, LED merah, LED hijau.

```cpp
const int MQ2 = 35, BUZ = 25, RELAY = 14, LED_M = 4, LED_H = 5;
const int AMBANG = 2000;   // sesuaikan lewat slider gas di Wokwi

void beep(int ms) {
  for (long i = 0; i < ms * 2L; i++) {
    digitalWrite(BUZ, HIGH); delayMicroseconds(250);
    digitalWrite(BUZ, LOW);  delayMicroseconds(250);
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(BUZ, OUTPUT); pinMode(RELAY, OUTPUT);
  pinMode(LED_M, OUTPUT); pinMode(LED_H, OUTPUT);
}

void loop() {
  int gas = analogRead(MQ2);
  bool bahaya = gas > AMBANG;
  digitalWrite(RELAY, bahaya);          // exhaust fan
  digitalWrite(LED_M, bahaya);
  digitalWrite(LED_H, !bahaya);
  if (bahaya) beep(200);
  Serial.printf("Gas ADC=%d  Status=%s\n", gas, bahaya ? "BAHAYA" : "AMAN");
  delay(300);
}
```

**Catatan:** MQ2 fisik butuh **pemanasan beberapa menit** dan kalibrasi di udara bersih. Nilai ADC bukan konsentrasi ppm. Jangan jadikan proyek ini satu-satunya pengaman gas di dunia nyata.

**Tantangan:** tambahkan dua level (waspada dan bahaya) dengan bunyi berbeda.

---

## Latihan 6: Proyek Terpadu (Smart Room Monitor)

**Tujuan:** menggabungkan semua komponen dalam satu sistem dan mengirim data lewat MQTT.

**Spesifikasi:**

| Fungsi | Komponen | Perilaku |
| --- | --- | --- |
| Suhu dan kelembapan | DHT22 | Kirim tiap 5 detik |
| Kualitas udara | MQ2 | Alarm bila melewati ambang |
| Deteksi orang mendekat | HC-SR04 | Buka pintu (servo 90 derajat) bila jarak kurang dari 30 cm |
| Kipas otomatis | Relay | Hidup bila suhu tinggi atau gas bahaya |
| Mode manual | Pushbutton | Ganti mode otomatis dan manual |
| Set batas suhu | Potensiometer | Atur ambang suhu 25 sampai 40 C |
| Indikator | LED, buzzer | Hijau untuk aman, merah untuk peringatan |

**Topik MQTT yang disarankan** (ganti awalan dengan yang unik):

```
smk-demo/ruang1/suhu
smk-demo/ruang1/kelembapan
smk-demo/ruang1/gas
smk-demo/ruang1/status
```

**Petunjuk pengerjaan:**

1. Gabungkan kode Latihan 1 sampai 5 dalam fungsi terpisah (`bacaSensor()`, `kendaliKipas()`, `kendaliPintu()`).
2. Gunakan `millis()` sebagai pengganti `delay()` agar semua tugas berjalan bersamaan.
3. Tambahkan koneksi WiFi dan MQTT dari [Cheatsheet utama](index.html#mqtt-kirim-data-sensor).
4. Simpan SSID dan kredensial di `secrets.h`, bukan di kode utama.

---

## Rubrik Penilaian Singkat

| Aspek | Skor |
| --- | --- |
| Rangkaian sesuai peta pin dan berfungsi | 30 |
| Kode rapi, terkomentari, tanpa `delay` panjang | 25 |
| Logika kontrol benar (histeresis, ambang) | 25 |
| Keamanan dasar (kredensial tidak tertanam, topik unik) | 10 |
| Dokumentasi (tangkapan layar dan tautan proyek Wokwi) | 10 |

---

[Kembali ke atas](#top) | [Cheatsheet utama](index.html)
