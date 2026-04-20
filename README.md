# tardigrade-lab-journal
Laboratory journal for Project 4: Tardigrade genome analysis (Ramazzottius varieornatus). Bioinformatics course. #Лабораторный журнал для проекта 4: Анализ генома тихоходки (Ramazzottius varieornatus). Курс биоинформатики.
# Лабораторный журнал
**Проект 4: Анализ генома тихоходки *Ramazzottius varieornatus***  
**Курс: Анализ геномных данных**

---

## 20.04.2025 — Настройка окружения

Создал conda-окружение и установил необходимые инструменты.

```bash
conda create -n tardigrades -y
conda activate tardigrades
conda install -c bioconda -c conda-forge blast samtools seqtk -y
```

Проверил установку:
```bash
blastp -version
samtools --version
seqtk
```
Все инструменты установлены успешно.

---

## 20.04.2025 — Скачивание данных

Скачал геном, предварительно рассчитанные результаты AUGUSTUS и пептиды хроматиновой фракции.

```bash
# Геном
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/001/949/185/GCA_001949185.1_Rvar_4.0/GCA_001949185.1_Rvar_4.0_genomic.fna.gz
gunzip GCA_001949185.1_Rvar_4.0_genomic.fna.gz

# Белки AUGUSTUS и GFF (Google Drive)
gdown 1hCEywBlqNzTrIpQsZTVuZk1S9qKzqQAq  # augustus.whole.aa
gdown 12ShwrgLkvJIYQV2p1UlXklmxSOOxyxj4  # augustus.whole.gff

# Скрипт getAnnoFasta
wget http://augustus.gobics.de/binaries/scripts/getAnnoFasta.pl

# Пептиды хроматиновой фракции (Яндекс Диск)
YADISK_URL=$(curl -s "https://cloud-api.yandex.net/v1/disk/public/resources/download?public_key=https://disk.yandex.ru/d/xJqQMGX77Xueqg" | python3 -c "import sys,json; print(json.load(sys.stdin)['href'])")
wget -O peptides.fa "$YADISK_URL"
```

---

## 20.04.2025 — Подсчёт предсказанных белков

```bash
grep -c ">" augustus.whole.aa
```

**Результат: 16 435 белков** — соответствует ожидаемым 15 000–20 000 для многоклеточных организмов.

---

## 20.04.2025 — BLAST: поиск пептидов в протеоме

Создал локальную базу данных из белков AUGUSTUS, затем выполнил поиск пептидов хроматиновой фракции.

```bash
makeblastdb -in augustus.whole.aa -dbtype prot -out tardigrade_db

blastp -db tardigrade_db \
       -query peptides.fa \
       -outfmt "6 qseqid sseqid evalue qcovs pident stitle" \
       -out blast_results.txt \
       -evalue 1e-5

wc -l blast_results.txt   # → 5 хитов
cat blast_results.txt
```

**Результат:** 5 хитов, соответствующих **3 уникальным белкам-кандидатам**: g4106.t1, g12510.t1, g15484.t1.  
Все хиты: идентичность 100%, покрытие запроса 100%, E-value ≤ 1e-5.

---

## 20.04.2025 — Извлечение последовательностей кандидатов

```bash
samtools faidx augustus.whole.aa
samtools faidx augustus.whole.aa g4106.t1 g12510.t1 g15484.t1 > candidates.fa
cat candidates.fa
```

Три белковые последовательности успешно извлечены.

---

## 20.04.2025 — Предсказание субклеточной локализации

### WoLF PSORT (animal)
| Белок | Предсказание | Score |
|-------|-------------|-------|
| g4106.t1 | ЭПР / секреторный путь | ER: 14.5 |
| g12510.t1 | Плазматическая мембрана | 29 |
| g15484.t1 | **Ядро** | 17.5 |

### TargetP 2.0 (non-plant)
| Белок | Предсказание | Вероятность |
|-------|-------------|-------------|
| g4106.t1 | Other | 0.73 (SP: 0.27) |
| g12510.t1 | Other | 0.9997 |
| g15484.t1 | Other | 1.0 |

---

## 20.04.2025 — BLAST по базе UniProtKB/Swiss-Prot

| Белок | Лучший хит | E-value | Идентичность | Покрытие |
|-------|-----------|---------|-------------|---------|
| g4106.t1 | Нет хитов | — | — | — |
| g12510.t1 | Нет хитов | — | — | — |
| g15484.t1 | VPS51 гомолог (*Danio rerio*) | 0.0 | 45% | 78% |

g15484.t1 имеет гомологию с VPS51 у нескольких видов метазоа: человек, мышь, данио, дрозофила.

---

## 20.04.2025 — Предсказание доменов Pfam (HMMER)

| Белок | Домен | Accession | E-value |
|-------|-------|-----------|---------|
| g4106.t1 | Не найдено | — | — |
| g12510.t1 | Не найдено | — | — |
| g15484.t1 | VPS51/EXO84 N-концевой | PF08700 | 8.7e-24 |
| g15484.t1 | Экзоцистный комплекс Sec5 | PF15469 | 1.6e-21 |
| g15484.t1 | Комплекс GARP (Vps54) | PF10475 | 3.3e-10 |

---

## Итоговая таблица

| | g4106.t1 | g12510.t1 | g15484.t1 |
|---|---|---|---|
| BLAST (Swiss-Prot) | Нет хитов | Нет хитов | VPS51 (E=0.0, 45%) |
| Домены Pfam | Нет | Нет | VPS51_Exo84_N, Sec5, GARP |
| WoLF PSORT | ЭПР / секреторный | Плазм. мембрана | **Ядро** |
| TargetP | Other (SP? 0.27) | Other (0.9997) | Other (1.0) |

**Основной кандидат для экспериментальной верификации: g15484.t1** — ядерная локализация + присутствие в хроматиновой фракции + гомология с VPS51 указывают на возможную двойную (moonlighting) ДНК-ассоциированную функцию.
