---
name: academic-draft-generator
version: 1.6.0
description: |-
  Generate academic literature review drafts (Bahasa Indonesia) from
  research Q&A data (CSV). Mapping pertanyaan ke outline generik,
  parallel section generation via sub-agents with strict anti-AI style,
  post-processing polish via paper-humanizer-id, Bab 1-3 generation,
  audience-tailored compaction, kompilasi ke DOCX/PDF via python-docx
  or weasyprint. General purpose — works for any scientific/medical topic.
tags: [academic, literature-review, draft-generation, bahasa-indonesia, general-purpose, anti-ai]
related_skills:
  - paper-humanizer-id
  - humanize-writing
---

# Academic Draft Generator: Pipeline Telaah Pustaka dari Research Q&A ke PDF

## Kapan Skill Ini Digunakan

Gunakan ketika user memiliki:
1. **Research answers CSV** — output dari PaperQA pipeline (kolom: Question Number;Question;Answer)
2. **Outline file** (`outline.txt`) — struktur bab dengan elaborasi per sub-section
3. **Prompt template** — aturan Academic Draft Helper (gaya anti-AI, format prosa)

Dan user meminta output berupa **draft lengkap dalam Bahasa Indonesia** sebagai PDF.

### Target Volume: 20-30 Halaman A4

Target 20-30 halaman A4 (~400-500 kata per halaman dengan font 12pt, spasi 1.5, margin 2.5cm) berarti total **10.000-15.000 kata**. Distribusi:

| Tingkat | Jumlah | Kata per item | Subtotal |
|---------|--------|---------------|----------|
| Section utama (2.1, 2.2...) | ~7 | 800-1200 | 5.600-8.400 |
| Sub-section (2.1.1, 2.1.2...) | ~11 | 400-700 | 4.400-7.700 |
| **Total** | **18** | | **10.000-16.000** |

Gunakan ini sebagai panduan saat menentukan target word count per sub-agent.

## Integrasi Skill Terkait

Pipeline ini mengintegrasikan dua skill anti-AI yang saling melengkapi:

| Skill | Peran | Bahasa | Sumber |
|-------|-------|--------|--------|
| paper-humanizer-id | Post-processing — polish final draft, hapus AI patterns Indonesia | Indonesia | kustom |
| humanize-writing | Style rules baseline — detection heuristics, tier-1/tier-2 vocabulary, 8-pass editing framework | Inggris/adaptatif | jpeggdev/humanize-writing |

### Cara Integrasi

1. Pre-flight: skill_view("paper-humanizer-id") dan skill_view("humanize-writing") di awal sesi untuk memuat aturan gaya.
2. Style rules sub-agent: ambil larangan leksikal dari kedua skill dan gabungkan ke setiap goal sub-agent (lihat Step 3 Extended AI Vocabulary).
3. Post-processing: setelah semua seksi terkumpul dan sebelum kompilasi PDF, jalankan polish pass dengan paper-humanizer-id (Step 6b).
4. Detection heuristic: gunakan adapted checklist dari humanize-writing untuk QA akhir draft.

## Workflow Ringkas

```
1. Baca outline → mapping pertanyaan → seksi
2. Ekstrak Q&A per seksi & closed-set sitasi
3. Delegate parallel sub-agent per seksi (max 3 concurrent)
4. Tulis section yang missed secara lokal
5. Verifikasi sitasi — rewrite ulang yg 0 cites
6. Compile Bab 2 → generate Bab 1 (naratif 5 paragraf dengan sitasi dari Bab 2)
7. Generate Bab 3 (3 paragraf compact tanpa sitasi)
8. Opsional: compact Bab 2 untuk target audience (potong elaborasi molekuler)
9. Gabung Bab 1 + Bab 2 + Bab 3 → kompilasi DOCX dan/atau PDF
10. Verifikasi heading setiap section cocok persis dengan outline
11. Kirim file ke user
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

### Step 1b: Pre-ekstraksi Sitasi dari CSV (WAJIB — cegah sub-agent menulis tanpa sitasi)

Sebelum delegasi ke sub-agent, EKSTRAK SEMUA sitasi dari CSV asli dan mapping per grup section. Ini adalah langkah KRITIS karena sub-agent sering menulis paragraf tanpa sitasi sama sekali jika tidak diberikan data sitasi sebagai closed set.

```python
import re, json

with open('research_answers_deepseek_clean.csv', 'r') as f:
    text = f.read()

# Ekstrak semua (AuthorYear pages X-Y) dari seluruh CSV
all_citations = re.findall(r'[A-Z][a-zA-Z]+\d{4}[a-z]?\s+pages\s+\d+-\d+', text)

# Mapping per grup section
groups = {
    'definisi': ['1','2','3','4','5'],
    'epidemiologi': ['6','7','8','9'],
    # ... semua grup
}

for gname, qnums in groups.items():
    group_citations = set()
    for qn in qnums:
        if qn in qa_data:
            group_citations.update(extract_cites(qa_data[qn]))
    with open(f'/tmp/qa_cites_{gname}.json', 'w') as f:
        json.dump(sorted(group_citations), f)
```

**Output:** `/tmp/qa_cites_{group}.json` berisi daftar closed-set sitasi yang valid untuk section tersebut. Sub-agent HARUS hanya menggunakan sitasi dari file ini — TIDAK BOLEH menambahkan sitasi lain.

**Kasus khusus:** Jika CSV tidak mengandung `pages X-Y` untuk grup tertentu (contoh Q6-Q9 epidemiologi = 0 citations), ambil sitasi dari pertanyaan Q&A lain yang relevan, atau dari outline itu sendiri. Jangan panggil sub-agent tanpa data sitasi — tulis manual.

### Step 2: Prompt Sub-Agent — Wajib Sertakan Closed-Set Sitasi

Tiap sub-agent menerima:

- `context`: path file JSON Q&A + **path file JSON sitasi** (dari Step 1b) + ringkasan isi Q&A + target section + strict style rules
- `goal`: instruksi eksplisit dengan format berikut. **WAJIB** cantumkan daftar sitasi sebagai closed set:

```
Tulis bagian X.Y.Z dalam Bahasa Indonesia sebagai prosa akademik mengalir.

KRITERIA WAJIB:
1. SETIAP paragraf (100%) harus diakhiri minimal SATU sitasi dalam format persis: (AuthorYear pages X-Y)
2. HANYA gunakan sitasi dari daftar berikut — JANGAN buat sitasi baru:
   [list semua sitasi untuk grup ini]
3. Panjang: 500-700 kata
4. TANPA bullet, TANPA bold, TANPA kata AI klise, TANPA paralelisme negatif, TANPA trailing -ing, TANPA kesimpulan formulaik
5. Kopula dasar (adalah, merupakan). Sitasi wajib dipertahankan format asli.
```

**PENTING:** Jika data sitasi kosong (0 citations untuk grup ini), jangan panggil sub-agent. Tulis section manual berdasarkan konteks outline dan pengetahuan umum, dengan sitasi yang diambil dari sumber terdekat yang tersedia.

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

### Extended AI Vocabulary (Gabungan humanize-writing + paper-humanizer-id)

Selain larangan di atas, gabungkan vocabulary dari kedua skill terkait:

**Tier 1 — Red flag (HINDARI):**
- Inggris: delve, landscape (metafora), tapestry, paradigm shift, leverage, harness, navigate (metafora), realm, embark on journey, myriad, plethora, multifaceted, groundbreaking, revolutionize, synergy, ecosystem (non-teknis), resonate, streamline
- Indonesia: menyelami, mengupas, membedah, menonjolkan, menggarisbawahi, memupuk, lanskap, permadani, fondasi utama, bukti nyata, panggung utama

**Tier 2 — Suspicious dalam cluster (3+ dalam satu tulisan):**
- Inggris: robust, seamless, cutting-edge, innovative, comprehensive, pivotal, nuanced, compelling, transformative, bolster, underscore, evolving, fostering, imperative, intricate, overarching, unprecedented
- Indonesia: dalam rangka meningkatkan, tidak terlepas dari, guna mencapai, sebagaimana mestinya, pada hakikatnya, oleh karena itu maka

**Frasa pengisi (fluff) — English:** serves as, stands as, represents (berlebihan), boasts, features, offers (kopula), it is worth noting, it is important to note, due to the fact that, at this point in time, in order to

**Frasa pengisi (fluff) — Indonesia:** perlu diperhatikan bahwa, pada dasarnya, dapat dikatakan bahwa, berfungsi sebagai pengingat, memainkan peran penting, menyoroti pentingnya, menjadi tonggak sejarah

**Kata sifat hiperbolik:** sangat signifikan, sangat penting, sangat besar, luar biasa, sangat krusial, revolusioner, tak ternilai

### Detection Heuristic (Adapted from humanize-writing)

Setelah draft selesai, skoring untuk deteksi AI residual. Jika 5+ dari checklist ini terdeteksi, jalankan post-processing:

- [ ] Mengandung 3+ kata Tier 1 (Inggris atau Indonesia)
- [ ] Mengandung 5+ kata Tier 2
- [ ] Ada frasa pengisi filler ("perlu diperhatikan bahwa", "penting untuk dicatat")
- [ ] Setiap seksi mengikuti struktur paragraf yang identik
- [ ] Paralelisme negatif muncul lebih dari 1x
- [ ] Em dash muncul lebih dari 1x per 3 paragraf
- [ ] Tidak ada kalimat pendek (under 10 kata)
- [ ] Tidak ada variasi panjang kalimat
- [ ] Kesimpulan formulaik ("Kesimpulannya", "Secara keseluruhan")
- [ ] Kopula avoidance ("berfungsi sebagai" bukan "adalah")
- [ ] Klaim absolut tanpa konteks keterbatasan
- [ ] Transisi generik berulang ("Selain itu" di awal kalimat berturut-turut)
- [ ] Significance inflation di paragraf pembuka/penutup

### Step 4: Parallel Generation

Gunakan `delegate_task(tasks=[...])` dengan max 3 concurrent tasks.

**Mapping pertanyaan Q&A ke outline user:**

Outline sudah disediakan oleh user (`outline.txt`). Cara mapping:

1. Baca outline user — catat semua heading (2.1, 2.1.1, 2.2...)
2. Baca CSV — baca semua pertanyaan sesuai topik
3. Kelompokkan secara manual per section outline berdasarkan kesesuaian topik
4. Ekstrak closed-set sitasi per grup (Step 1b)
5. Delegasikan sub-agent per grup dengan daftar sitasi masing-masing

Contoh mapping untuk referat PPCM (hanya ilustrasi):
```
2.1.1 → Q1-5 (definisi, diagnosis PPCM)
2.1.2 → Q6-9 (epidemiologi, mWHO)
2.2.1 → Q10-14 (hemodinamik kehamilan)
2.3 → Q15-20 (GDMT HFrEF)
2.4.1 → Q21-24 (RAAS/ARNI kontraindikasi)
2.4.2 → Q25-28 (MRA/SGLT2i)
2.4.3 → Q29-32 (beta-blocker)
2.4.4 → Q33-36 (hydralazine-ISDN)
2.5.1 → Q37-40 (pra-konsepsi)
2.5.2 → Q41-46 (kehamilan trimester 1-3)
2.5.3 → Q47-52 (persalinan)
2.5.4 → Q53-54 (laktasi)
2.6.1 → Q68,69,73,74,77,78,79
2.6.2 → Q70,71,72,76
2.6.3 → Q80,81,82,83,84
2.7.1 → Q60-63
2.7.2 → Q55-59
2.7.3 → Q64-67
```

### Step 5: Cek Output Sub-Agent

Setelah tiap batch, sub-agent biasanya menulis file ke `/home/linuxmint/`. Cek dengan:
```bash
ls -la /home/linuxmint/{bab_,bagian_,seksi_}* 2>/dev/null
```

Jika ada seksi yang tidak ditulis file (sub-agent hanya output di summary), tulis manual berdasarkan data Q&A.

### Step 5b: Verifikasi Sitasi — Setiap Paragraf Wajib Punya Sitasi

Setelah semua seksi terkumpul, jalankan verifikasi sitasi DARI SETIAP FILE sebelum kompilasi:

```python
import re
for fname in all_section_files:
    content = open(fname).read()
    paras = [p.strip() for p in content.split('\\n\\n') 
             if len(p.strip()) > 100 and not p.startswith('#')]
    no_cite = []
    for i, p in enumerate(paras):
        if not re.search(r'pages \d', p):
            no_cite.append(i+1)
    if no_cite:
        print(f'{fname}: {len(no_cite)}/{len(paras)} paras TANPA sitasi → {no_cite}')
```

**Tindakan jika ditemukan paragraf tanpa sitasi:**
1. Catat nomor paragraf dan section
2. Delegasikan ulang section tersebut dengan instruksi LEBIH ketat: "TULIS ULANG. SETIAP paragraf HARUS diakhiri (AuthorYear pages X-Y). Hanya dari daftar: [closed set]"
3. Verifikasi ulang setelah selesai

**Jangan lanjut ke kompilasi PDF** sebelum semua section memiliki 100% paragraf bersitasi.

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

### Step 6b: Post-processing Polish dengan paper-humanizer-id

Setelah semua seksi terkumpul dalam satu file markdown (sebelum konversi ke PDF), lakukan polish pass:

1. Load skill: skill_view("paper-humanizer-id") — memuat aturan PUEBI, hapus AI patterns Indonesia, perbaiki koherensi paragraf.
2. Baca file markdown gabungan.
3. Proses per paragraf dengan aturan paper-humanizer-id:
   - Hapus frasa pembuka generik ("dalam era globalisasi", "tidak dapat dipungkiri")
   - Hapus AI-speak Indonesia ("dalam rangka", "guna mencapai", "tidak terlepas dari")
   - Perbaiki repetisi pola kalimat
   - Kurangi generalisasi berlebihan
   - Hapus kata sifat hiperbolik ("sangat", "luar biasa")
   - Perbaiki pola terjemahan literal dari Inggris
   - Pastikan PUEBI dan konsistensi istilah
4. Jangan ubah: data, angka, sitasi, makna inti, struktur argumen.
5. Simpan hasil polish.

### Step 6c: Detection Heuristic Check

Sebelum konversi ke PDF, jalankan Detection Heuristic (lihat Extended AI Vocabulary di atas). Jika 5+ item terdeteksi, ulangi Step 6b dengan target spesifik pada pola yang terdeteksi.

### Step 7: Deliver

Kirim file PDF via `MEDIA:/path/to/draft.pdf` di response.

### Step 8: Generate Bab 1 (Pendahuluan) — 5 Paragraf Naratif

Setelah Bab 2 selesai, generate Bab 1 sebagai pendahuluan yang **memperkenalkan masalah penelitian dengan menjadikan isi Bab 2 sebagai landasan argumen**.

**Struktur 5 Paragraf (TANPA sub-judul — teks mengalir bersambung):**

| Paragraf | Fungsi | Teknik Menulis |
|----------|--------|----------------|
| 1 | Hook — konteks umum topik | Mulai dengan pernyataan menarik tentang topik Bab 2. Tunjukkan relevansi. Ciptakan "kail" pembaca. |
| 2 | Penghubung — mengerucut ke masalah | Gunakan konsep kunci Bab 2 untuk tunjukkan area kompleks/problematic. Transisi ke spesifik. |
| 3 | Research gap — inti argumen | Tunjukkan secara eksplisit kesenjangan riset, perdebatan belum tuntas, masalah praktis belum terjawab. Jelaskan signifikansi gap. |
| 4 | Tujuan penelitian | Jawaban langsung atas gap. Kalimat jangkar: "Oleh karena itu, studi ini bertujuan untuk..." |
| 5 | Penutup — kontribusi & peta jalan | Kontribusi/manfaat studi. Peta jalan ke Bab 2 sebagai 'teaser'. |

**Teknik Penulisan:**

- **Argumentatif, bukan deskriptif**: Gunakan materi Bab 2 untuk membangun argumen _mengapa_ tinjauan ini penting — jangan hanya mendeskripsikan isi Bab 2.
- **Ambil sitasi dari Bab 2**: Setiap klaim faktual harus didukung sitasi yang sama dengan yang digunakan di Bab 2. Buka file Bab 2 dan kutip referensi yang relevan.
- **Alur logis**: Tiap paragraf membangun argumen dari paragraf sebelumnya. Paragraf 1 → 2 → 3 (gap) → 4 (tujuan) → 5 (roadmap).
- **Jangan gunakan sitasi di paragraf 5** (peta jalan ke Bab 2 tidak perlu sitasi).
- **5-7 paragraf** total. Idealnya 5 paragraf, maksimal 7.
- **Panjang: 800-1.200 kata**.

**Prompt template untuk sub-agent (jika didelegasikan):**

```
Anda adalah asisten penulis akademik. Buatkan Bab 1 (Pendahuluan) untuk telaah pustaka tentang [TOPIC].

Baca file Bab 2 di /home/linuxmint/draft_aulia_revisi.md sebagai sumber konten.

KRITERIA:
1. 5-7 paragraf naratif TANPA sub-judul (jangan gunakan "Latar Belakang", "Rumusan Masalah", dll)
2. Struktur: hook (umum) → penghubung (spesifik) → research gap → tujuan → kontribusi & roadmap
3. Argumentatif — gunakan Bab 2 untuk bangun argumen MENGAPA tinjauan penting, bukan deskripsi isi Bab 2
4. Setiap paragraf fakta WAJIB diakhiri sitasi format (AuthorYear pages X-Y) — ambil dari Bab 2
5. Paragraf penutup (roadmap) TANPA sitasi
6. TANPA bullet, TANPA bold, TANPA AI klise, TANPA kesimpulan formulaik
7. Panjang: 800-1.200 kata Bahasa Indonesia prosa mengalir
```

**Setelah selesai:** Gabungkan Bab 1 + Bab 2 → kompilasi ulang PDF.

### Step 9: Compile Final PDF (Bab 1 + Bab 2 + Bab 3)

```
files = ["bab_1_pendahuluan.md"] + bab2_files + ["bab_3_kesimpulan.md"]
# Combine, remove --- separators, convert to PDF
```

### Step 10: Generate Bab 3 (Kesimpulan) — 3 Paragraf Compact

Setelah Bab 1 dan Bab 2 selesai, generate Bab 3 sebagai kesimpulan yang merangkum argumen utama.

**Struktur 3 Paragraf (tanpa sub-judul):**

Setiap paragraf secara implisit mewakili satu pilar, tanpa menyebut 'pertama', 'kedua', atau 'ketiga':

| Paragraf | Pilar implisit | Isi |
|----------|---------------|-----|
| 1 | **Strategi substitusi sekuensial** | Dekonstruksi 4 pillars, reposisi beta-blocker, hidralazin-ISDN, fase pra-konsepsi hingga laktasi |
| 2 | **Kesenjangan bukti** | Eksklusi RCT, MRA/SGLT2i tanpa data manusia, bromokriptin dilemma, shared decision-making |
| 3 | **Sintesis dan arah ke depan** | Reintroduksi bertahap GDMT postpartum, terapi adjuvan, MCS bridge-to-recovery, registri prospektif |

**Aturan penulisan:**
- Tepat 3 paragraf, jangan lebih
- 300-450 kata total — ringkas dan padat
- TANPA sub-judul, TANPA bullet, TANPA bold, TANPA AI klise
- TANPA sitasi baru — kesimpulan tidak perlu sitasi
- Gaya formal konklusif
- Transisi alami antar paragraf tanpa penanda eksplisit

## Referensi Terkait

- [Citation Retrofit Workflow](references/citation-retrofit-workflow.md) — Workflow lengkap untuk rewrite semua section dengan closed-set sitasi setelah user komplain sitasi hilang.
- [Incremental PDF Update](references/incremental-pdf-update.md) — Cara menambahkan PDF baru ke embedding index yang sudah ada, query perubahan, dan patch draft — tanpa re-run semua pertanyaan.
- [Prompt Bab 1](references/prompt-bab-1.md) — Template prompt lengkap untuk generate Bab 1 Pendahuluan 5 paragraf naratif.
- [Prompt Bab 3](references/prompt-bab-3.md) — Template prompt lengkap untuk generate Bab 3 Kesimpulan 3 paragraf compact.
- [Combine to PDF](scripts/combine-to-pdf.py) — Script Python untuk menggabungkan semua file section markdown + Bab 1 + Bab 3 dan mengonversi ke PDF via weasyprint.

## Pitfalls

1. **Sub-agent tidak menulis file** — beberapa sub-agent hanya mengembalikan summary dengan teks embedded, bukan menulis file. Selalu cek filesystem setelah tiap batch. Tulis manual jika perlu.
2. **CSV delimiter** — file dari `research_answers_deepseek_clean.csv` menggunakan `;` delimiter dengan pipe `|` di header line. Python csv reader mungkin bingung; parse manual dengan `.split(';')`.
3. **Q&A terpotong** — beberapa jawaban PaperQA dipotong di JSON. Ambil data langsung dari CSV asli via terminal grep untuk versi lengkap.
4. **Sub-agent sitasi keluar dari sumber** — sub-agent kadang menambahkan sitasi yang tidak ada di data Q&A (hallucination). Periksa dan patch jika perlu. Atau beri peringatan di prompt: "HANYA gunakan sitasi yang ada di data Q&A, jangan buat baru."
5. **PDF dependencies** — weasyprint perlu system packages. Fallback: kirim file markdown saja jika PDF gagal.
6. **Word count** — sub-agent sering melebihi target. It's fine; lebih baik kelebihan daripada kurang.
7. **Nama file inkonsisten** — sub-agent punya kebiasaan naming sendiri. Selalu ls setelah batch untuk tahu path aktual.
8. **Batch size** — max 3 concurrent tasks via delegate_task. Loop dalam batch of 3.
9. **Sitasi tidak tersedia di data Q&A** — jika pertanyaan di outline tidak memiliki jawaban di CSV, section tersebut HARUS ditulis manual berdasarkan konteks outline dan pengetahuan umum. Jangan panggil sub-agent untuk pertanyaan tanpa data.
10. **Rewrite scenario: citation page-range mismatch.** Sub-agent sering mengubah format sitasi dari `(AuthorYear pages X-Y)` menjadi `(Author et al., tahun)` atau format bebas. Jika ini terjadi, rewrite dengan closed-set eksplisit: "GUNAKAN PERSIS format sitasi dari daftar ini". Lihat [Citation Retrofit Workflow](references/citation-retrofit-workflow.md).
11. **"Setiap paragraf diakhiri sitasi" constraint.** Beberapa sub-agent menulis paragraf tanpa sitasi sama sekali meskipun data diberikan. Solusi: Step 1b (pre-ekstraksi) + Step 5b (verifikasi) + iterasi rewrite. Jangan lanjut ke PDF sebelum verifikasi lolos.
12. **Batch rewrite citation.** Jika 12+ section perlu di-rewrite untuk sitasi, lakukan dalam batch 3 concurrent. Setelah tiap batch, verifikasi dengan `grep -c 'pages' each_file.md`. File yang masih 0 cites → rewrite ulang dengan instruksi lebih ketat.
13. **Sub-agent tidak menulis file saat rewrite.** Saat rewrite section, sub-agent kadang mengembalikan teks di summary tapi tidak menyimpan file. Cek filesystem setelah tiap batch. Jika tidak ada file baru, ambil teks dari summary dan tulis manual.

14. **Section headings MUST match outline exactly.** Jangan merge heading names atau pakai range seperti "2.4.1–2.4.2". Setiap subsection dari outline mendapat heading sendiri dengan nomor DAN judul persis dari outline. Jika outline menulis "### 2.4.1 RAAS/ARNI: Kontraindikasi & Konsekuensi", heading di draft harus sama persis — bukan "RAAS inhibitor pada kehamilan".

15. **Target audience menentukan densitas konten.** Untuk general cardiologist: fokus pada implikasi klinis, angka kunci, terapi, dan evidence gaps. Potong elaborasi molekuler, detail teknis ekokardiografi, sejarah definisi. Jika user minta "compact" atau "kurangi 50%", target Bab 2 ~3.000-4.000 kata dengan tetap mempertahankan sitasi pada setiap paragraf.

16. **DOCX output mungkin lebih diinginkan daripada PDF.** `pip install python-docx` untuk konversi. Format: Times New Roman 12pt, spasi 1.5, justify, first-line indent 1.27cm. Heading levels: # = Heading 1 (14pt), ## = Heading 2 (13pt), ### = Heading 3 (12pt).

17. **Guideline update (ESC 2018 → 2025).** Jika user memberikan PDF pedoman baru: (a) copy ke folder pdf/ — `get_docs_async()` akan append-only embed. (b) JANGAN re-run semua pertanyaan. Cukup 3-5 targeted queries: "Apa perbedaan [guideline baru] vs [guideline lama]?" (c) Patch hanya section yang berubah signifikan. Jika hanya tahun berubah, update sitasi saja. Jika ada perubahan rekomendasi (dosis, kelas, indikasi), rewrite paragraph terkait. (d) Jangan rewrite section yang tidak berubah isinya.
