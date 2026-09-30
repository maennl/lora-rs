# AI-generierte Patches in diesem Fork

Dieser Fork von [lora-rs](https://github.com/lora-rs/lora-rs) trägt Patches am Crate
`lora-phy`, die **von einer KI (Claude) erzeugt** und von Maennl Elektronik geprüft wurden. Sie
sind bewusst **nicht** upstream gemeldet.

Jeder Patch ist zusätzlich an seiner Stelle im Quelltext mit `[AI-GENERATED PATCH]`
gekennzeichnet, und die zugehörigen Commits tragen `[AI]` im Betreff. Wer diesen Code liest,
soll an jeder Stelle sehen, woher er kommt, und nicht erst durch das Lesen dieser Datei.

Konsumenten: die Firmwares `com-lora` und `ebon-lora-modul` des ebon-Verbunds, eingebunden per
`[patch.crates-io]`. Beide versionieren ihre `Cargo.lock` nicht.

## Branch-Aufbau

Je Zielversion von `lora-phy` gibt es **einen eigenen Branch**, abgezweigt vom Versions-Tag:

| Branch | Basis | Enthält |
|---|---|---|
| `ai/header-error-rx-ende-3.0.1` | Tag `lora-phy-v3.0.1` (`ca04c228`) | Patch 1 |

`main` bleibt unverändert der Upstream-Stand und wird nicht angefasst.

**Beim Sprung auf eine neue `lora-phy`-Version** wird ein neuer Branch vom neuen Tag angelegt
und der Patch dorthin übertragen: **kein Rebase und kein Force-Push auf einen bestehenden
Branch.** Die Konsumenten versionieren ihre `Cargo.lock` nicht, also zieht jeder frische Bau den
**Kopf** des Branches. Ein umgeschriebener Branch ändert dann still, was gebaut wird, ohne dass
im Konsumenten irgendeine Datei anders aussieht.

Die Version im Branch bleibt die des Tags (3.0.1). Mit einem anderen Wert wäre der Patch nicht
mehr semver-verträglich, und Cargo lehnte ihn ab. `lora-modulation` kommt über die
Pfad-Abhängigkeit im Workspace aus demselben Tag mit.

## Patch 1: `HeaderError` beendet ein Empfangsfenster

**Stelle:** `lora-phy/src/sx126x/mod.rs`, `process_irq_event`, Zweig
`RadioMode::Receive(rx_mode)`, Prüfung auf `IrqMask::HeaderError`.

**Änderung:** Außerhalb von `RxMode::Continuous` gibt die Funktion bei gesetztem `HeaderError`
`Err(RadioError::ReceiveTimeout)` zurück, statt nur eine `debug!`-Zeile auszugeben und mit
`Ok(None)` weiterzumachen. Die Prüfung steht vor denen auf `RxDone`, `PreambleDetected` und
`HeaderValid`. Sind diese Flags im selben Lesevorgang mit gesetzt, endet das Fenster trotzdem.

**Ursache:** `do_rx` setzt `SetStopRxTimerOnPreamble = 1` und startet `SetRx` ohne RTC-Timer,
nur mit Symbol-Timeout. Nach einer erkannten Präambel läuft damit kein Timeout mehr. Folgt ein
Header-Fehler, löscht `process_irq_event` alle Flags und liefert `Ok(None)`, und `LoRa::rx`
wartet erneut auf DIO1. Der Chip löst danach kein Ereignis mehr aus, und das Warten endet nie,
es sei denn, zufällig trifft ein gültiges Paket ein. Semtechs Referenztreiber (LoRaMac-node)
wertet `IRQ_HEADER_ERROR` im Single-Modus ebenfalls als Ende des Empfangs.

**Wirkung:** `rx()` kehrt mit `ReceiveTimeout` zurück und schaltet dabei über seinen
vorhandenen Fehlerzweig in den Standby. Für den Aufrufer ist das ein leeres Fenster wie jedes
andere. `RadioError` bleibt unverändert.

**Beleg:** Todo #517 im ebon-Verbund. Eine com-lora-Firmware (SX1262/HPD16A, SF7/125 kHz,
Fenster `RxMode::Single(255)`) blieb im 8-h-Dauerlauf zweimal im Abstand von rund 3,5 h in
`rx()` stehen, ohne dass je ein Paket empfangen wurde, und wurde vom Task-Wächter neu
gestartet. Der Konsument begrenzt `rx()` zusätzlich mit einer eigenen Frist und zählt die
Fälle, in denen sie greift. Das Ergebnis des nachfolgenden Dauerlaufs zeigt, ob der Patch
allein trägt.

**Checkliste für den Sprung auf eine neue Version:**

1. Neuen Branch ab dem neuen `lora-phy`-Tag anlegen.
2. Prüfen, ob Upstream `HeaderError` inzwischen selbst als Fensterende behandelt. Wenn ja, kann
   der Fork für diese Version entfallen.
3. Andernfalls den Patch übertragen und in beiden Konsumenten den `[patch.crates-io]`-Eintrag
   auf den neuen Branch umstellen.
