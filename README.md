
#define BLYNK_TEMPLATE_ID "TMPL66tN2n3-y"
#define BLYNK_TEMPLATE_NAME "Monitoring CH4"
#define BLYNK_AUTH_TOKEN "rBLfwZ9JH12qGT-Mn9imwo6D8hdtN1Qi"

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>

// ================= WIFI =================
char ssid[] = "Lokambinggggggggg";
char pass[] = "aqiruuuuu";

// ================= SENSOR MQ2 =================
const int pinMQ2 = 35;

const float RL = 5.0;
float Ro = 7.2;

// Batas peringatan PPM
const float batasWaspada = 300.0;
const float batasBahaya = 800.0;

// ADC rata-rata
int adcRataRata = 0;

// Timer Blynk
BlynkTimer timer;

// ================= PEMBACAAN SENSOR =================
float bacaRsRataRata(int pin, int jumlahSampel = 30) {

  float totalRs = 0;
  float totalADC = 0;
  int validSampel = 0;

  for (int i = 0; i < jumlahSampel; i++) {

    int adc = analogRead(pin);

    float vrl = (adc / 4095.0) * 3.3;

    if (vrl > 0.01 && vrl < 3.29) {

      float rs = ((3.3 - vrl) / vrl) * RL;

      totalRs += rs;
      totalADC += adc;

      validSampel++;
    }

    delay(10);
  }

  if (validSampel == 0) {
    adcRataRata = 0;
    return Ro * 9.8;
  }

  adcRataRata = totalADC / validSampel;

  return totalRs / validSampel;
}

// ================= PENGIRIMAN DATA =================
void kirimData() {

  float rs = bacaRsRataRata(pinMQ2, 30);

  float ratio = rs / Ro;

  if (ratio <= 0.01) {
    ratio = 0.01;
  }

  // Pendekatan kurva LPG MQ-2
  float ppm = pow(10, ((log10(ratio) - 1.30) / -0.45));

  // Pengaman nilai PPM
  if (isnan(ppm) || isinf(ppm) || ppm < 0) {
    ppm = 0.0;
  }
  else if (ppm > 10000.0) {
    ppm = 10000.0;
  }

  // Menentukan status gas
  String statusGas;

  if (ppm >= batasBahaya) {
    statusGas = "BAHAYA";
  }
  else if (ppm >= batasWaspada) {
    statusGas = "WASPADA";
  }
  else {
    statusGas = "AMAN";
  }

  // ================= SERIAL MONITOR =================
  Serial.print("ADC: ");
  Serial.print(adcRataRata);

  Serial.print(" | Rs/Ro: ");
  Serial.print(ratio, 2);

  Serial.print(" | LPG: ");
  Serial.print(ppm, 2);

  Serial.print(" PPM | Status: ");
  Serial.println(statusGas);

  // ================= KIRIM KE BLYNK =================

  Blynk.virtualWrite(V0, ppm);          // LPG PPM
  Blynk.virtualWrite(V1, ratio);        // Rs/Ro
  Blynk.virtualWrite(V2, statusGas);    // Status gas
  Blynk.virtualWrite(V3, adcRataRata);  // ADC sensor
}

// ================= SETUP =================
void setup() {

  Serial.begin(115200);

  analogReadResolution(12);

  delay(1000);

  Serial.println("================================");
  Serial.println(" MONITORING GAS MQ-2 ESP32");
  Serial.println("================================");

  // Menghubungkan ESP32 ke Blynk
  Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

  Serial.println("Blynk berhasil terhubung!");

  // Mengirim data setiap 1 detik
  timer.setInterval(1000L, kirimData);
}

// ================= LOOP =================
void loop() {

  Blynk.run();

  timer.run();
}
