Bei **150 RPM** (Umdrehungen pro Minute) macht dein Deckenventilator exakt **2,5 Umdrehungen pro Sekunde**. Eine vollständige Drehung ($360^\circ$) dauert somit genau **400 Millisekunden ($400.000\,\mu\text{s}$)**.

Da du **5 Flügel** hast, ist der Abstand von Flügelspitze zu Flügelspitze exakt $72^\circ$, was einem Zeitfenster von **80 Millisekunden ($80.000\,\mu\text{s}$)** entspricht.

Bei dieser relativ langsamen Rotationsgeschwindigkeit (für klassische POV-Displays) funktioniert ein klassisches "Text-Schreiben" nicht optimal, da das Auge die Zeilen zu stark trennt. Was hingegen **spektakulär** aussieht, sind **geometrische, symmetrische Muster**, die die Kreisbahn optisch aufbrechen.

Hier sind drei maßgeschneiderte, hocheffiziente POV-Effekte, die direkt auf deine 5 Flügel à 30 LEDs (Spalten) und die gemessene Umlaufszeit (`gLastStrobePeriodUs`) abgestimmt sind.

---

### Vorbereitung: Wie wir die Position der Flügel bestimmen

Um zu wissen, wo sich deine Flügel gerade im Raum befinden, nutzen wir deine gemessene Umlaufzeit `gLastStrobePeriodUs` (ca. $400.000\,\mu\text{s}$).
Wir berechnen die aktuelle Mikrosekunde innerhalb der aktuellen Drehung:

```cpp
// Gibt die aktuelle Position im Kreis von 0 bis 999 zurück
uint16_t getCirclePosition() {
  if (gLastStrobePeriodUs == 0) return 0;
  uint32_t timeInCircle = micros() % gLastStrobePeriodUs;
  return (timeInCircle * 1000UL) / gLastStrobePeriodUs;
}

```

---

## 1. Der "Sternen-Tunnel" (Geometrischer 10-Zack-Stern)

Dieser Effekt erzeugt die Illusion eines statisch an der Decke stehenden, scharf gezeichneten 10-zackigen Sterns. Da du 5 Flügel hast, blitzen wir die LEDs immer dann auf, wenn die Flügel exakt auf den vollen $72^\circ$-Schritten oder genau dazwischen ($36^{\circ}$-Schritte) stehen.

Dabei wandert die Farbe von innen nach außen, was dem Stern eine plastische Tiefenwirkung (wie ein Tunnel) verleiht.

```cpp
void eff_SternenTunnel() {
  const uint16_t pos = getCirclePosition(); // 0 bis 999
  
  // Ein 10-Zack-Stern hat alle 100 Einheiten (36 Grad) eine Spitze
  const uint16_t sector = pos % 100; 

  // Wir blitzen nur in einem sehr schmalen Korridor (Spitze des Sterns)
  if (sector < 4 || sector > 96) {
    // Erzeuge einen Farbverlauf von Innen (LED 0) nach Außen (LED 29)
    static uint8_t hueShift = 0;
    
    for (uint8_t w = 0; w < 5; ++w) {
      for (uint8_t i = 0; i < 30; ++i) {
        // Berechne ein Muster, das wie ein Tunnel nach außen zieht
        uint8_t pixelHue = hueShift + (i * 8);
        setWingPixel(w, i, CHSV(pixelHue, 255, 255));
      }
    }
    FastLED.show();
    delayMicroseconds(800); // Sehr kurzer, scharfer Blitz
    FastLED.clear();
    FastLED.show();
    
    hueShift++; // Langsame Farbrotation des Sterns
  }
}

```

---

## 2. Die "Invers-Spirale" (Hypnose-Wirbel)

Statt alle LEDs gleichzeitig blitzen zu lassen, verzögern wir den Blitz von innen nach außen minimal. Dadurch krümmen sich die eigentlich geraden Flügel für das Auge des Betrachters im Raum und es entsteht eine perfekt stehende, hypnotische Spirale an der Decke.

```cpp
void eff_HypnoseSpirale() {
  const uint16_t pos = getCirclePosition(); // 0 bis 999
  
  FastLED.clear();
  bool showActive = false;

  for (uint8_t w = 0; w < 5; ++w) {
    // Jeder Flügel ist um 200 Einheiten (72 Grad) versetzt
    uint16_t wingOffset = w * 200;
    
    for (uint8_t i = 0; i < 30; ++i) {
      // Durch den Faktor (i * 2) krümmen wir die Linie im Raum zu einer Spirale
      uint16_t targetPos = (wingOffset + (i * 2)) % 1000;
      int16_t diff = pos - targetPos;
      if (diff < -500) diff += 1000;
      if (diff > 500) diff -= 1000;

      // Wenn der Flügel genau auf der gekrümmten Linie liegt -> LED an!
      if (abs(diff) < 3) { 
        // Cyan-Blau Verlauf für einen spacigen Look
        setWingPixel(w, i, CHSV(130 + (i * 3), 255, 255));
        showActive = true;
      }
    }
  }

  if (showActive) {
    FastLED.show();
  } else {
    // Da wir hier Pixel-genau arbeiten, reicht ein einfaches Löschen 
    // der nicht aktiven Pixel ohne permanentes show() im Loop.
    FastLED.clear();
    FastLED.show();
  }
}

```

---

## 3. Der "Gefrorene Propeller" (Gegenläufige Scheiben)

Dieser Effekt teilt deine 30 LEDs pro Flügel in zwei Hälften (0–14 und 15–29).

* Die **innere Hälfte** blitzt so, dass sie sich scheinbar langsam im Uhrzeigersinn dreht.
* Die **äußere Hälfte** blitzt so, dass sie sich gegen den Uhrzeigersinn dreht.

Das erzeugt an deiner Decke das physikalisch völlig paradoxe Bild von zwei ineinanderlaufenden, Zahnrad-ähnlichen Mustern.

```cpp
void eff_GefrorenerPropeller() {
  const uint16_t pos = getCirclePosition();
  static uint16_t rotInner = 0;
  static uint16_t rotOuter = 0;

  // Geschwindigkeiten der virtuellen Rotation (alle paar Frames leicht verschieben)
  EVERY_N_MILLISECONDS(50) {
    rotInner = (rotInner + 2) % 1000;
    rotOuter = (rotOuter - 2 + 1000) % 1000;
  }

  FastLED.clear();
  bool draw = false;

  for (uint8_t w = 0; w < 5; ++w) {
    uint16_t wingBase = w * 200; // 5 Sektoren à 72 Grad (200/1000)

    // 1. Innenbereich (LED 0-14): Dreht sich virtuell mit "rotInner"
    uint16_t targetInner = (wingBase + rotInner) % 1000;
    int16_t diffInner = pos - targetInner;
    if (abs(diffInner) < 5 || abs(diffInner) > 995) {
      for (uint8_t i = 0; i < 15; ++i) {
        setWingPixel(w, i, CRGB::Red); // Innerer Ring rot
      }
      draw = true;
    }

    // 2. Außenbereich (LED 15-29): Dreht sich entgegengesetzt mit "rotOuter"
    uint16_t targetOuter = (wingBase + rotOuter) % 1000;
    int16_t diffOuter = pos - targetOuter;
    if (abs(diffOuter) < 5 || abs(diffOuter) > 995) {
      for (uint8_t i = 15; i < 30; ++i) {
        setWingPixel(w, i, CRGB::Gold); // Äußerer Ring gold/gelb
      }
      draw = true;
    }
  }

  if (draw) {
    FastLED.show();
    delayMicroseconds(600); // Extrem scharfer Blitz für stehende Kanten
    FastLED.clear();
    FastLED.show();
  }
}

```

---

### 💡 Profi-Tipps für diese Effekte auf dem iFan04:

1. **Der Blitz-Vierteltakt (`delayMicroseconds`):** Ein mechanischer Flügel legt bei 150 RPM in $600\,\mu\text{s}$ nur einen minimalen Weg zurück. Je kürzer das `delayMicroseconds()` nach dem `FastLED.show()` gewählt wird, desto **schärfer und weniger verschwommen** ist das Muster an der Decke. Bei $600\,\mu\text{s}$ bis $1000\,\mu\text{s}$ erhältst du das beste Ergebnis.
2. **Stromversorgung beachten:** Wenn bei Effekt 1 schlagartig 150 LEDs weiß aufblitzen, bricht die 5V-Schiene des kleinen Sonoff iFan04 sofort zusammen (der eingebaute Wandler liefert meist nur max. 1A). Die Verwendung von satten Einzelfarben (nur Rot, nur Grün) oder CHSV-Farbverläufen (wie oben definiert) halbiert die Stromaufnahme drastisch und schützt das Gerät vor unkontrollierten Abstürzen.