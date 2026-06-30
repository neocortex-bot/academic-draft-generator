---
name: academic-draft-generator
version: 1.0.0
description: |-
  Generate academic literature review drafts (Bahasa Indonesia) from
  PaperQA/HyDE research Q&A CSV data. Mapping pertanyaan ke outline,
  parallel section generation via sub-agents with strict anti-AI style,
  kompilasi ke PDF via weasyprint.
tags: [academic, literature-review, draft-generation, bahasa-indonesia, paperqa]
---

# Academic Draft Generator: Pipeline Telaah Pustaka dari Research Q&A ke PDF

## Kapan Skill Ini Digunakan

Gunakan ketika user memiliki:
1. **Research answers CSV** — output dari PaperQA pipeline (kolom: Question Number;Question;Answer)
2. **Outline file** (`outline.txt`) — struktur bab dengan elaborasi per sub-section
3. **Prompt template** — aturan Academic Draft Helper (gaya anti-AI, format prosa)

Dan user meminta output berupa **draft lengkap dalam Bahasa Indonesia** sebagai PDF.

## Workflow Ringkas

```
1. Baca outline → mapping pertanyaan → seksi
2. Ekstrak Q&A per seksi → simpan sebagai /tmp/qa_{group}.json
3. Delegate parallel sub-agent per seksi (max 3 concurrent)
4. Tulis section 2.4.1 dan seksi yang missed secara lokal
5. Compile semua .md → satu markdown → weasyprint PDF
6. Kirim file ke user
```

## Step-by-Step

### Step 1: Baca Input

Baca `outline.txt` untuk mapping. Baca CSV untuk daftar pertanyaan.

```python
# Mapping kira-kira: tiap baris outline (2.1.1, 2.1.2, ...) → grup pertanyaan
# Pertanyaan 1-5 → definisi/diagnosis
# Pertanyaan 6-9 → epidemiologi/risiko
# ...dst
```

Simpan tiap grup Q&A sebagai `/tmp/qa_{group_name}.json` untuk di-load sub-agent.

### Step 2: Prompt Sub-Agent

Tiap sub-agent menerima:
- `context`: path file JSON + ringkasan isi Q&A + target section + strict style rules
- `goal`: instruksi eksplisit "Tulis bagian X.Y.Z dalam Bahasa Indonesia sebagai prosa akademik mengalir. Minimal N kata. TANPA bullet, TANPA bold, TANPA kata AI klise, TANPA paralelisme negatif, TANPA trailing -ing, TANPA kesimpulan formulaik. Kopula dasar. Sitasi wajib."

### Step 3: Style Rules (Wajib)

Embed aturan ini di setiap goal sub-agent:

**Larangan leksikal (AI vocabulary):**
- `dalam era globalisasi`, `tidak dapat dipungkiri`, `patut dicatat bahwa`
- `menyoroti pentingnya`, `berfungsi sebagai`, `menggarisbawahi`
- `lanskap`, `permadani`, `fondasi utama`, `revolusioner`
- `sangat krusial`, `sangat signifikan`, `tak ternilai`
- `bukan hanya...melainkan`, `tidak hanya...tetapi juga` (paralelisme negatif)

**Larangan struktural:**
- TANPA bullet points / numbered lists — semuanya prosa paragraf
- TANPA bold dalam teks
- TANPA trailing present-participle ("yang menunjukkan...", "yang menyoroti...")
- TANPA kesimpulan formulaik ("Kesimpulannya", "Secara keseluruhan", "Meskipun terdapat tantangan")
- TANPA kalimat disclaimer ("Penting untuk dicatat bahwa")
- TANPA false ranges ("Dari tingkat seluler hingga populasi") kecuali skala kuantitatif

**Kewajiban:**
- Kopula dasar: "adalah", "merupakan", "memiliki" — BUKAN "berfungsi sebagai", "menawarkan"
- Sitasi dipertahankan format asli: `(Sliwa2010a pages 3-4)`
- Definisi singkatan pada first use: "Peripartum cardiomyopathy (PPCM)"
- Gaya impersonal, sudut pandang ketiga
- Parafrase, bukan kutipan langsung
- Nada kering, faktual, akademik
- Heading: sentence case ("Definisi peripartum cardiomyopathy")
- 1 deskriptor kuat per konsep (patahkan "rule of three")

### Step 4: Parallel Generation

Gunakan `delegate_task(tasks=[...])` dengan max 3 concurrent tasks.

**Kelompok seksi:**

| Grup | Q | Section |
|------|---|---------|
| definisi | 1-5 | 2.1.1 Definisi, kriteria diagnostik, DD |
| epidemiologi | 6-9 | 2.1.2 Klasifikasi risiko & epidemiologi |
| hemodinamik | 10-14 | 2.2.1 Perubahan hemodinamik & farmakokinetik |
| gdmt | 15-20 | 2.3 Four pillars GDMT non-hamil |
| raas | 21-24 | 2.4.1 RAAS/ARNI kontraindikasi |
| mra_sglt2 | 25-28 | 2.4.2 MRA & SGLT2i |
| beta_blocker | 29-32 | 2.4.3 Beta-blocker pilar utama |
| hydralazine | 33-36 | 2.4.4 Hydralazine-ISDN substitusi |
| prekonsepsi | 37-40 | 2.5.1 Fase pra-konsepsi |
| kehamilan | 41-46 | 2.5.2 Fase kehamilan trimester 1-3 |
| persalinan | 47-52 | 2.5.3 Fase persalinan & immediate postpartum |
| laktasi | 53-54 | 2.5.4 Fase laktasi & reintroduksi |
| bias | 68,69,73,74,77,78,79 | 2.6.1 Keterbatasan bukti & bias |
| realworld | 70,71,72,76 | 2.6.2 Extrapolasi & real-world data |
| shared | 80,81,82,83,84 | 2.6.3 Individualized care & SDM |
| antikoagulasi | 60-63 | 2.7.1 Antikoagulasi LVEF ≤30% |
| bromocriptine | 55-59 | 2.7.2 Bromocriptine pada PPCM |
| device | 64-67 | 2.7.3 Device & MCS |

### Step 5: Cek Output Sub-Agent

Setelah tiap batch, sub-agent biasanya menulis file ke `/home/linuxmint/`. Cek dengan:
```bash
ls -la /home/linuxmint/{bab_,bagian_,seksi_}* 2>/dev/null
```

Jika ada seksi yang tidak ditulis file (sub-agent hanya output di summary), tulis manual berdasarkan data Q&A.

### Step 6: Compile & PDF

```python
# Urutan file sesuai outline
files = [
    "bab_2_1_1_ppcm.md", "bagian_2_1_2_ppcm.md", "bagian_2_2_1.md",
    "bab23_gdmt_hfref.md", "bab_2_4_1_raas_kontraindikasi.md",
    "bagian_2_4_2.md", "bagian_2_4_3.md", "bab_2_4_4_hydralazine_isdn.md",
    "bab_2_5_1_pra_konsepsi.md", "bab_2_5_2_kehamilan_trimester.md",
    "bab_2_5_3_persalinan.md", "seksi_2_5_4_laktasi_reintroduksi.md",
    "bagian_261_keterbatasan_bukti.md", "bab_2_6_2_extrapolasi_rwe.md",
    "bagian_2_6_3.md", "bagian_2_7_1_antikoagulasi.md",
    "bagian_2_7_2_bromocriptine_ppcm.md", "bab_2_7_3_device_mcs_ppcm.md"
]

# Combine with chapter header
# Convert markdown → HTML → PDF via weasyprint
from weasyprint import HTML
import markdown
html_body = markdown.markdown(combined_md, extensions=['extra'])
HTML(string=html).write_pdf(output_path)
```

Pastikan `pip install weasyprint markdown` tersedia.

### Step 7: Deliver

Kirim file PDF via `MEDIA:/path/to/draft.pdf` di response.

## Pitfalls

1. **Sub-agent tidak menulis file** — beberapa sub-agent hanya mengembalikan summary dengan teks embedded, bukan menulis file. Selalu cek filesystem setelah tiap batch. Tulis manual jika perlu.
2. **CSV delimiter** — file dari `research_answers_deepseek_clean.csv` menggunakan `;` delimiter dengan pipe `|` di header line. Python csv reader mungkin bingung; parse manual dengan `.split(';')`.
3. **Q&A terpotong** — beberapa jawaban PaperQA dipotong di JSON. Ambil data langsung dari CSV asli via terminal grep untuk versi lengkap.
4. **Sub-agent sitasi keluar dari sumber** — sub-agent kadang menambahkan sitasi yang tidak ada di data Q&A (hallucination). Periksa dan patch jika perlu. Atau beri peringatan di prompt: "HANYA gunakan sitasi yang ada di data Q&A, jangan buat baru."
5. **PDF dependencies** — weasyprint perlu system packages. Fallback: kirim file markdown saja jika PDF gagal.
6. **Word count** — sub-agent sering melebihi target. It's fine; lebih baik kelebihan daripada kurang.
7. **Nama file inkonsisten** — sub-agent punya kebiasaan naming sendiri. Selalu ls setelah batch untuk tahu path aktual.
8. **Batch size** — max 3 concurrent tasks via delegate_task. Loop dalam batch of 3.
