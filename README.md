# Github Duplicate Bug detector

**Student:** Vasantha Perugu  
**Project type:** Duplicate bug detection with GitHub issue data, text similarity, model evaluation and a Streamlit interface

## What this project does

I built this project to check whether a new bug report looks like an issue that already exists. The main idea is to compare the new bug title and description with issues collected from GitHub before the report is saved.

The detector does not depend on an exact title match. Bug reports often describe the same problem in different ways so I compare several kinds of text evidence. The final result combines title similarity, description similarity, important words, character fragments and broader semantic meaning. I also added rules that prevent one strong but incomplete signal from being treated as a confirmed duplicate.

The notebook collects recent issues from ten public GitHub repositories, cleans the text, explores the dataset, builds the similarity features, creates candidate pairs, evaluates the detector, saves the trained artifacts and creates a Streamlit application.

## Project files

```text
Github_Duplicate_Bug_Detector_Project/
├── Github_Duplicate_Bug_Detector.ipynb
├── README.md
├── requirements.txt
├── .gitignore
├── app/
│   └── streamlit_app.py
├── data/
│   ├── raw/
│   │   └── api_pages/
│   ├── processed/
│   └── interim/
├── models/
├── outputs/
│   ├── figures/
│   └── tables/
└── docs/
    ├── Ground_Truth_Labeling_Guide.md
    └── ground_truth_duplicate_pairs_template.csv
```

The data, model, and output folders are mostly empty in the submitted folder because the notebook creates those files during a real run. This keeps the project smaller and avoids submitting a private GitHub token or pretending that generated results are fixed forever.

## Before running the notebook

I recommend using Python 3.10 or a newer compatible Python 3 version.

Install the packages from the project folder:

```bash
pip install -r requirements.txt
```

Open the notebook in Jupyter Notebook, JupyterLab, or VS Code:

```bash
jupyter notebook Github_Duplicate_Bug_Detector.ipynb
```

The notebook asks for a GitHub token before collecting data. The token is used only for API requests in the current notebook session. The code does not save it to a CSV file, model file, metadata file, or notebook variable output.

A token can also be placed in the `GITHUB_TOKEN` environment variable. In that case, the notebook uses the environment value and skips the prompt.

Here is the github token if you want to run this project. You can generate one or use this one.
 'GITHUB_TOKEN = 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'

## Repositories used

The current notebook is configured to collect issues from these ten repositories:

- `streamlit/streamlit`
- `scikit-learn/scikit-learn`
- `anthropics/claude-code`
- `huggingface/transformers`
- `pandas-dev/pandas`
- `numpy/numpy`
- `pytorch/pytorch`
- `tensorflow/tensorflow`
- `fastapi/fastapi`
- `microsoft/vscode`

The collection window covers the most recent 365 days at the time the notebook is run. The notebook checks for at least 8,000 clean issue rows and keeps no more than 10,000 rows for the main corpus. Since the data is collected live, repository totals and evaluation results can change from one run to another.

## Notebook walkthrough

### 1. Approach

The notebook starts by describing what the duplicate checker looks for when it compares issues. For each possible match, it does not depend on one score only. It checks five parts of the issue text:

1. Semantic similarity from the combined title and description
2. Description similarity
3. Title similarity
4. Word-level TF-IDF similarity
5. Character-level TF-IDF similarity

I use more than one signal because title by itself can be vague. For example, titles such as “test failed” or “application error” can appear in unrelated reports. A stronger duplicate result should have supporting evidence from the description or the combined technical text.

### 2. Data settings

This section defines the date range, repository list, row target, API page limits and cache behavior.

Live GitHub collection is enabled. If the API fails after a collection attempt and a cached raw CSV already exists, the notebook can use that cached file. The fallback is there to make the notebook easier to recover, but the normal path is still live collection.

### 3. GitHub access

The notebook checks for `GITHUB_TOKEN`. When it is not found, it asks for the token with hidden input. The authorization header exists only in memory while the notebook is running.



### 4. Collect recent GitHub issues

The notebook calls the GitHub issues API one page at a time. It saves each raw JSON page under `data/raw/api_pages/`, which gives me a record of what came back from the API.

Pull requests are removed because GitHub returns issues and pull requests through the same endpoint. The final rows contain fields such as repository, issue number, issue key, title, body, state, labels, dates, comments and URL.

The combined raw issue table is saved as:

```text
data/raw/github_issues_raw.csv
```

### 5. Clean the dataset and check its size

This step prepares the collected GitHub issues before any matching is done. Notebook fixes the date fields, removes rows without an issue key and drops repeated issue records. It then keeps only issues from the 365-day window and fills any missing title or body text with empty values. After that, it adds few helper columns such as text length and issue week, so the dataset can be checked before building the duplicate detector.

The cleaned corpus is saved as:

```text
data/processed/github_issues_clean.csv
```

The notebook stops with a clear message when the clean dataset has fewer than 8,000 rows. I used this check so that the detector is not quietly built from a much smaller dataset than the project requires.

### 6. Visual review of the dataset

This section gives a quick picture of the collected corpus. It creates:

- issue count by repository
- open and closed issue count by repository
- weekly issue volume

The figures are saved under `outputs/figures/`.

These charts help me confirm that the data is not coming from only one repository and that the collection window contains a reasonable spread of issues.

### 7. Prepare title and bug-description text

The cleaning function removes code blocks, web links, HTML tags, repeated spaces and most punctuation while keeping technical characters that may be useful in bug reports.

The notebook creates separate cleaned fields for the title and description, plus a combined search field. Keeping title and description separate matters because the app reports both percentages and does not make a decision from title similarity alone.

This section also identifies bug-like language and duplicate-related language. The EDA output includes:

- a chart for bug and duplicate text signals
- a word cloud for common terms in bug-like issues
- a table of the top twenty terms
- a horizontal frequency graph for those terms

The word cloud is descriptive only. It helps show what the corpus talks about, but it is not used as an input to the duplicate decision.

### 8. Build text and semantic features

The notebook creates four TF-IDF feature sets:

- combined title and description word features
- combined character-fragment features
- title-only word features
- description-only word features

TTF-IDF gives more value to terms that are useful in a specific issue and less value to words that appear almost everywhere. The character features use short pieces of words, which can help with spelling differences, version strings, exception names, file paths, and error fragments.

The semantic feature uses Latent Semantic Analysis by default. LSA reduces the large TF-IDF matrix into a smaller set of patterns. In simple terms, it helps the detector notice that two reports may discuss a similar problem even when they do not use exactly the same words.

The notebook includes an optional sentence-transformer path, but it is disabled by default. The LSA path keeps the project reproducible with scikit-learn and avoids requiring a large external model download.
### 9. Find similar bug reports

The detector does not compare every possible pair in the full corpus because that would be slow and wasteful. It uses nearest-neighbor searches inside each repository to create a smaller candidate list.

Each candidate receives the five similarity values and two combined scores:

- `final_score`, which is the weighted hybrid similarity
- `duplicate_score`, which applies the evidence rules and coherence checks

The main weights are:

| Signal | Weight |
|---|---:|
| Semantic similarity | 0.30 |
| Description similarity | 0.30 |
| Title similarity | 0.12 |
| Combined word TF-IDF | 0.20 |
| Character TF-IDF | 0.08 |

The title weight is intentionally lower than the description and semantic weights. I did this because short titles can look similar even when the actual bugs are different.

The word contribution is capped at 40%. 

### 10. Manual review file

The notebook creates a review sheet with the strongest candidate pairs:

```text
data/interim/manual_duplicate_review_labels.csv
```

This file is meant for checking the model output by hand. Each row shows the two issue keys, short text previews, similarity score and the evidence found by the notebook. The last columns are left blank so a reviewer can add the final label, reviewer name and notes. This keeps the review work separate from the code that generates the candidate pairs.

### 11. Ground-truth evaluation

The evaluation uses:

```text
data/processed/ground_truth_duplicate_pairs.csv
```

The required columns are:

- `source_issue_key`
- `target_issue_key`
- `label`

An optional `label_source` column can record where a label came from.

When a ground-truth file is not present, the notebook looks for explicit duplicate references in the collected GitHub issues. It creates positive pairs from those references and adds low-overlap control pairs as non-duplicates. This is a practical way to create a starting evaluation set, although manually reviewed labels are still better for a final project.

The labeled pairs are split into validation and test sets. The validation set is used to choose the likely-duplicate threshold. Precision, recall, F1 score, and the confusion matrix are then reported on the held-out test set.

This separation is important because choosing a threshold and reporting the final score on the same examples would make the performance look better than it really is.

The section saves threshold results, scored validation pairs, scored test pairs, and an evaluation summary under outputs/.

### 12. Duplicate Check Logic

The duplicate checker takes a new bug title, description and compares them with the issues collected from the same GitHub repository. Before using the main similarity score, it checks for stronger evidence such as an exactly repeated description or a large copied part of an older issue description.

The title and description are also checked separately. This matters because a short title can sometimes match one issue while the longer description matches a different issue. In that case, the app does not combine those two weak signals into one confident duplicate result. This helps reduce misleading duplicate warnings.

The decision messages are kept simple:

- duplicate of this bug
- possible duplicate of these bugs
- no matching duplicate found in the collected corpus
- 
The detector is meant to help a reviewer. It does not automatically reject new issue.


### 13. Save artifacts for the app

The notebook saves the fitted vectorizers, feature matrices, cleaned corpus, weights, thresholds, evaluation summary, semantic model information and decision settings in:

```text
models/duplicate_bug_detector_artifacts.joblib
```

Saving the full set of artifacts keeps the Streamlit app consistent with the notebook. The app does not retrain a different model or use a separate scoring method.

### 14. Build the Streamlit interface

The notebook writes the application code to:

```text
app/streamlit_app.py
```

The app title is **Github Duplicate Bug detector**. A user selects a repository, enters a bug title and description, chooses the number of candidates and clicks **Check before save**.

The app displays the closest existing issues, title and description match percentages, issue links and optional technical scores. It also shows the held-out evaluation summary in the sidebar.

### 15. Open the Streamlit app

The notebook looks for a free port from 8502 through 8525. It starts Streamlit in headless mode and opens one browser tab. Re-running the cell in the same notebook session reuses the current process instead of launching several copies.

The app can also be started from a terminal after the notebook has saved the model artifacts:

```bash
streamlit run app/streamlit_app.py
```

### 16. Final submission notes

The last section summarizes the project settings and reminds the reader that the notebook uses live GitHub data, a minimum row target, word-cloud EDA, held-out test evaluation, and the same saved artifacts in the app.

## How the duplicate decision works

The weighted score is only the starting point. The detector also checks whether the evidence for one candidate makes sense together.

A candidate can receive stronger duplicate evidence when, for example:

- both its title and description are similar
- the description is strong and the semantic or word evidence supports it
- the combined technical text is strong and the description is not weak
- semantic similarity is high and the description supports the same candidate

A title-only match or a semantic-only match is normally kept in the possible-review range. Generic similarity is capped so that reports about the same broad topic do not automatically become duplicates.

The user-facing field rule is 40%. A candidate is displayed when its description reaches the threshold or when an informative title reaches the threshold. Generic one-word titles do not create a title-only duplicate warning.

## Files created after a complete run

A successful run creates files similar to these:

```text
data/raw/github_issues_raw.csv
data/raw/api_pages/*.json
data/processed/github_issues_clean.csv
data/processed/ground_truth_duplicate_pairs.csv
data/interim/manual_duplicate_review_labels.csv
models/duplicate_bug_detector_artifacts.joblib
outputs/collection_metadata.json
outputs/evaluation_summary.json
outputs/figures/*.png
outputs/tables/*.csv
app/streamlit_app.py
```

## Ground-truth labels

A template is included at:

```text
docs/ground_truth_duplicate_pairs_template.csv
```

Replace the example issue keys with real keys from the cleaned corpus. Then save the completed file as:

```text
data/processed/ground_truth_duplicate_pairs.csv
```

Use `true_duplicate` and `not_duplicate` as labels. The notebook also accepts a few simple equivalents, but keeping one consistent label style makes the file easier to review.

More details are in `docs/Ground_Truth_Labeling_Guide.md`.

## Important limits

This project is a similarity-based review tool, not a final authority on whether two issues are duplicates.

Some limitations are:

- live GitHub data changes over time
- issue descriptions can be missing or very short
- an explicit duplicate reference is useful but is not the same as a fully reviewed human label
- repository writing styles are different
- technical issues can use the same words while having different causes
- LSA captures broad text patterns but does not understand software behavior the way an engineer does

The results should be used to help with triage and review, not to close issues automatically.

## Reproducing the project

For the closest reproduction of the submitted workflow:

1. Install the packages.
2. Open the notebook from the project root.
3. Run the cells from top to bottom.
4. Enter a GitHub token when prompted.
5. Allow the notebook to collect and clean at least 8,000 issues.
6. Review the EDA figures and word cloud.
7. Check or improve the ground-truth pair file.
8. Run the evaluation section.
9. Save the artifacts.
10. Run the Streamlit section or start the app from the terminal.
    
Because the project uses a moving 365-day window, exact counts and metrics are expected to change. The workflow, feature construction, validation/test split and scoring rules remain reproducible.
