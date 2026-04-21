# ECB Interview Scraper

A Python workflow that collects every English-language interview published on the European Central Bank's press site and turns it into a research-ready dataset. The pipeline discovers interview URLs from the official archive, downloads each page, and extracts structured fields: date, speaker, media outlet, title, cleaned full text, and separated question–answer pairs.

The output is one row per interview (plus a long-format companion file at the Q&A level) — suitable for topic modelling, sentiment analysis, speaker-stratified panels, and anything else downstream.

## What's in the dataset

A single run produces four files:

| File | Description |
|---|---|
| `ecb_interviews.csv` | One row per interview. CSV with aggressive quoting, safe for any text content. |
| `ecb_interviews.parquet` | Same content, typed and lossless. Preferred for reloading in Python/R. |
| `ecb_interviews_qa.csv` | Long format — one row per Q&A pair, with interview-level metadata repeated. |
| `ecb_interviews_errors.csv` | Only written if any URL failed. URL + error message. |

Columns in the main file ("ecb_interviews.csv"):

```
url              ECB press page URL (acts as unique ID)
date             Parsed from URL slug, ISO format (YYYY-MM-DD)
speaker          Canonical speaker name (see normalisation below)
media_outlet     Parsed from the page title
title            Raw <h1> text
full_text        Cleaned paragraph text (questions + answers)
answers_only     Answers concatenated, preamble excluded
n_qa_pairs       Count of extracted Q&A pairs
qa_pairs_json    List of {question, answer} dicts (JSON-encoded in CSV)
word_count       Whitespace-split token count of full_text
```

## Current corpus

Numbers from the latest run against the live archive:

- 590 interviews
- 8,771 Q&A pairs
- 31 unique speakers (Executive Board members, past and present)
- 234 unique media outlets
- Earliest parseable interview: 2004
- Zero URL-level errors on the most recent run

## How it works

The notebook is organised into eleven labelled sections. The order matches the pipeline.

**1. URL discovery (`discover_interview_urls`)**
**2. Date parsing (`parse_date_from_url`)**
**3. Media outlet (`parse_media_outlet`)**
**4. Text cleaning (`clean_text`)**
**5. Q&A extraction (`extract_qa_pairs`)**
**6. Speaker extraction (`extract_speaker` + `CANONICAL_SPEAKERS`)**
**7. Batch extraction `extract_interview`**

## Requirements

```
python >= 3.9
requests
beautifulsoup4
pandas
pyarrow          # for Parquet export
selenium >= 4
```

You also need Chrome (or Chromium) installed and a matching `chromedriver` on your `PATH`. Selenium 4 will manage the driver automatically in most setups; if yours doesn't, install `chromedriver` manually or use `webdriver-manager`.

Install with:

```bash
pip install requests beautifulsoup4 pandas pyarrow selenium
```

## Usage

Open `ECBInterviewsScraper.ipynb` in Jupyter and run top to bottom. The notebook is linear — each section depends only on what precedes it. A full run against the live archive takes roughly 15–20 minutes, most of which is the `time.sleep(1)` between requests.

If you want to re-run only the extraction step (for example, to retry after editing a helper), you can skip the Selenium cell and load `interview_urls` from a previous run.

### Politeness

The scraper sends a descriptive `User-Agent` (`Mozilla/5.0 (research-scraper; academic use)`) and sleeps one second between requests. Please keep both. The ECB site is public but not unlimited; being a good citizen matters if we want this to stay usable.

## Known limitations

- **English only.** The discovery step filters for `.en.html` URLs. Non-English versions of the same interview are not collected.
- **Interview scope.** The listing page is filtered to `name_of_publication=Interview`. Speeches, press conferences, blog posts, and podcasts are out of scope by design.
- **Q&A heuristic is paragraph-level.** Interviews that don't use the paragraph-break convention (rare, but they exist) produce a single large preamble. These are detectable via `n_qa_pairs == 0` and can be hand-reviewed.
- **`Interview of X with Y` titles** leave `media_outlet` empty on purpose; those need manual review if outlet-level analysis matters.
- **Name dictionary needs maintenance.** New Executive Board appointments require a single-line update to `CANONICAL_SPEAKERS`. Without it, new speakers fall through to the empty string or the `Mr/Mrs` fallback.
- **Site template drift.** Noise-removal patterns were compiled against the current template. If the ECB redesigns the press section, expect to inspect output and extend `NOISE_PATTERNS`.

## Repository layout

```
ECB-Interviews-Dataset/
│
├── ECBInterviewsScraper.ipynb           # main scraper pipeline
├── README.md                # this file
├── requirements.txt         # Python dependencies
├── .gitignore
│
├── ecb_interviews.csv       # interview-level dataset (one row per interview)
├── ecb_interviews.parquet   # typed copy of the above
└── ecb_interviews_qa.csv    # long-format Q&A (one row per question)
```

## Citation

The transcript text is the ECB's and their reproduction notice asks that the source be acknowledged. If you use the dataset in published work, please credit:

> the European Central Bank for the underlying transcripts, and this project for the scraper and parsed dataset:
> Hussein Khalid Hussein (2026). ECB Interview Dataset.
> https://github.com/HusseinKh90/ECB-Interviews-Dataset

## License

Code is released under the MIT License. The interview content itself belongs to the European Central Bank and the respective media outlets; redistribution of the scraped text should respect their terms.

## Disclaimer

This project is not affiliated with or endorsed by the European Central Bank. It scrapes publicly available material for research purposes.
