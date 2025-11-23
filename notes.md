# Lab 11: Data & Model Versioning with DVC - Notes

## Key Concepts

### Git vs DVC
- **Git** is designed for code (small text files)
- **DVC** is designed for large data files and models
- Can't track the same file with both Git and DVC at the same time
- The `.dvc` file (which is small) gets tracked by Git, while the actual data file is tracked by DVC
- Think of `.dvc` files as "pointers" or "receipts" that tell DVC which version of data to retrieve

### Version Switching (Two-Step Dance)
Git and DVC work together:

**Step 1: `git checkout HEAD~1 data/raw/data.csv.dvc`**
- `HEAD~1` means "go back 1 commit from the current commit"
- This checks out the older version of the `.dvc` file (the metadata file that Git tracks)
- Git updated the `.dvc` file to point to the original, smaller dataset
- But the actual `data.csv` file hasn't changed yet!

**Step 2: `dvc checkout`**
- DVC reads the `.dvc` file and sees it points to the old version of the data
- DVC fetches the old data from the cache and replaces the current `data.csv`
- `M data\raw\data.csv` means "Modified" - the actual data file was updated
- Now you have the original, smaller dataset in your workspace

---

## Lab Summary

### Part 1: Basic DVC Workflow
**What we did:**
1. Initialized DVC in the repository
2. Configured local remote storage at `/tmp/dvc-storage`
3. Tracked `data/raw/data.csv` with DVC (had to remove from Git first)
4. Augmented the dataset using `scripts/augment_data.py`
5. Tracked the new version
6. Demonstrated switching between versions using Git + DVC commands
7. Trained and tracked a model with DVC

**Key Takeaway:** DVC extends Git to handle large files efficiently using content-addressable storage

---

### Part 2: DVC Pipelines
**What we did:**
1. Created `dvc.yaml` with three stages:
   - **preprocess**: `data/raw/data.csv` → `data/processed/train.csv` + `test.csv`
   - **train**: `train.csv` + `params.yaml` → `models/classifier.pkl`
   - **evaluate**: `classifier.pkl` + `test.csv` → `metrics/scores.json`
2. Updated scripts to load from `params.yaml` and use processed data
3. Ran the pipeline with `dvc repro`
4. Visualized dependencies with `dvc dag`
5. Changed hyperparameters and observed smart caching

**Key Takeaway:** DVC only re-runs stages when their dependencies change
- Changed `n_estimators` from 100 → 150
- **preprocess** was skipped (data didn't change)
- **train** ran (hyperparameter changed)
- **evaluate** ran (model changed)

---

### Part 3: Experiment Tracking
**What we did:**
1. Ran 3+ experiments with different hyperparameters:
   - Experiment 1: `n_estimators=200, max_depth=5` → accuracy: 0.956
   - Experiment 2: `n_estimators=200, max_depth=10` → accuracy: 0.956
   - Experiment 3: `n_estimators=50, max_depth=3` → accuracy: 0.947
2. Compared results using `dvc exp show`
3. Best model: `n_estimators=150, max_depth=5` with **96.55% accuracy**

**Key Takeaway:** DVC experiments allow running multiple trials without cluttering your workspace

---

## Quick Reference Commands

### Basic DVC Commands
```bash
# Initialize DVC
dvc init

# Configure remote storage
dvc remote add -d myremote /tmp/dvc-storage

# Track a file with DVC
dvc add <file>

# Push/pull data to/from remote
dvc push
dvc pull

# Checkout data for current .dvc files
dvc checkout
```

### Pipeline Commands
```bash
# Run the entire pipeline
dvc repro

# Visualize the pipeline
dvc dag

# Check pipeline status
dvc status
```

### Experiment Commands
```bash
# Run experiment with modified parameters
dvc exp run --set-param train.n_estimators=200

# Show all experiments
dvc exp show

# Apply an experiment to workspace
dvc exp apply <experiment-name>

# Push experiments to remote
dvc exp push origin
```

### Git + DVC Combined
```bash
# Remove file from Git tracking (before adding to DVC)
git rm -r --cached <file>

# Switch to previous version
git checkout HEAD~1 <file>.dvc
dvc checkout

# Return to latest version
git checkout HEAD <file>.dvc
dvc checkout
```

---

## Answers to TA Questions

### Deliverable 1: Why Git alone is insufficient for ML projects and how DVC solves this

**Why Git is insufficient:**
1. **Storage inefficiency**: Git stores full file history. For a 1GB dataset with 10 versions, you'd have 10GB in the repository
2. **Performance issues**: Git slows down significantly with large binary files (models, datasets)
3. **Binary file handling**: Git diff doesn't work well with binary files - you can't see meaningful changes
4. **Collaboration bottlenecks**: Large repos are slow to clone and push/pull
5. **Repository bloat**: Over time, the `.git` folder becomes enormous and unmanageable

**How DVC solves this:**
1. **Content-addressable storage**: Stores files by their hash, deduplicates automatically
2. **Pointer files**: Git only tracks small `.dvc` metadata files (~100 bytes), not the actual data
3. **Flexible remotes**: Can use local storage, S3, GCS, Azure, SSH, etc.
4. **Efficient caching**: Only downloads what you need, when you need it
5. **Version control**: Full versioning capability like Git, but optimized for large files
6. **Reproducibility**: Links code versions (Git commits) with data versions (DVC cache)

---

### Deliverable 2: How DVC determines which stages need to re-run when you change hyperparameters

**The Mechanism:**
1. **Dependency tracking**: `dvc.yaml` defines dependencies for each stage:
   - Input files (deps)
   - Parameter values (params)
   - Code/scripts (deps)

2. **Lock file (`dvc.lock`)**: Records exact state of all dependencies:
   - File hashes (MD5)
   - Parameter values
   - Commands executed

3. **Change detection**: When you run `dvc repro`:
   - DVC compares current state vs `dvc.lock`
   - If any dependency changed → re-run that stage + all downstream stages
   - If nothing changed → skip (use cached output)

**Example from our lab:**
```yaml
train:
  params:
    - train.n_estimators  # DVC watches this parameter
```

When we changed `n_estimators: 100` → `150`:
- **preprocess stage**: Skipped (no dependencies changed)
  - Still uses cached `train.csv` and `test.csv`
- **train stage**: Re-ran (parameter dependency changed)
  - Trains new model with 150 estimators
- **evaluate stage**: Re-ran (model dependency changed)
  - New model needs new evaluation

**Why this matters:**
- Saves time: Don't reprocess data that hasn't changed
- Ensures correctness: Always re-runs when needed
- Reproducibility: `dvc.lock` records exact experiment state

---

### Deliverable 3: DVC vs W&B for experiment tracking, and how they complement each other

**When to use DVC:**
- **Data versioning**: Track datasets, preprocessed data, feature stores
- **Model versioning**: Version trained models alongside code
- **Pipeline orchestration**: Define multi-stage ML workflows
- **Reproducibility**: Guarantee exact recreation of experiments
- **Local-first workflow**: Work offline, push when ready
- **Storage flexibility**: Use your own infrastructure (S3, local, NAS)
- **Large artifacts**: Handle multi-GB models and datasets efficiently

**When to use W&B (Weights & Biases):**
- **Real-time monitoring**: Watch training metrics live
- **Rich visualizations**: Interactive charts, confusion matrices, ROC curves
- **Hyperparameter tuning**: Sweep across parameter spaces automatically
- **Team collaboration**: Share results, compare runs across team
- **Media logging**: Track images, audio, text samples during training
- **Model registry**: Centralized model management with staging/production tags
- **Cloud-native**: Access experiments from anywhere

**How they complement each other:**

| Aspect | DVC | W&B | Together |
|--------|-----|-----|----------|
| **Data** | Version datasets | - | DVC versions, W&B uses |
| **Training** | Run pipeline | Log metrics/visualizations | DVC ensures reproducibility, W&B monitors |
| **Models** | Version artifacts | Registry + comparison | DVC stores, W&B catalogs |
| **Experiments** | Lightweight tracking | Rich analysis | DVC for data lineage, W&B for insights |
| **Collaboration** | Git-based | Cloud dashboard | DVC for artifacts, W&B for results |

**Practical workflow:**
1. Use DVC to version your dataset and define preprocessing pipeline
2. Run training with W&B logging for real-time monitoring
3. W&B tracks metrics, losses, validation curves
4. DVC versions the final trained model
5. Team reviews results in W&B dashboard
6. Best model is versioned with DVC and deployed
7. Full reproducibility: `git checkout` + `dvc checkout` recreates exact experiment

---

### Deliverable 3 (continued): How DVC could be used for the team project

**Team Project Context:** ML system with incoming data streams

**DVC Use Cases:**

**1. Data Stream Versioning**
- **Problem**: Data evolves over time (new batches, schema changes)
- **Solution**: Version each data batch with DVC
  ```bash
  # When new data arrives
  dvc add data/streams/batch_2024_11_21.csv
  git add data/streams/batch_2024_11_21.csv.dvc
  git commit -m "Add data batch from 2024-11-21"
  ```
- **Benefit**: Trace which model was trained on which data batch

**2. Feature Store Management**
- **Problem**: Feature engineering creates large intermediate datasets
- **Solution**: Track processed features separately
  ```yaml
  stages:
    feature_engineering:
      cmd: python scripts/create_features.py
      deps:
        - data/raw/stream.csv
      outs:
        - data/features/engineered.parquet
  ```
- **Benefit**: Reuse expensive feature computations across experiments

**3. Model Registry**
- **Problem**: Multiple team members training different model versions
- **Solution**: Each model version tracked with its training data
  ```bash
  dvc add models/classifier_v2.pkl
  git tag v2.0 -m "Model with accuracy 95%"
  ```
- **Benefit**: Easy rollback to previous model versions in production

**4. Reproducible Pipeline**
- **Problem**: "It worked on my machine" - hard to reproduce results
- **Solution**: DVC pipeline ensures same inputs → same outputs
  ```yaml
  stages:
    ingest:
      cmd: python ingest_stream.py
    preprocess:
      cmd: python preprocess.py
    train:
      cmd: python train.py
    evaluate:
      cmd: python evaluate.py
  ```
- **Benefit**: Any team member can reproduce exact experiment

**5. Design Considerations:**

**Data Storage Strategy:**
- Use S3/Azure Blob for remote DVC storage (team accessible)
- Separate remotes for raw data vs processed data
- Set up `.dvcignore` for temporary files

**Pipeline Design:**
- **Streaming data**: Add timestamp-based stages
  ```yaml
  ingest:
    cmd: python ingest.py --date ${date}
    outs:
      - data/raw/${date}.csv
  ```
- **Incremental training**: Track base model + fine-tuned versions
- **Data validation**: Add data quality checks as pipeline stage

**Collaboration:**
- **Shared remote**: All team members push/pull from same DVC remote
- **Branch strategy**: Feature branches for experiments, main for production models
- **Lock files**: Commit `dvc.lock` to ensure reproducibility

**CI/CD Integration:**
- Automated pipeline runs on new data
- DVC in Docker containers for deployment
- Model validation gates before promotion

**Example Team Workflow:**
```bash
# Data engineer ingests new batch
python scripts/ingest.py
dvc add data/raw/new_batch.csv
git commit -m "Add new data batch"
dvc push

# ML engineer pulls latest data
git pull
dvc pull

# Runs experiment
dvc exp run --set-param model.learning_rate=0.01

# Compares with baseline
dvc exp show

# Applies best model
dvc exp apply exp-abc123
git commit -m "Update to best model"
dvc push
```

**Key Benefits for Team Project:**
1. **Data lineage**: Know exactly which data produced which model
2. **Experimentation**: Try different approaches without breaking main branch
3. **Rollback**: Instantly revert to previous data/model version
4. **Collaboration**: Share artifacts without bloating Git repo
5. **Automation**: CI/CD can run DVC pipelines automatically

---

## Additional Notes

### Troubleshooting
- If cache is empty, use `dvc pull`
- If pipeline won't run, check `dvc.yaml` syntax and file paths
- If experiments show `!` for metrics, ensure `metrics/scores.json` is tracked in Git
- Create `data/processed/` directory if it doesn't exist

### Best Practices
1. Always commit `.dvc` files to Git
2. Never commit large files to Git (use DVC instead)
3. Use meaningful commit messages that describe data/model changes
4. Regularly push to DVC remote to avoid data loss
5. Keep `dvc.lock` in Git for reproducibility