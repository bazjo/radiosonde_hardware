# RS41_ReverseEngineering
Für allgemeine Informationen über Radiosonden [Wikipedia](https://de.wikipedia.org/wiki/Radiosonde).

Für weitere Informationen über die RS41 [meine Website](https://example.com).

Der blockweise Aufbau der RS41 soll im nachfolgenden beschrieben werden. Bereitgestellt wird weiterhin der Schaltplan im Eagle-Format, Logic-Analyzer-Aufzeichnungen der funktionalen Blöcke und hochauflösende Scans der Leiterplatten.

Die Untersuchung der RPM411 Tochterplatine mit barometrischem Sensor ist im separaten Unterordner zu finden

#ToDo
* Werte der passiven Bauteile identifizieren
* Herausfinden, welche der im Schaltpaln eingetragen Widerstände in Wahrheit ESD-Surpressor etc. sind.
* Detaillierte Beschreibung des SPI-Busses
* Detaillierte Beschreibung des UART zwischen GPS und MCU
* funktionale Untersuchung NFC-Interface
* Sniffing der Kommunikation zwischen RI41 Groundcheck Device und RS41
* Flashdump des Controllers erhalten und reversen

#Einleitung
Die Sonde ist in sechs funktionale Baugruppen zu unterteilen, die auf dem nachfolgengen Bild hervorgehoben sind.

* Stromversorgung
* Mikrocontroller
* Mess-Frontend
* GPS
* Radio
* Interface

#Stromversorgung
Die Stromversorgung lässt sich in drei Teile einteilen

* ein Boost-Converter erzeugt 3,8 V Spannung aus der variablen Batteriespannung
* drei Low Dropout Regulators (LDOs) erzeugen aus diesen jeweils eine 3-V-Schiene für unterschiedliche Schaltungsteile
* eine hartverdrahtete Logik ermittelt den Betriebszustand des Boostconverters und damit der Sonde

##Boost-Converter
Als Boost-Converter kommt ein [TPS61200](http://www.ti.com/lit/ds/symlink/tps61200.pdf) U502 von TI zum Einsatz, desen Beschaltung der Typical Application entspricht. Im Eingangskreis befindet sich zunächst eine SMD-Sicherung R502 und eine Clamping-Diode D501. Zwischen Batterie und Boost-Converter befindet sich ein P-Kanal MOSFET Q501, der während der Lagerung durch den Pullup-Widerstand R501 geschlossen ist. Q501 kann durch eine hartverdahtete Logik (s.u.) geöffnet oder wieder geschlossen werden, um die Sonde ein- oder auszuschalten.

##LDOs
Es werden [TVS70030](http://www.ti.com/lit/ds/symlink/tlv700-q1.pdf) von TI eingesetzt. U501 erzeugt die Spannung für den Mikrocontroller (MCU), U503 für das Mess-Frontend und U504 für das GPS-Modul. Pin 4 der LDOs, der Laut Datenblatt NC ist, ist gegen Masse entkoppelt, vermutlich damit auch ansonsten pinkompatible Versionen wie der [MAX997](https://datasheets.maximintegrated.com/en/ds/MAX8887-MAX8888.pdf) eingesetzt werden können.

##Hartverdrahtete Logik
Der oben besprochene P-Kanal MOSFET Q501 wird durch einen N-Kanal MOSFET Q502 gesteuert. 
* Die Sonde ist eingeschaltet, wenn dieser Transistor geschlossen ist, sein Gate also HIGH.
* Die Sonde ist ausgeschaltet, wenn dieser Transistor offen ist, sein Gate also LOW.

Über R506 und D502 gelangt ein einweg-gleichgerichtetes Signal der NFC-Spule auf das Gate. Hierdurch kann die Sonde über NFC eingeschaltet werden. Weiterhin wird diese Signal auch zur Kommunikation mit dem RI41 Groundcheck Device über den Spannungsteiler aus R510 und R514/C525 zum MCU geführt.

Ist die Sonde einmal eingeschaltet, wird sie über die geschaltete Batteriespannung, die über R505 an das Gate geführt wird, in diesem Zustand gehalten.

Um sie wieder auszuschalten, kann der MCU den N-Kanal MOSFET Q503 schließen, was das Gate von Q502 auf LOW bringt.

Der Taster an der Unterseite der Sonde S501 schaltet das Gate von Q502 ebenfalls über R507 auf HIGH. Damit der Mikrocontroller auch den Zustand des Tasters abfragen kann, ist die Gatespannung von Q502 über den Spannungsteiler R506 und R512/C524 an einen ADC-Eingang des MCU geführt werden. Ausgewerter wird hier der geringere Spannungsabfall über R507, der zu einer höheren Gatepannung führt, wenn dieser gedrückt ist.

Letztlich kann auch die Batteriespannung selbst über den Spannungsteiler R508 und R512/C524 ausgewertet werden.
