# GenASM: Approximate String Matching (ASM) Acceleration Framework für die Genomsequenzanalyse
GenASM ist ein ASM-Beschleunigungsframework (Approximate String Matching) für die Genomsequenzanalyse. GenASM ist ein schnelles, effizientes und flexibles Framework für kurze und lange Reads, das zur Beschleunigung mehrerer Schritte der Genomsequenzanalyse-Pipeline verwendet werden kann. GenASM basiert auf dem Bitap-Algorithmus. Bitap verwendet nur schnelle und einfache bitweise Operationen, um ein annäherndes String-Matching durchzuführen. Unseres Wissens nach ist GenASM die erste Arbeit, die Bitap verbessert und beschleunigt.

Um Bitap für GenASM zu verwenden, nehmen wir zwei wichtige algorithmische Änderungen vor, die es uns ermöglichen, die wichtigsten Einschränkungen zu überwinden, die den ursprünglichen Bitap-Algorithmus daran hindern, für die Genomsequenzanalyse effizient zu sein. Erstens: Um die Anwendbarkeit von Bitap auf verschiedene Sequenzierungstechnologien und seine Leistung zu verbessern, modifizieren wir (1) den Algorithmus, um lange Reads zu unterstützen (zusätzlich zu den bereits unterstützten kurzen Reads), und (2) eliminieren schleifengetragene Datenabhängigkeiten, so dass wir eine einzelne String-Matching-Operation parallelisieren können. Zweitens entwickeln wir einen neuartigen Bitap-kompatiblen Algorithmus für Traceback. Der ursprüngliche Bitap-Algorithmus ist nicht in der Lage, Traceback durchzuführen.

## GenASM-DC Algorithmus

Bitap ist ein ASM-Algorithmus, der das Problem der Berechnung der minimalen Edit-Distanz zwischen einem Referenztext (z. B. Referenzgenom) und einem Abfragemuster (z. B. Read) mit maximal k Fehlern löst. Wenn k gleich 0 ist, findet der Algorithmus die exakten Übereinstimmungen. 

Wir modifizieren den grundlegenden Bitap-Algorithmus, um (1) ein effizientes Alignment von langen Reads zu ermöglichen, (2) die Datenabhängigkeit zwischen den Iterationen zu beseitigen und (3) Parallelität für die große Anzahl von Iterationen zu bieten.

**Der GenASM-DC-Algorithmus überwindet die Wortlängenbegrenzung von Bitap, indem er die Bitvektoren in mehreren Wörtern speichert, wenn die Abfrage länger als die Wortgröße ist. Obwohl diese Änderung zu zusätzlichen Berechnungen bei der Durchführung von Verschiebungen führt, hilft sie GenASM, sowohl kurze als auch lange Lesevorgänge zu unterstützen. Wenn das Wort i eines Mehrwort-Bitvektors verschoben wird, muss das herausgeschobene Bit (MSB) von Wort i - 1 separat gespeichert werden, bevor die Verschiebung auf Wort i - 1 durchgeführt wird. Dann muss dieses gespeicherte Bit als das niederwertigste Bit (LSB) von Wort i geladen werden, wenn die Verschiebung erfolgt. 

**Entfernung der Schleifenabhängigkeit** Um die Beschränkung des Bitap-Basisalgorithmus auf zwei Datenebenen zu beseitigen, führt GenASM-DC eine Schleifenentrollung durch und ermöglicht die parallele Berechnung nicht benachbarter (d. h. unabhängiger) Bitvektoren. Bitte beachten Sie, dass die aktuelle CPU-Implementierung von GenASM-DC diese Änderung nicht berücksichtigt, aber wir werden unsere Version bald mit dieser Änderung aktualisieren.

**Parallelität auf Textebene** Zusätzlich zu der Parallelität, die durch die Beseitigung der Schleifenabhängigkeiten ermöglicht wird, ermöglichen wir dem GenASM-DC-Algorithmus, Parallelität auf Textebene zu nutzen. Diese Parallelität wird durch die Aufteilung des Textes in sich überschneidende Teiltexte und die parallele Suche der Abfrage in jedem dieser Teiltexte ermöglicht. Die Überlappung stellt sicher, dass wir keine mögliche Übereinstimmung übersehen, die an den Rändern eines Teiltextes liegen könnte. Um dies zu gewährleisten, muss die Überlappung die Länge m + k haben, wobei m die Abfragelänge und k der Schwellenwert für den Editierabstand ist.

## GenASM-TB Algorithmus

Nach der Ermittlung der übereinstimmenden Textstelle und der Editierdistanz mit GenASM-DC findet unser neuer Traceback-Algorithmus, GenASM-TB, die Sequenz von Übereinstimmungen, Substitutionen, Einfügungen und Löschungen zusammen mit ihren Positionen (d. h. CIGAR-String) für die übereinstimmende Region (d. h. die Textregion, die an der von GenASM-DC gemeldeten Stelle beginnt und eine Länge von m + k hat) und meldet die optimale Ausrichtung. Die Ausführung von Traceback (1) beginnt mit dem ersten Zeichen der übereinstimmenden Region zwischen dem Referenztext und dem Abfragemuster, (2) untersucht jedes Zeichen und entscheidet, welche der vier Operationen in jeder Iteration ausgewählt werden soll, und (3) endet, wenn das letzte Zeichen der übereinstimmenden Region erreicht ist. 

GenASM-TB verwendet die Zwischenbitvektoren, die in jeder Iteration des GenASM-DC-Algorithmus erzeugt und gespeichert werden (d. h. die Bitvektoren für Übereinstimmung, Ersetzung, Löschung und Einfügung). Nachdem ein Wert 0 am MSB eines der R[d]-Bitvektoren gefunden wurde (d. h. eine String-Übereinstimmung mit d Fehlern wurde gefunden), geht GenASM-TB durch die Bitvektoren zurück zum LSB, folgt einer Kette von 0s (die Übereinstimmungen an jeder Stelle anzeigen) und kehrt die bitweisen Operationen um. An jeder Position, basierend darauf, welcher der vier Bitvektoren in jeder Iteration den Wert 0 hat (beginnend mit einem MSB mit einer 0 und endend mit einem LSB mit einer 0), wird die Sequenz von Übereinstimmungen, Substitutionen, Einfügungen und Löschungen (d. h. die Traceback-Ausgabe) für jede Position des entsprechenden Alignments gefunden, das von GenASM-DC gefunden wurde. Im Gegensatz zu GenASM-DC hat GenASM-TB einen unregelmäßigen Kontrollfluss innerhalb der gespeicherten Zwischenbitvektoren, der vom Text und dem Muster abhängt.


**Da GenASM-DC alle Zwischenbitvektoren speichert, kann die Länge der Textregion, auf die das Abfragemuster abgebildet wird, im ungünstigsten Fall m + k betragen, wenn man davon ausgeht, dass es sich bei allen Fehlern um Löschungen aus dem Muster handelt. Da wir alle Bitvektoren für m + k Zeichen speichern und in jeder Textiteration 4 × k viele Bitvektoren (jeweils m Bit lang) berechnen müssen, beträgt der Speicherbedarf ((m + k) × 4 × k × m) Bit.
Um den Speicherbedarf des Algorithmus zu verringern, wenden wir einen Divide-and-Conquer-Ansatz an. Anstatt alle Bitvektoren für m + k Textzeichen zu speichern, unterteilen wir den Text und das Muster in sich überlappende Fenster (d. h. Untertext und Untermuster) und führen die Rückverfolgungsberechnung für jedes Fenster durch, wobei W die Fenstergröße und O die Überlappungsgröße ist. Nachdem alle partiellen Traceback-Ausgaben der Fenster generiert wurden, fügen wir sie zusammen, um die vollständige Traceback-Ausgabe zu finden. Dieser Ansatz hilft uns, (W × 4 × W × W) Bits zu speichern.

## GenASM ausführen

### Ausrichtung lesen (DC+TB)

Rufen Sie die folgende Funktion in der Datei genasm_aligner.c in Ihrem C-Code auf oder aktualisieren Sie die bestehende main() in der Datei genasm_aligner.c:

```bash
genasm_aligner(<Referenzsequenz>, <Abfragesequenz>, <Abstandsschwelle>, <Übereinstimmungswert>, <Substitutionsstrafe>, <Lückenöffnungsstrafe>, <Lückenerweiterungsstrafe>)
```
Beispiel:

```bash
genasm_aligner("AATGTCC", "ATCTCGC", 3, 3, 4, 5, 1);
```

### Leseausrichtung (DC+TB) mit Divide-and-Conquer (DQ)

Rufen Sie die folgende Funktion in der Datei genasm_aligner_withDQ.c in Ihrem C-Code auf oder aktualisieren Sie die bestehende main() in der Datei genasm_aligner_withDQ.c:

```bash
genasm_aligner(<Referenzsequenz>, <Abfragesequenz>, <Abstandsschwelle>, <Fenstergröße>, <Überlappungsgröße>, <Übereinstimmungsbewertung>, <Substitutionsstrafe>, <Lückenöffnungsstrafe>, <Lückendehnungsstrafe>)
```
Beispiel:

```bash
genasm_aligner("AATGTCC", "ATCTCGC", 3, 20, 10, 3, 4, 5, 1);
```

### Filterung vor dem Alignment (nur DC)

Rufen Sie die folgende Funktion in der Datei genasm_filter.c in Ihrem Code auf oder aktualisieren Sie die bestehende main() in der Datei genasm_filter.c:

```bash
genasm_filter(<Referenzsequenz>, <Abfragesequenz>, <Abstandsschwelle>)
```
Zum Beispiel:

```bash
genasm_filter("ACTACGCTTGAAATCCACAAGCATGCTGAG", "ACAAAGCTTGAATCCAGAAGCATGCTAGAG", 6);
```

## Zitat
>Damla Senol Cali, Gurpreet S. Kalsi, Zülal Bingöl, Can Firtina, Lavanya Subramanian, Jeremie S. Kim, Rachata Ausavarungnirun, Mohammed Alser, Juan Gomez-Luna, Amirali Boroumand, Anant Nori, Allison Scibisz, Sreenivas Subramoney, Can Alkan, Saugata Ghose, und Onur Mutlu.
[**"GenASM: A High-Performance, Low-Power Approximate String Matching Acceleration Framework for Genome Sequence Analysis. "**](https://people.inf.ethz.ch/omutlu/pub/GenASM-approximate-string-matching-framework-for-genome-analysis_micro20.pdf)
In _Proceedings of the 53rd International Symposium on Microarchitecture (MICRO),_ Virtual, Oktober 2020.

## Talks
Die Vorträge des Papiers sind auf YouTube verfügbar:

* [MICRO 2020 Full Talk Video (18 Minuten)](https://www.youtube.com/watch?v=srQVqPJFqjo&t=5s)
  * [Folien (PDF)](https://people.inf.ethz.ch/omutlu/pub/GenASM-approximate-string-matching-framework-for-genome-analysis_micro20-talk.pdf)

* [MICRO 2020 Short Talk Video (6 Minuten)](https://www.youtube.com/watch?v=I__0Tu6LxAY) 
  * [Folien (PDF)](https://people.inf.ethz.ch/omutlu/pub/GenASM-approximate-string-matching-framework-for-genome-analysis_micro20-short-talk.pdf)

* [MICRO 2020 Lightning Talk Video (1,5 Minuten)](https://www.youtube.com/watch?v=nJs3RRnvk_k&t=2s) 
  * [Folien (PDF)](https://people.inf.ethz.ch/omutlu/pub/GenASM-approximate-string-matching-framework-for-genome-analysis_micro20-lightning-talk.pdf)

* [ARM Research Summit Vortragsvideo (21 Minuten)](https://www.youtube.com/watch?v=oKYqVo1UTdE)

* [ARM Forschungsgipfel Kurzvortrag Video (15 Minuten)](https://www.youtube.com/watch?v=omqCTHlnMA0)

* [ARM Research Summit Short Talk Video und Q&A (31 Minuten)](https://www.youtube.com/watch?v=y1S6gtGz2bo)

## Kontakt
Damla Senol Cali (damlasenolcali@gmail.com)
