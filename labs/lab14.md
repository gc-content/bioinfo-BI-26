# Wprowadzenie do analizy danych NGS

---

## Wprowadzenie

Sekwencjonowanie następnej generacji (NGS) umożliwia szybkie uzyskanie dużych ilości krótkich fragmentów sekwencji DNA (również cDNA). Obecnie standardem jest układ paired-end, czyli sekwencjonowanie obu końców każdego fragmentu DNA.

W analizie NGS stosuje się dwie główne strategie:


### Składanie *de novo*

Stosowane, gdy nie istnieje genom referencyjny – polega na łączeniu odczytów w dłuższe fragmenty, docelowo całe chromosomy.

### Mapowanie odczytów

Gdy mamy genom referencyjny, możemy dopasować do niego odczyty. Analizy oparte na mapowaniu obejmują m. in.:

- identyfikację wariantów genetycznych (SNP, indeli),
- określanie poziomu ekspresji genów (RNA-seq),
- identyfikację interakcji DNA–białko (ChIP-seq).

---

### Format SAM/BAM

Dopasowania zapisywane są w formacie **SAM**, który składa się z:

- **Nagłówka** (linie zaczynające się od `@`) – zawiera metadane, m. in.:
  - `@SQ` – nazwy i długości sekwencji referencyjnych, do których mapowane były odczyty
  - `@PG` – komendy wykorzystane do stworzenia pliku

- **Rekordy odczytów** – każda linia to jeden odczyt i jego dopasowanie (pozycja, jakość, szczegóły dopasowania).

Do pracy z plikami SAM/BAM używamy narzędzia `samtools`.

[Pełna dokumentacja formatu SAM](https://samtools.github.io/hts-specs/SAMv1.pdf)

---

### Variant Calling – identyfikacja różnic genetycznych.

Po zmapowaniu odczytów do referencji możemy zidentyfikować warianty genetyczne.

**Rodzaje małych wariantów:**

- **SNP / SNV** – pojedyncze zmiany nukleotydów
- **INDEL** – małe insercje lub delecje (do 50 bp)

**Rodzaje wariantów strukturalnych:**

- **INDEL** – duże insercje lub delecje (powyżej 50 bp)
- **Duplikacje**
- **Inwersje**
- **Translokacje**

Warianty przedstawiane są w formacie **VCF**, który zawiera informacje m. in. o pozycji genomie, allelach, jakości i genotypie próbki.

[Pełna dokumentacja formatu VCF](https://samtools.github.io/hts-specs/VCFv4.3.pdf)

---

### IGV – Integrative Genomics Viewer

[IGV (Integrative Genomics Viewer)](https://software.broadinstitute.org/software/igv/) to popularne narzędzie do wizualizacji danych genomicznych.

- Umożliwia przeglądanie sekwencji referencyjnej wraz z dopasowanymi odczytami, adnotacjami genów oraz wykrytymi wariantami.
- Obsługuje standardowe formaty stosowane w genomice i transkryptomice (BAM, VCF, GTF...)
- Umożliwia m. in. ocenę jakości sekwencjonowania i mapowania, a także wizualną ocenę zmienności

---

## Cel zajęć

Celem zajęć jest **praktyczne zapoznanie się z podstawowymi elementami analizy danych NGS** – na przykładzie uproszczonego workflow:

- mapowanie odczytów do genomu referencyjnego (`bwa mem`),
- analiza plików SAM (`samtools`),
- identyfikacja wariantów genetycznych (`bcftools`),
- wizualizacja wyników (`IGV`).

---

## Ćwiczenia


### Uruchomienie środowiska

1. Uruchom **VirtualBox** i wystartuj maszynę wirtualną z Ubuntu.
2. Zaloguj się – login: `student`, hasło: `student`.
3. Otwórz terminal (skrót: **Ctrl + Alt + T**).

---

### Połączenie z serwerem

W terminalu wpisz:

```
ssh biotech25@IP
```

Hasło i adres serwera udostępnione jest na sali ćwiczeń.

---

### Aktywacja środowiska `Conda`

W terminalu wpisz:

```
conda activate mapowanie
```

Polecenie aktywuje środowisko, w którym zainstalowane są wszystkie potrzebne narzędzia: `bwa`, `samtools`, `bcftools`, itp.

---

### Sprawdzenie danych wejściowych

Wpisz:

```
ls ~/dane
```

Polecenie `ls` (*list*) pokazuje zawartość katalogu. Znak `~` określa katalog domowy.

Zajrzyj do pliku FASTQ – czyli surowych danych z sekwencjonowania, korzystając z komendy `head`, która domyślnie wyświetla 10 pierwszych linijek pliku, natomiast liczbę tę możemy modyfikować przełącznikiem `-`:

```
head -4 ~/dane/gt2_paired_1.fq
```

**Zadanie:** Co oznacza ostatnia (czwarta) linijka rekordu FASTQ?



Policzmy wszystkie linijki w pliku, korzystając z `cat`, kon**kat**enującego zawartość plików (w tym wypadku jednego, więc po prostu wyświetlana jest jego zawartość), i  `wc -l` czyli *word count -lines*:

```
cat ~/dane/gt2_paired_1.fq | wc -l
```

- `|` operator "pipe", pozwala na bezpośrednie przekierowanie wyników poprzedzającego je polecenia do następującego.

**Zadanie:** Ile odczytów znajduje się w każdym z plików *fastq*?

---

### Utwórz własny katalog roboczy

Korzystając z `mkdir` stwórz swój katalog roboczy nazywając go zgodnie ze swoim uniwersyteckim ID (**TWOJE_ID**@st.amu.edu.pl).

```
mkdir ~/studenci/TWOJE_ID
```

Za pomocą `cd` (*change directory*) przejdź do tego katalogu:

```
cd ~/studenci/TWOJE_ID
```

---


### Skopiuj dane do swojego katalogu

Wpisz:

```
cp ~/dane/* .
```

- `cp` *copy*
- `~/dane/*` oznacza „wszystkie pliki z folderu `~/dane`”
- `.` oznacza „tutaj”, czyli do folderu, w którym jesteś

Aby sprawdzić, czy dane znajdują się w Twoim katalogu wpisz `ls .` lub po prostu `ls`. 
`ls` warto stosować kontrolnie za każdym razem, kiedy tworzysz lub usuwasz nowe pliki.

---

### Zamontowanie zdalnego katalogu

1. Otwórz menedżer plików (ikonka „Files” na pasku), 
2. Przejdź do: **Other Locations**
3. Wpisz adres: `sftp://biotech25@IP`
4. Wprowadź hasło

Dzięki temu zdalny katalog dostępny będzie jak klasyczny folder. 
Znawiguj do swojego katalogu i upewnij się, czy wszystkie poprzednio skopiowane pliki są w nim widoczne.

---

## Analiza 

### Krok 1. Mapowanie odczytów (BWA)

`bwa mem` jest jednym z najczęściej stosowanych narzędzi do przyrównywania krótkich odczytów. Opiera sie o algorytm oparty o słowa, nieco podobny do omawianego wczesniej BLASTa. Dlatego pierwszym krokiem mapowania jest stworzenie indeksu genomu zawierajacego zbiór słów i ich lokalizację.

```
bwa index GCA_007859695.1_chr20.fasta
```

Następnie mapujemy odczyty (FASTQ) do genomu:

```
bwa mem -t 5  GCA_007859695.1_chr20.fasta \
gt2_paired_1.fq gt2_paired_2.fq > aln.sam
```

- `t`  ilość "wątków" (*threads*), czyli równolegle wykonywanych obliczeń.
- `> aln.sam`  zapisujemy wynik do pliku `aln.sam`
- `\`  "escape", pozwala potraktować znak specjalny dosłownie – w tym przypadku oznacza kontynuację polecenia w kolejnej linii. (poprzedza `enter`, który bez `\` zatwierdza uruchomienie polecenia)

**Pytanie:** Czy mapowanie odczytów to dopasowanie lokalne czy globalne?

Sprawdź zawartość pliku:

```
head aln.sam
```

Policz linijki:

```
cat aln.sam | wc -l
```

Aby policzyć dopasowania policz linie, pomijając linie nagłówka, zaczynające się od `@`:

```
grep -v "^@" aln.sam | wc -l
```
`grep` zwraca linijki, w których znaleziono konkretne słowo (bądź wyrażenie). 
`-v` odwraca logikę

---

### Krok 2: Konwersja SAM → BAM i sortowanie

*SAM* to plik tekstowy – *BAM* to jego skompresowana, binarna wersja. 

`samtools view` umożliwia m. in. wyświetlanie zawartości plików *sam* i *bam* i konwersję pomiędzy formatami:

```
samtools view -bS aln.sam > aln.bam
```

Pliki bam nie są czytelne dla człowieka:

```
head aln.bam
```


`samtools sort` domyślnie sortuje rekordy pod względem pozycji w genomie:

```
samtools sort aln.bam -o aln_sorted.bam
```

`samtools index` tworzy indeks pliku *bam* o rozszerzeniu *.bai*, który jest "mapą" pliku, umożliwiącą narzędziom szybki dostęp do odczytów przyrównanych do dowolnej części sekwencji referencyjnej.

```
samtools index aln_sorted.bam
```

Zadanie: korzystając z polecenia `du -h plik` (*disk usage*) określ, który z plików - *sam*, czy *bam* jest większy? Zapisz ich wielkość. 

Poniewaz oba zawierają te same dane, usuń ten większy korzystając z polecenia `rm plik` (*remove*). 


---

### Krok 3: Statystyka mapowania

```
samtools flagstat aln_sorted.bam
```

 **Zadanie:**
- Ile było surowych odczytów?
- Ile z nich zostało zmapowanych? Jaki to procent całości?
- Ile odczytów jest poprawnie sparowanych?

---

### 4. Przegląd wyników mapowania w IGV

Pobierz i uruchom program `IGV`. W tym celu otwórz nowy terminal (starego nie zamykaj), następnie wpisz:

```
wget https://data.broadinstitute.org/igv/projects/downloads/2.18/IGV_Linux_2.18.4_WithJava.zip
unzip IGV_Linux_2.18.4_WithJava.zip
IGV_Linux_2.18.4/igv.sh
```

W programie otwórz genom referencyjny i zmapowane do niego odczyty ze swojego zdalnego katalogu:

1. `Genomes → Load Genome from File` i wskaż `GCA_007859695.1_chr20.fasta`
2. `File → Load from File` i wybierz `aln_sorted.bam`

**Pytanie:**
-  Jaka jest (mniej więcej) średnia głębokość pokrycia (*coverage*)?
-  Czy widzisz jakies potencjalne warianty genetyczne? Po czym je rozpoznajesz? Wklej screen z takim wariantem.
-  Czym warianty różnią się od losowych błędów sekwencjonowania?

---

### Krok 5: Detekcja wariantów (bcftools)

Manualna identyfikacja setek tysięcy wariantów różniących genomy byłaby niemożliwa. Istnieje wiele narzędzi stworzonych w tym celu, jednym z klasycznych rozwiązań jest pipeline `bcftools mpileup | bcftools call`, znajdujący jednonukletodydowe substytucje (SNP) i małe insercje/delecje (INDEL).

Na podstawie genomu referencyjnego i zmapowanych odczytów zidentyfikuj małe warianty, i zapisz je do pliku `variants.vcf`:

```
bcftools mpileup -Ou -f GCA_007859695.1_chr20.fasta aln_sorted.bam | \
bcftools call -mv > variants.vcf
```

**Zadanie:** 
- Dołącz plik `variants.vcf` do otwartej wcześniej sesji `IGV`. 
- Z pliku `variants.vcf` wybierz 2 dowolne warianty znajdujace się co najmniej 20000 bp od końców chromosomu i znajdź je w `IGV`. Zamieść screeny każdego z wariantów i odpowiadającego mu rekordu w pliku VCF.
- Policz ile wariantów wykryto.


Wyświetlmy 5 pierwszych substytucji i wariantów homozygotycznych:

```
grep -v "^#" variants.vcf | awk 'length($4)==length($5)' | head -5

grep -v "^#" variants.vcf | awk -F"\t" '$10 ~ /1\/1/' | head -5
```

`awk` to narzędzie służące do filtrowania i przetwarzania tekstu linia po linii, traktując dane oddzielone separatorem jako kolumny.

`awk 'length($4)==length($5)'` – porównuje długości kolumn 4 i 5 (allel referencyjny i alternatywny); wypisuje linie, w których są sobie równe. Odwrotną zależność określisz operatorem `!=`. 

`awk -F"\t" '$10 ~ /1\/1/'` – analizuje kolumnę 10 (genotyp); wypisuje linie, które zawierają wzorzec 1/1. `\` pozwala na dosłowne potraktowanie usytuowanego za nim znaku `/`, który w przeciwnym razie ma znaczenie funkcjonalne. Separator kolumn (`-F`) ustawiony na tabulator (`\t`). Warianty heterozygotyczne oznaczane są `0/1`.



**Zadanie:**
- Ile jest SNPów, INDELi, wariantów homo- i heterozygotycznych? Zapisz wykorzystane komendy.

---

### Krok 7. Warianty w IGV

1. Załaduj plik VCF do IGV: `variants.vcf`  

**Zadanie**:
Odszukaj 5 wariantow heterozygotycznych z pliku variants.vcf w `IGV`. Dla każdego z nich:
- Wklej zrzut ekranu rekordu VCF i widoku w IGV.
- Podaj głębokość sekwencjonowania wariantu.
- Ile odczytów wspiera każdy z alleli? Czy wartości te są spójne z polem `AF`?

## Zadanie programistyczne: analiza głębokości sekwencjonowania

### Cel

Na podstawie danych uzyskanych z pliku `aln_sorted.bam` wykonaj prostą analizę głębokości pokrycia wzdłuż genomu.

---

### Krok 1. Wygenerowanie pliku z głębokością

W terminalu wpisz:

```bash
samtools depth aln_sorted.bam > coverage.txt
```

Plik `coverage.txt` zawiera informacje o głębokości pokrycia dla każdej pozycji w genomie w postaci trzech kolumn oddzielonych tabulatorem:

```
chromosom   pozycja   głębokość
```

---

### Krok 2. Analiza danych w Pythonie

Na podstawie danych zawartych w `coverage.txt`, napisz skrypt w języku Python, który:

1. Wczyta dane do struktury umożliwiającej analizę (np. `pandas.DataFrame`),
2. Obliczy:
   - średnią głębokość pokrycia,
   - minimalną i maksymalną wartość głębokości,
3. Wygeneruje dwa wykresy i zapisze je do plików PNG:
   - wykres liniowy przedstawiający średnią głębokość pokrycia w oknach 1000 bp, przesuwanych co 500 bp (na osi X: środek okna),
   - histogram przedstawiający rozkład wartości głębokości (ile pozycji ma daną wartość głębokości).

---

### Zapis wyników

- Wykres liniowy zapisz jako `coverage_windowed.png`
- Histogram zapisz jako `coverage_histogram.png`
- W skrypcie wypisz na ekranie wartości: średnią, minimum i maksimum głębokości w całym genomie

---

### Zadanie

- Dołącz skrypt w języku Python jako `coverage_analysis.py`
- Do raportu dołącz oba wygenerowane wykresy oraz wypisane statystyki
- W kilku zdaniach opisz, czy głębokość pokrycia jest jednorodna w całym genomie, czy zauważalne są regiony o znacząco niższym lub wyższym pokryciu
