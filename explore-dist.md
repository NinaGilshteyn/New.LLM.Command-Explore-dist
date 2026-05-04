Interactively explore the distribution of a dataset column using a swarmplot and histogram side by side. Iterate on bin count until the user is satisfied, then save the final figure to disk.

**Arguments** (`$ARGUMENTS`): `<dataset_path> [column_name]`
- `dataset_path` — path to a CSV, TSV, XLSX, or XLS file
- `column_name` (optional) — column to visualize; defaults to the first numeric column

---

## Step 1 — Write the Python helper script

Write the following content verbatim to `C:\Windows\Temp\explore_dist.py` (overwrite if it exists):

```python
import sys, os, math, tempfile, subprocess
import numpy as np
import pandas as pd
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.image as mpimg
import seaborn as sns

def load_df(path):
    ext = os.path.splitext(path)[1].lower()
    if ext in ('.xlsx', '.xls'):
        return pd.read_excel(path)
    if ext in ('.tsv', '.txt'):
        return pd.read_csv(path, sep='\t')
    return pd.read_csv(path)

def open_image(path):
    if sys.platform == 'win32':
        os.startfile(path)
    elif sys.platform == 'darwin':
        subprocess.run(['open', path])
    else:
        subprocess.run(['xdg-open', path])

def main():
    dataset  = sys.argv[1]
    column   = sys.argv[2] if len(sys.argv) > 2 and sys.argv[2] else None
    bins     = int(sys.argv[3]) if len(sys.argv) > 3 and sys.argv[3] else None
    switch   = int(sys.argv[4]) if len(sys.argv) > 4 and sys.argv[4] else 0
    out_dir  = sys.argv[5] if len(sys.argv) > 5 and sys.argv[5] else None

    df = load_df(dataset)

    if not column:
        numeric_cols = df.select_dtypes(include=[np.number]).columns.tolist()
        if not numeric_cols:
            print("ERROR: no numeric columns found")
            sys.exit(1)
        column = numeric_cols[0]

    data = df[column].dropna().reset_index(drop=True)
    N    = len(data)

    if bins is None:
        bins = max(1, int(round(math.sqrt(N))))

    sqrt_n = math.sqrt(N)
    tmpdir = tempfile.mkdtemp(prefix='explore_dist_')
    swarm_f    = os.path.join(tmpdir, 'swarm.png')
    dist_f     = os.path.join(tmpdir, 'dist.png')
    combined_f = os.path.join(tmpdir, 'combined.png')

    q25, median, q75 = np.percentile(data, [25, 50, 75])

    # --- Swarmplot (subsample if large to keep render fast) ---
    plot_data = data if N <= 400 else data.sample(400, random_state=0)
    fig1, ax1 = plt.subplots(figsize=(7, 4))
    sns.swarmplot(x=plot_data, ax=ax1, size=3)
    ax1.axvline(q25,    color='steelblue', linestyle='--', linewidth=1.2, label=f'Q25 ({q25:.2f})')
    ax1.axvline(median, color='crimson',   linestyle='-',  linewidth=1.5, label=f'Median ({median:.2f})')
    ax1.axvline(q75,    color='steelblue', linestyle='--', linewidth=1.2, label=f'Q75 ({q75:.2f})')
    ax1.legend(fontsize=8)
    ax1.set_xlabel(column)
    label = f'N={N}' if N <= 400 else f'sample 400/{N}'
    ax1.set_title(f'Swarmplot: {column}  [{label}]')
    fig1.savefig(swarm_f, dpi=120, bbox_inches='tight')
    plt.close(fig1)

    # --- Displot ---
    g = sns.displot(data, bins=bins, kde=True, height=5, aspect=1.2)
    g.set_axis_labels(column, 'Count')
    g.figure.suptitle(
        f'Distribution: {column}  |  bins={bins}  |  N={N}  |  √N≈{sqrt_n:.1f}',
        y=1.03, fontsize=10
    )
    ax = g.ax
    ax.axvline(q25,    color='steelblue', linestyle='--', linewidth=1.2, label=f'Q25 ({q25:.2f})')
    ax.axvline(median, color='crimson',   linestyle='-',  linewidth=1.5, label=f'Median ({median:.2f})')
    ax.axvline(q75,    color='steelblue', linestyle='--', linewidth=1.2, label=f'Q75 ({q75:.2f})')
    ax.legend(fontsize=8)
    g.savefig(dist_f, dpi=120, bbox_inches='tight')
    plt.close('all')

    # --- Combine side by side ---
    i1 = mpimg.imread(swarm_f)
    i2 = mpimg.imread(dist_f)
    fig_c, (al, ar) = plt.subplots(1, 2, figsize=(14, 7))
    al.imshow(i1); al.axis('off')
    ar.imshow(i2); ar.axis('off')
    plt.tight_layout(pad=1)

    # Metadata printed for Claude to parse
    print(f"BINS={bins}")
    print(f"N={N}")
    print(f"SQRT_N={sqrt_n:.2f}")
    print(f"COLUMN={column}")

    if switch == 0:
        fig_c.savefig(combined_f, dpi=120, bbox_inches='tight')
        plt.close('all')
        open_image(combined_f)
        print(f"TMPFILE={combined_f}")
    else:
        if not out_dir:
            out_dir = os.path.dirname(os.path.abspath(dataset))
        safe     = "".join(c if c.isalnum() or c in '-_' else '_' for c in column)
        out_name = f'distribution_{safe}_bins{bins}.png'
        out_path = os.path.join(out_dir, out_name)
        fig_c.savefig(out_path, dpi=150, bbox_inches='tight')
        plt.close('all')
        print(f"SAVED={out_path}")
        print(f"FOLDER={out_dir}")

if __name__ == '__main__':
    main()
```

---

## Step 2 — Parse arguments

From `$ARGUMENTS`, extract:
- `DATASET` = the dataset path (first token)
- `COLUMN`  = the column name (second token, or empty string if not provided)

---

## Step 3 — Exploration loop (repeat until user approves)

Initialize `BINS` as empty (the script will compute √N on the first run).

**Each iteration:**

1. Run the script with switch=0:
   ```
   python C:\Windows\Temp\explore_dist.py "DATASET" "COLUMN" "BINS" 0
   ```
   Pass an empty string `""` for COLUMN or BINS when not yet set.

2. Parse stdout for the key=value lines:
   - `BINS=`, `N=`, `SQRT_N=`, `COLUMN=`

3. Tell the user (replace placeholders with parsed values):
   > Plots are open. Column: **COLUMN** | N = **N** | Current bins: **BINS** (√N ≈ **SQRT_N**)
   >
   > Keep this binning, or enter a different bin count?

4. Wait for the user's response:
   - **"keep" / "yes" / "ok" / "looks good"** (or any affirmation) → exit the loop, go to Step 4
   - **A number** → set `BINS` to that number, go back to iteration step 1
   - **A different column name** → set `COLUMN` to that name, reset `BINS` to `""`, go back to iteration step 1

---

## Step 4 — Ask for save location

Ask the user:
> Where would you like to save the figure? (Press Enter to use the dataset's folder: `<dataset_dir>`)

- If the user provides a path → use it as `OUT_DIR`
- If the user presses Enter / says nothing / says "here" or "same" → set `OUT_DIR` to the dataset's directory

## Step 5 — Save to disk (switch=1)

Run the script with the final approved BINS and OUT_DIR:
```
python C:\Windows\Temp\explore_dist.py "DATASET" "COLUMN" "BINS" 1 "OUT_DIR"
```

Parse `FOLDER=` from stdout and report to the user:
> Figure saved to: **FOLDER**
