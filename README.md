# RS41_ReverseEngineering
Für allgemeine Informationen über Radiosonden [Wikipedia](https://de.wikipedia.org/wiki/Radiosonde).

Für weitere Informationen über die RS41 [meine Website](https://example.com).

Der blockweise Aufbau der RS41 soll im nachfolgenden beschrieben werden. Bereitgestellt wird weiterhin der Schaltplan im Eagle-Format, Logic-Analyzer-Aufzeichnungen der funktionalen Blöcke und hochauflösende Scans der Leiterplatten.

Die Untersuchung der RPM411 Tochterplatine mit barometrischem Sensor ist im separaten Unterordner zu finden.

Pull Requests mit Verbesserungen, Übersetzungen und Fehlerkorrekturen sind gerne gesehen!

# ToDo
* Unidentifizierte Komponenten identifizieren
* Unbekannte Verbindungen finden
* Werte der passiven Bauteile identifizieren
* Herausfinden, welche der im Schaltpaln eingetragen Widerstände in Wahrheit ESD-Surpressor etc. sind.
* Vermessung des Frontends im Klimaschrank
* Detaillierte Beschreibung des SPI-Busses
* Detaillierte Beschreibung des UART zwischen GPS und MCU
* funktionale Untersuchung NFC-Interface
* Sniffing der Kommunikation zwischen RI41 Groundcheck Device und RS41
* Flashdump des Controllers erhalten und reversen

# Einleitung
Die Sonde ist in sechs funktionale Baugruppen zu unterteilen, die auf dem nachfolgengen Bild hervorgehoben sind.

* Stromversorgung
* Mikrocontroller
* Mess-Frontend
* GPS
* Radio
* Interface

Das Reverse-Engineering wird dadurch erschwert, dass die Platine vierlagig ist.

# Stromversorgung
![Power Supply](__used_asset__/supply_sch.png?raw=true "Power Supply")

Die Stromversorgung lässt sich in drei Teile einteilen

* ein Boost-Converter erzeugt 3,8 V Spannung aus der variablen Batteriespannung
* drei Low Dropout Regulators (LDOs) erzeugen aus diesen jeweils eine 3-V-Schiene für unterschiedliche Schaltungsteile
* eine hartverdrahtete Logik ermittelt den Betriebszustand des Boostconverters und damit der Sonde

## Boost-Converter
Als Boost-Converter kommt ein [TPS61200](http://www.ti.com/lit/ds/symlink/tps61200.pdf) `U502` von TI zum Einsatz, desen Beschaltung der Typical Application entspricht. Im Eingangskreis befindet sich zunächst eine SMD-Sicherung `R502` und eine Clamping-Diode `D501`. Zwischen Batterie und Boost-Converter befindet sich ein P-Kanal MOSFET `Q501`, der während der Lagerung durch den Pullup-Widerstand `R501` geschlossen ist. `Q501` kann durch eine hartverdahtete Logik (s.u.) geöffnet oder wieder geschlossen werden, um die Sonde ein- oder auszuschalten.

## LDOs
Es werden [TVS70030](http://www.ti.com/lit/ds/symlink/tlv700-q1.pdf) von TI eingesetzt. `U501` erzeugt die Spannung für den Mikrocontroller (MCU), `U503` für das Mess-Frontend und `U504` für das GPS-Modul. Pin 4 der LDOs, der Laut Datenblatt NC ist, ist gegen Masse entkoppelt, vermutlich damit auch ansonsten pinkompatible Versionen wie der [MAX8887](https://datasheets.maximintegrated.com/en/ds/MAX8887-MAX8888.pdf) eingesetzt werden können.

## Hartverdrahtete Logik
Der oben besprochene P-Kanal MOSFET Q501 wird durch einen N-Kanal MOSFET `Q502` gesteuert. 
* Die Sonde ist eingeschaltet, wenn dieser Transistor geschlossen ist, sein Gate also HIGH.
* Die Sonde ist ausgeschaltet, wenn dieser Transistor offen ist, sein Gate also LOW.

Über `R506` und `D502` gelangt ein einweg-gleichgerichtetes Signal der NFC-Spule auf das Gate. Hierdurch kann die Sonde über NFC eingeschaltet werden. Weiterhin wird diese Signal auch zur Kommunikation mit dem RI41 Groundcheck Device über den Spannungsteiler aus `R510` und `R514`/`C525` zum MCU geführt.

Ist die Sonde einmal eingeschaltet, wird sie über die geschaltete Batteriespannung, die über `R505` an das Gate geführt wird, in diesem Zustand gehalten.

Um sie wieder auszuschalten, kann der MCU den N-Kanal MOSFET `Q503` schließen, was das Gate von `Q502` auf LOW bringt.

Der Taster an der Unterseite der Sonde `S501` schaltet das Gate von `Q502` ebenfalls über `R507` auf HIGH. Damit der Mikrocontroller auch den Zustand des Tasters abfragen kann, ist die Gatespannung von `Q502` über den Spannungsteiler `R506` und `R512`/`C524` an einen ADC-Eingang des MCU geführt werden. Ausgewerter wird hier der geringere Spannungsabfall über `R507`, der zu einer höheren Gatepannung führt, wenn dieser gedrückt ist.

Letztlich kann auch die Batteriespannung selbst über den Spannungsteiler `R508` und `R512`/`C524` ausgewertet werden.

# Mikrocontroller
![Microcontroller](__used_asset__/mcu_sch.png?raw=true "Microcontroller")

Der Mikrocontroller ist ein [STM32F100C8](https://www.st.com/resource/en/datasheet/stm32f100c8.pdf) `U101` von ST im LQFP48-Gehäuse, der seinen Takt vom  24 MHz-Quarz `X101` erhält. Außer der Tatsache, dass alle IO-Pins belegt sind, ist nur erwähnenswert, dass an vielen Ausgängen RC-Tiefpassfilter vorhanden sind.

# Mess-Frontend
![Mess-Frontend](__used_asset__/frontend_sch.png?raw=true "Mess-Frontend")

## Schaltungsanordnung
Mechanisch besteht das Frontend aus dem Messfühler, der aus Flex-PCB-Material besteht, welches mit silberner Farbe verdeckt ist und mit einem 20-Pin-FPC-Steckverbinder mit dem Board verbunden ist. Der Messfühler beinhaltet zum einen einen PT1000-Thermofühler, der diskret als Draht ausgeführt ist (der charakteristische 'Haken') und zum anderen aus einem Keramik-Hybridmodul, welches der Messung der Luftfeuchtigkeit dient. Es vereinigt drei Funktionen
* Messung der Luftfeuchtigkeit über ein kapazitives Hygrometer (Dielektrizitätskonstante des hydrophilen Dielektrikums verändert sich und damit die Kapazität des Kondensators über den Messbereich.
* Messung der Temperatur des Moduls mittels PT1000 als Rückführgröße für die Regelung der Modulbeheizung.
* Modulbehizung über einen Dickschichtwidestand. Während des Fluges wird das Modul 5 K über Umgebungstemeperatur gehalten, um Kondesation zu verhindern. Beim Preflight-Check wird es aufgeheizt, um Verunreinigungen zu entfernen und eine Zero-Humidity-Plausibiltätsprüfung durchzuführen. Es findet keine On-Site Kalibrierung im Ground Check Device für Temperatur und Feuchtigkeit statt.

Weiterhin sind auf einem durch gefräste Slots vom Rest der Leiterplatte abgegrenzten Bereich zwei Referenzwiderstände `R208` und `R209`, ein Refernzkondensator `C209`, fünf Heizwiderstände `R201-205` und die zur Ansteuerung dieser benötigten MOSFETs `Q203-204,` sowie ein unidentifiziertes Bauteil `R209` vorhanden, bei dem es sich um einen Thermistor handeln könnte. Es konnte bei Tests mit Zimmertemperatur kein Heizen der Referenzen beobachtet werden.

## Schaltungstopologie
ELektrisch gesehen besteht das Frontend aus zwei Ringoszillatoren für Temperatur bzw. Luftfeuchtigkeit, deren Frequenz durch mithilfe von Analogschaltern in den Rückkopplungspfad eingefügten Impedanzen variiert wird.

Jeweils ein Ringoszillator wird durch 3/6 eines [74HCU04](https://assets.nexperia.com/documents/data-sheet/74HCU04.pdf) Hex Inverters `U205` gebildet. Es sind jeweils drei Inverter hintereinandergeschaltet. Der jeweils erste und dritte Inverter sind direkt über ein RC-Reihenglied `C207`/`R210`, `R214`/`C215`, `C208`/`R211`, `R215`/`C216` rückgekoppelt, der gesamte Ringoszillator über einen Kondensator `C212`, `C213`. Beide Ringoszillatoren können durch einen P-Kanal-MOSFET `Q201`, `Q202` am ersten Inverter mit jeweils separater Anteuerung auf +3 V gezogen werden, was Masse am Ausgang entspricht, um sie zu deaktivieren. Weiterhin am Eingang der Inverter verortet sind die Heizwiderstände für die Referenz, die durch schließen von `Q203`, `Q204` bei geschlossenen P-Kanal-MOSFETs `Q201`, `Q202` aktiviert werden können.

Der Rückkopplungspfad für die Temperaturmessung besteht aus einer Hintereinanderschaltung von 2/3 Buffern eines [74LVC3G34](https://assets.nexperia.com/documents/data-sheet/74LVC3G34.pdf) `U207` und zwei Widerständen `R219` und `R223`, die vermutlich der Feinabstimmung der Resonanzfrequenz dienen. Daran schließen sich vier Single-Pole-Single-Throw(SPST)-Schalter [TS3A4751](http://www.ti.com/lit/ds/symlink/ts3a4751.pdf) `U201` an, deren NO mit dem Ausgang des Buffers verbunden ist, und die jeweils einen der vier möglichen Messwiderstände in die Rückkoppelschleife schalten. Der Messabgriff besfindet sich zwischen Buffer und Schalter mit einem Reihenwiderstand `R224`.

Im Rückkopplungspfad der Feuchtigkeitsmessung befindet sich eine Schaltung aus dem verbleibenden Buffer und zwei Widerständen `R226` und R220, Der Messabgriff erfolgt genau wie bei der Temperaturmessung durch einen Widerstand `R225`. Weitrhin im Rückkopplungspfad befinden sich drei SPDT-Schalter [TS5A9411](http://www.ti.com/lit/ds/symlink/ts5a9411.pdf) `U202-204`, die den Mess- und Refernzkondensator entweder in das Rückkopplungsnetzwerk, oder gegen Masse schalten. U205 weist zwischen COM und dem Eingang des ersten Buffers, wo sich normalerweise ein Referenzkondensator befinden sollte, keine Verbindung auf, sodass dieser Schalter keinen offensichtlichen Zweck erfüllt. Da er aber wie die anderen beiden auch in Software angesteuert wird, kann vermutet werden, dass die Schalter eine exemplarunabhängige nichtlineare Wirkung auf die Rückkopplungsfrequenz aufweisen, die an mit diesem Schalter gemessen wird, um sie zu kompensieren. Parallel zu den Schaltern wird fix mit dem Widerstand `R212` rückgekoppelt.

Die beiden Messausgänge werden in einem NOR-Gatter `U208` umgesetzt und das Ergebnis an den MCU geführt, sodass immer nur eine Messung durchgeführt werden kann. Messungen mit dem Logic Analyzer zeigen, dass die Temperatur zwei Mal pro Sekunde und die Feuchtigkeit ein Mal pro Sekunde gemessen wird.

![Logic Analyzer](__used_asset__/meas_out.png?raw=true "Logic Analyzer")

Die Beheizung des Feuchtigkeitssensors wird über ein unidentifiziertes Bauteil `U206` gesteuert.

# GPS
![GPS](__used_asset__/gps_sch.png?raw=true "GPS")

Das GPS-Modul [UBX-6010](__used_asset__/gps_datasheet.pdf?raw=true) `U302` ist über einen UART an den MCU angebunden. Die Beschaltung entspricht bis auf den diskreten SAW-Filter und LNA im wesentlichen der Typical Application

# Radio
![Radio](__used_asset__/radio_sch.png?raw=true "Radio")

Das Radio-Interface ist eine One-Chip-Solution mti dem [Si4032](https://www.silabs.com/documents/public/data-sheets/Si4030-31-32.pdf) `U401`, der über SPI mit dem MCU verbunden ist. Erwähnenswert sind neben dem sekundären, unbenutzten Antennen-Pad zwei nicht zuortbare Leiterbahnen an TX und XOUT.

Von den drei GPIO-Pins werden zwei verwendet. GPIO1 schaltet die N-Kanal-MOSFETS, um die Referenz zu beheizen. GPIO2 ist bisher nicht identifiziert.

# Interface
![Interface](__used_asset__/interface_sch.png?raw=true "Interface")

