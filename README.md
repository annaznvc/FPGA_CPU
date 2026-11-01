# FPGA-CPU: Entwicklungsworkflow

Ziel: Eine eigene 8-Bit-CPU in **SystemVerilog** entwickeln, automatisiert verifizieren und anschließend auf einem FPGA-Board betreiben. Danach optional auf **RV32I (RISC-V)** erweitern.

## 1. Toolchain

| Werkzeug | Aufgabe |
|---|---|
| VS Code | RTL, Tests und Skripte bearbeiten |
| SystemVerilog | Synthesefähige Hardwarebeschreibung (RTL) |
| Verilator | Simulation und Linting |
| cocotb + pytest | Automatisierte Tests in Python |
| GTKWave | Wellenformen bei Fehlern untersuchen |
| Python | Referenzmodell der CPU und Testdaten |
| Git + GitHub Actions | Versionierung, Code-Review und CI |
| Gowin EDA (Tang Nano 20K) **oder** AMD Vivado (Cmod A7) | Synthese, Place & Route, Timing-Analyse, Bitstream und Programmierung |

**Prinzip:** Die Simulation und CI laufen zunächst ohne FPGA. Board-spezifische Schritte kommen erst nach bestandenen Tests.

## 2. Repository-Struktur

```text
fpga-cpu/
├── rtl/                     # Synthesefähige SystemVerilog-Module
│   ├── cpu.sv
│   ├── alu.sv
│   ├── register_file.sv
│   ├── program_counter.sv
│   └── control_unit.sv
├── tb/                      # cocotb-Tests
│   ├── test_alu.py
│   ├── test_register_file.py
│   └── test_cpu.py
├── sim/
│   └── reference_cpu.py    # Software-Referenzmodell
├── firmware/programs/       # Testprogramme / Maschinenbefehle
├── constraints/             # Pins, Clocks, Timing (boardspezifisch)
├── scripts/                 # Reproduzierbare Synthese-/Build-Skripte
├── docs/
│   ├── architecture.md
│   └── isa.md
├── .github/workflows/ci.yml
├── Makefile
├── .gitignore
└── README.md
```

Generierte Dateien (`build/`, `sim_build/`, Wellenformen, Tool-Caches, Bitstreams) grundsätzlich nicht committen. Bei Releases kann ein geprüfter Bitstream als CI-Artefakt archiviert werden.

## 3. Der tägliche Entwicklungszyklus

1. **Spezifikation:** Verhalten eines Moduls festlegen (Eingänge, Ausgänge, Takt, Reset, Grenzfälle).
2. **Test zuerst:** cocotb-Test schreiben, der zunächst fehlschlägt.
3. **RTL implementieren:** Ein kleines SystemVerilog-Modul entwickeln.
4. **Lokal prüfen:** Lint, Simulation und Tests ausführen; bei Fehlern Wellenformen ansehen.
5. **Integration:** Modul in CPU einbinden und Integrations-/Programmtests laufen lassen.
6. **Pull Request:** CI muss grün sein; RTL, Tests und Architekturänderungen gemeinsam reviewen.
7. **FPGA-Build:** Synthese, Place & Route und Timing-Analyse ausführen.
8. **Hardwaretest:** Bitstream programmieren und Ergebnisse über LEDs/UART kontrollieren.

**Merke:** Eine bestandene Simulation beweist noch nicht, dass die Schaltung auf echter Hardware die Timing-Anforderungen erfüllt.

## 4. Teststrategie

### Unit-Tests

- ALU: ADD, SUB, AND, OR, Null, Überlauf und Randwerte.
- Register File: Lesen, Schreiben, Reset und gleichzeitige Zugriffe.
- Program Counter: Inkrement, Sprung und Reset.
- Steuerwerk: Opcode-Decodierung und Zustandsübergänge.

Beispiel für einen cocotb-Test (Signalnamen müssen zur ALU passen):

```python
import cocotb
from cocotb.triggers import Timer

@cocotb.test()
async def test_alu_addition(dut):
    dut.a.value = 12
    dut.b.value = 7
    dut.op.value = 0  # ADD
    await Timer(1, unit="ns")
    assert int(dut.result.value) == 19
```

### Integrationstests

- Kleine Maschinenprogramme in den Instruktionsspeicher laden.
- Register- und Speicherzustände nach jeder Instruktion überprüfen.
- Ergebnisse gegen ein **Python-Referenzmodell** vergleichen (Differential Testing).
- Später zufällige Testprogramme und Assertions ergänzen.

### Hardwaretests

- Definierten Reset prüfen.
- Ein bekanntes Testprogramm ausführen.
- Ergebnis per UART oder LED sichtbar machen.
- Timing-Reports prüfen: insbesondere Setup-/Hold-Verletzungen und Clock-Constraints.

## 5. Einheitliche Befehle

Ein `Makefile` dient als stabile Schnittstelle. Die konkreten Targets hängen von der gewählten Simulator- und Boardkonfiguration ab.

```text
make lint      # RTL statisch prüfen
make test      # cocotb-Simulationen starten
make synth     # RTL synthetisieren
make build     # FPGA-Bitstream erstellen
make program   # FPGA programmieren
make clean     # Generierte Dateien löschen
```

Die Befehle werden **erst nach Einrichtung** der cocotb-Runner-Konfiguration und der passenden Gowin-/Vivado-Skripte ausführbar. Tool-Versionen und Python-Abhängigkeiten fixieren.

## 6. CI/CD mit GitHub Actions

Bei jedem Push und Pull Request:

1. Repository auschecken.
2. Gepinnte Python-/Verilator-Abhängigkeiten installieren.
3. RTL linten.
4. Unit- und Integrationstests ausführen.
5. Testberichte und bei Fehlern Wellenformen als Artefakte sichern.

Optional für `main`, Tags oder Nightly Builds:

6. Board-spezifische Synthese und Place & Route starten (ggf. auf Self-hosted Runner wegen Toolgröße/Lizenz).
7. Ressourcenverbrauch und Timing-Slack aus Reports prüfen.
8. Bitstream mit Commit-Hash und Tool-Version archivieren.

**Hardware-in-the-Loop** erst später: Ein Runner mit angeschlossenem FPGA programmiert das Board und führt einen UART-basierten Smoke-Test aus.

## 7. FPGA-spezifische Best Practices

- **Synchrones Design:** Möglichst eine klar definierte Clock-Domain verwenden; zusätzliche Clock-Domains bewusst behandeln.
- **Reset:** Reset-Polarität, synchron/asynchron und Startzustände dokumentieren.
- **Keine unbeabsichtigten Latches:** In kombinatorischen Prozessen alle Ausgänge vollständig zuweisen.
- **Nonblocking Assignments (`<=`)** in getakteten `always_ff`-Prozessen nutzen.
- **Timing-Constraints versionieren:** Taktfrequenzen, Pins und I/O-Anforderungen festlegen.
- **CDC prüfen:** Signale zwischen Clock-Domains korrekt synchronisieren.
- **Keine generierten Dateien in Git:** Stattdessen Build-Skripte, Constraints und Tool-Versionen committen.
- **Ressourcen und Timing beobachten:** LUTs, Flipflops, Block-RAM, DSPs und Slack nach jedem größeren Ausbau vergleichen.
- **Herstellerunabhängigkeit:** CPU-RTL und Tests von Board-spezifischen Top-Modulen und IPs trennen.

## 8. Meilensteine

- [ ] **M1 – Setup:** Git-Repository, Verilator, cocotb, Lint, erster ALU-Test, GitHub Actions.
- [ ] **M2 – Architektur:** Eigene 8-Bit-ISA, Register, Instruktionsformat, Reset und Speicherlayout spezifizieren.
- [ ] **M3 – Komponenten:** ALU, Register File, Program Counter, Steuerwerk implementieren und einzeln testen.
- [ ] **M4 – Integration:** CPU führt Testprogramme aus; Python-Referenzmodell bestätigt Ergebnisse.
- [ ] **M5 – FPGA:** Board-Constraints, Synthese, Place & Route, Timing-Analyse, Bitstream und UART-/LED-Smoke-Test.
- [ ] **M6 – Ausbau:** Eigenständigen RV32I-Kern entwickeln oder Architektur entsprechend neu aufbauen; später KI-Beschleuniger und Ethernet ergänzen.

## 9. Lernressourcen

- **Harris & Harris – Digital Design and Computer Architecture: RISC-V Edition:** Digitaltechnik, Datenpfad, ALU und CPU-Architektur. https://pages.hmc.edu/harris/ddca/ddcarv.html
- **HDLBits:** Kleine Verilog-Übungen zu kombinatorischer und sequentieller Logik. https://hdlbits.01xz.net/wiki/Main_Page
- **Nand2Tetris:** Vom Logikgatter zur CPU. https://www.nand2tetris.org/
- **RISC-V-Spezifikationen:** Später für RV32I. https://docs.riscv.org/
- **Verilator:** https://verilator.org/
- **cocotb:** https://www.cocotb.org/

## 10. Erster konkreter Schritt

**Ohne Hardware beginnen:** Repository anlegen, Verilator und cocotb installieren, eine 8-Bit-ALU in `rtl/alu.sv` implementieren und einen automatisierten Additionstest in `tb/test_alu.py` ausführen. Danach den Test in GitHub Actions übernehmen.
