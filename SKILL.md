---
name: charmm-gui-pdbreader
description: "Automate CHARMM-GUI PDB Reader & Manipulator via Chrome DevTools MCP browser automation. Supports: upload PDB (local or RCSB), select chains/models, terminal patches, model missing residues, preserve hydrogens, mutations, protonation states, disulfide bonds, phosphorylation (SER/THR/TYR/ARG), Lys/Arg PTMs (acetylation/methylation/succinylation/citrullination), ubiquitylation/SUMOylation, GPI anchor, glycosylation, heme coordination, lipidation (lipid-tail), peptide stapling, FRET/LRET fluorophore labels, LBT loops, MTS spin labels, MTS chemical modifiers, non-standard amino acid/RNA substitution, and generate PSF/CRD/PDB output with topology files. USE FOR: CHARMM-GUI PDB Reader, protein structure preparation, PTMs, phosphorylation, protonation, mutations, disulfide bonds, terminal patches. DO NOT USE FOR: running MD simulations, analyzing trajectories, Force Field Converter (separate skill), membrane/solution builder."
---

# CHARMM-GUI PDB Reader & Manipulator

> **AUTHORITATIVE GUIDANCE — MANDATORY COMPLIANCE**
>
> This skill automates the **PDB Reader & Manipulator** module of CHARMM-GUI (https://www.charmm-gui.org/?doc=input/pdbreader) using Chrome DevTools MCP browser automation.

## Prerequisites

1. **Chrome must be running** with remote debugging enabled.
2. **User must be logged in** to CHARMM-GUI. Verify by checking for a "Logout" button at https://www.charmm-gui.org/. If not logged in, ask the user to log in manually.
3. **MCP Chrome DevTools tools** must be available.

---

## Rules

> **TOOL CALL BUDGET**
>
> Target: Complete PDB Reader workflow in **under 10 tool calls**.

1. **Take one snapshot per page load** to confirm the page loaded. Do not take redundant snapshots.
2. **If the user provided a complete spec**, do not ask for confirmation — apply, validate, proceed.
3. **Use direct function calls** for phosphorylation, PTMs, protonation, and disulfide bonds. Do NOT manipulate dropdowns for these — pass values as function arguments.
4. **Checkboxes MUST be enabled** for each modification section. Calling `add_phos()` etc. creates DOM rows, but the server ignores them if the section checkbox is unchecked. Always enable the checkbox BEFORE calling the add function.
5. **Enable a checkbox like this:**
   ```js
   const cb = document.getElementById('phos_checked');
   if (cb && !cb.checked) cb.click();
   ```
6. **Click "Next Step"** via:
   ```js
   document.querySelectorAll('img').forEach(img => { if (img.src?.includes('next')) img.click(); });
   ```
7. **Click "download.tgz"** via:
   ```js
   document.querySelectorAll('a').forEach(a => { if (a.textContent.trim() === 'download.tgz') a.click(); });
   ```

---

## Direct Function Call API

> **Verified from CHARMM-GUI source code.** These global JavaScript functions handle DOM manipulation internally.

> **CRITICAL — HOW ADD FUNCTIONS ACTUALLY WORK:**
>
> Each `add_*()` function does TWO things:
> 1. **COMMITS** the current active row (whatever values are currently showing in the `_0` dropdowns)
> 2. **CREATES** a new active row populated with the passed parameters
>
> When a section checkbox is first enabled, CHARMM-GUI creates a **default active row** with preset values (e.g., TYR/41/TP1 for phosphorylation, LYS/4/KSUC for PTMs). This default row gets committed by the first `add_*()` call, creating an unwanted entry.
>
> **The pattern for N entries:**
> 1. Enable the section checkbox (default row appears)
> 2. Call `add_*()` N times with parameters for all N entries
> 3. This produces N correct committed rows + 1 unwanted default committed row + the Nth entry as the active row
> 4. **Delete the first committed row** (the default) using `remove_row()`
>
> Alternatively for N entries: call `add_*()` (N-1) times for entries 2 through N. Entry 1 must be set in the default row manually. But since the onchange handlers are async, the direct-call-then-delete approach is more reliable.

### add_phos(chain, residue, resid, patch)

Commits current active row, then creates a new row with the given parameters.

| Parameter | Type | Description |
|---|---|---|
| chain | string | Segment ID, e.g., `'PROA'` |
| residue | string | `'SER'`, `'THR'`, `'TYR'`, or `'ARG'` |
| resid | string | Residue number, e.g., `'10'` |
| patch | string | Phosphorylation type (see Verified Patch Values) |

Example: `add_phos('PROA', 'SER', '10', 'SP2');`

### add_ptm(chain, residue, resid, patch)

Commits current active row, then creates a new row with the given parameters.

| Parameter | Type | Description |
|---|---|---|
| chain | string | Segment ID |
| residue | string | `'LYS'` or `'ARG'` |
| resid | string | Residue number |
| patch | string | Modification type (see Verified Patch Values) |

Example: `add_ptm('PROA', 'LYS', '9', 'KAC');`

### add_prot(chain, residue, resid)

Commits current active row, then creates a new row. **3 parameters only — no patch.** The `residue` parameter is the target protonation state.

| Parameter | Type | Description |
|---|---|---|
| chain | string | Segment ID |
| residue | string | Target state (see Verified Patch Values) |
| resid | string | Residue number |

Example: `add_prot('PROA', 'ASPP', '77');`

### add_ssbond(segid1, resid1, segid2, resid2)

Adds a disulfide bond between two CYS residues.

| Parameter | Type | Description |
|---|---|---|
| segid1 | string | Segment ID of first CYS |
| resid1 | string | Residue number of first CYS |
| segid2 | string | Segment ID of second CYS |
| resid2 | string | Residue number of second CYS |

Example: `add_ssbond('PROA', '23', 'PROA', '88');`

### Deleting the Default Row

When a section checkbox is enabled, CHARMM-GUI creates a default active row. The first `add_*()` call commits this default, creating an unwanted entry. **This applies to: phosphorylation, Lys/Arg PTMs, protonation, and mutation.**

After calling `add_*()` for all entries in a section, delete the first committed row (the default):

```js
// Delete default row — pattern: first '-' button in the section table
// Confirmed table IDs: id_phos_table, id_ptm_table, id_prot_table, id_mutation_table
['id_phos_table', 'id_ptm_table', 'id_prot_table', 'id_mutation_table'].forEach(tableId => {
  const table = document.getElementById(tableId);
  const btn = table?.querySelector('input[value="-"]');
  if (btn) btn.click();
});
```

Only include table IDs for sections that were actually used. If a section was not enabled, its table won't exist and the `?.` safely skips it.

### add_mutation()

Adds a mutation. **Takes NO parameters** — reads from dropdowns. For mutations, you must set dropdown values manually:

| Dropdown ID | Purpose |
|---|---|
| `mutation_chain_0` | Segment ID |
| `mutation_rid_0` | Residue ID to mutate |
| `mutation_res_0` | Current residue (read-only) |
| `mutation_patch_0` | Target amino acid |

After setting dropdowns, call `add_mutation()`.

---

## Verified Patch Values

> **Extracted directly from `charmm_patch` on the live CHARMM-GUI page.** Do not guess or invent patch names — use only these values.

### Phosphorylation (`charmm_patch['phos']`)

| Residue | Charge -1 | Charge -2 |
|---|---|---|
| SER | `SP1` | `SP2` |
| THR | `THP1` | `THPB` |
| TYR | `TP1` | `TP2` |
| ARG | `RP1` | `RP2` |

### Lys/Arg PTMs (`charmm_patch['ptm']`)

**LYS modifications:**

| Patch | Description |
|---|---|
| `KAC` | Nε-acetylated |
| `KN1M` | Nε-monomethylated |
| `KN2M` | Nε-dimethylated |
| `KN3M` | Nε-trimethylated |
| `KSUC` | Nε-succinylated |
| `KLAC` | Nε-lactylated |
| `KN6C` | (additional Lys modification) |
| `KCBM` | Nε-carboxymethylated |

**ARG modifications:**

| Patch | Description |
|---|---|
| `RCBM` | Carboxymethylated |
| `RCIR` | Citrullinated |

### Protonation (`charmm_patch['protonation']`)

| Source Residue | Available Target States |
|---|---|
| ASP | `ASPP` (protonated) |
| GLU | `GLUP` (protonated) |
| LYS | `LSN` (neutral / deprotonated) |
| ARG | `RN1`, `RN2`, `RN3` (deprotonated variants) |
| CYS | `CYM` (thiolate / deprotonated) |
| HIS | `HSP` (doubly protonated), `HSD` (delta), `HSE` (epsilon) |
| HSP | `HSD`, `HSE` |
| HSD | `HSP`, `HSE` |
| HSE | `HSP`, `HSD` |

---

## Checkbox Reference

Every modification section has a checkbox that **must be enabled** for the server to process that section's entries.

| Section | Checkbox ID |
|---|---|
| Model missing residues | `missing_checked` |
| Preserve hydrogen coordinates | `hbuild_checked` |
| Mutation | `mutation_checked` |
| Protonation state | `prot_checked` |
| Disulfide bonds | `ssbonds_checked` |
| Phosphorylation | `phos_checked` |
| Ubiquitylation / SUMOylation | `ubq_checked` |
| GPI anchor | `gpi_checked` |
| Glycosylation / Glycan Ligand(s) | `glyc_checked` |
| Heme coordination | `heme_checked` |
| Add Lipid-tail | `ltl_checked` |
| Peptide Stapling | `stapling_checked` |
| FRET/LRET fluorophore labels | `ret_checked` |
| Model LBT-loop(s) | `lbt_checked` |
| MTS spin labels | `epr_checked` |
| MTS chemical modifier | `mts_checked` |
| Non-standard amino acid / RNA substitution | `uaa_checked` |
| Lys / Arg PTMs | `ptm_checked` |

---

## Chain Selection (Step 1b)

Chain checkboxes use the name attribute pattern `chains[{SEGID}][checked]` where `{SEGID}` is the segment ID (e.g., `PROA`, `DNAA`, `HETA`, `WATA`).

Segment ID prefixes:
- `PRO*` — Protein chains (PROA, PROB, PROC, ...)
- `DNA*` — DNA chains (DNAA, DNAB, ...)
- `HET*` — Heteroatom/ligand chains
- `WAT*` — Water chains

Select a specific chain:
```js
document.querySelector('input[name="chains[PROA][checked]"]')
```

---

## Terminal Patches

Terminal patches are always visible (no checkbox). Set via `name` attribute selectors:

| Dropdown | Name Attribute | Options |
|---|---|---|
| N-terminal (First) | `terminal[{SEGID}][first]` | `NTER`, `NNEU`, `ACE`, `ACP`, `NONE` |
| C-terminal (Last) | `terminal[{SEGID}][last]` | `CTER`, `CNEU`, `CT1`, `CT2`, `CT3`, `NONE` |

Replace `{SEGID}` with the segment ID (e.g., `PROA`, `PROB`).

```js
document.querySelector('select[name="terminal[PROA][first]"]').value = 'NNEU';
document.querySelector('select[name="terminal[PROA][last]"]').value = 'CNEU';
```

Common choices:
- `NTER` / `CTER` — standard charged termini (default)
- `NNEU` / `CNEU` — neutral termini
- `ACE` / `CT3` — acetyl / N-methylamide caps
- `NONE` — no patch

---

## All PDB Manipulation Sections

### Sections with Direct Function Call API

These sections use `add_*()` functions with parameters. Enable the checkbox, then call the function for each entry.

**1. Phosphorylation** — `add_phos(chain, residue, resid, patch)`
Phosphorylate SER, THR, TYR, or ARG. Two protonation states per residue (charge -1 and -2). Checkbox: `phos_checked`.

**2. Lys/Arg PTMs** — `add_ptm(chain, residue, resid, patch)`
Acetylation, methylation, succinylation, lactylation, carboxymethylation on LYS. Carboxymethylation and citrullination on ARG. Checkbox: `ptm_checked`.

**3. Protonation State** — `add_prot(chain, residue, resid)`
Change protonation of ASP, GLU, LYS, ARG, CYS, HIS. Three parameters only — no patch. Checkbox: `prot_checked`.

**4. Disulfide Bonds** — `add_ssbond(segid1, resid1, segid2, resid2)`
Define CYS-CYS disulfide bridges. Checkbox: `ssbonds_checked`.

### Sections Requiring Dropdown Manipulation

These sections' add functions take no useful parameters or use popup editors.

**5. Mutation** — `add_mutation()`
Takes no parameters. Set dropdowns manually (`mutation_chain_0`, `mutation_rid_0`, `mutation_patch_0`), then call `add_mutation()`. Checkbox: `mutation_checked`.

**6. Ubiquitylation / SUMOylation**
Add ubiquitin or SUMO chains to target LYS residues. Uses a popup editor accessed via the "edit" button (`openEditUbq(this)`). The predefined ubiquitin model (PDB 1UBQ) or SUMO model (PDB 1A5R) is attached with rigid-body search to avoid clashes. Checkbox: `ubq_checked`. Button: `add_ubq(this)`.

**7. Glycosylation / Glycan Ligand(s)**
Add N-linked or O-linked glycans. Uses a popup editor accessed via the "edit" button (`openEditGlycan(this)`). Optional CHARMM MC checkbox (`run_mc_glycan`) for conformational sampling. Complex multi-step workflow — take a snapshot after enabling to understand the sub-editor. Checkbox: `glyc_checked`. Button: `add_glyc(this)`.

**8. Non-standard Amino Acid / RNA Substitution**
Replace standard residues with any of 354 non-standard amino acids or 49 non-standard nucleotides. Uses a popup image selector accessed via the "select image" button (`openUAA(this)`). Dropdown IDs: `uaa_chain_0`, `uaa_rid_0`, `uaa_res_0`, `uaa_patch_0`. Checkbox: `uaa_checked`. Button: `add_uaa()`.

### Sections with Simple Dropdown + Add Button

These sections have a straightforward dropdown → add button workflow but their add functions may or may not accept parameters. Use dropdowns as fallback.

**9. Add Lipid-tail (Lipidation)**
Attach acyl tails for membrane-anchored proteins. Patch types: `CYSP` (palmitoylation), `CYSF` (farnesylation), `CYSG` (geranylgeranylation), `CYSL` (triacylation), `GLYM` (Gly myristoylation), `LYSM` (Lys myristoylation). Dropdown IDs: `ltl_chain_0`, `ltl_rid_0`, `ltl_patch_0`. Checkbox: `ltl_checked`. Button: `add_ltl()`.

**10. Peptide Stapling**
Cross-link two residues with conformational staples. Types: `META3`–`META7` (ring-closing metathesis), `RMETA3`–`RMETA7` (reverse). Dropdown IDs: `stapling_type_0`, `stapling_chain1_0`, `stapling_rid1_0`, `stapling_chain2_0`, `stapling_rid2_0`. Checkbox: `stapling_checked`. Button: `add_stapling()`.

**11. FRET/LRET Fluorophore Labels**
Attach fluorescent labels. Patch types: `CBDP`, `CY3R`, `CY3S`, `CY5R`, `CY5S`, `CBPI`. Dropdown IDs: `ret_chain_0`, `ret_rid_0`, `ret_patch_0`. Checkbox: `ret_checked`. Button: `add_ret()`.

**12. MTS Spin Labels (EPR)**
Attach nitroxide spin labels. Patch types: `CYR1`, `CYR5`, `CY2R`, `CBRT`, `ONDUM`, `ONDUM2`, `ONDUM4`. Dropdown IDs: `epr_chain_0`, `epr_rid_0`, `epr_patch_0`. Additional inter-label site dropdowns: `epr_ichain_0`, `epr_irid_0`. Checkbox: `epr_checked`. Button: `add_epr()`.

**13. MTS Chemical Modifier**
Attach chemical modification reagents. Patch types: `MT1`–`MT7`. Dropdown IDs: `mts_chain_0`, `mts_rid_0`, `mts_patch_0`. Checkbox: `mts_checked`. Button: `add_mts()`.

**14. Model LBT-loop(s)**
Model lanthanide-binding tag loops for paramagnetic NMR. Dropdown IDs: `lbt_chain_0_0`, `lbt_rid_0_0`, `lbt_atom_0_0`. Checkbox: `lbt_checked`. Button: `add_lbt()`.

### Checkbox-Only Sections

These sections are enabled/disabled by a checkbox with no additional entries to add.

**15. Model Missing Residues**
Builds coordinates for residues absent from the crystal structure. Sub-checkboxes for each missing segment (e.g., `missing[PROA][0][checked]`). Checkbox: `missing_checked`.

**16. Preserve Hydrogen Coordinates**
Retains existing hydrogen positions from the input PDB. Checkbox: `hbuild_checked`.

**17. GPI Anchor**
Attach glycosylphosphatidylinositol anchor to C-terminus. Single-entry — dropdown for chain selection (`gpi_chain`). Checkbox: `gpi_checked`.

**18. Heme Coordination**
Set up heme coordination to HIS/TYR residues. Dropdown IDs: `heme_chain_`, `heme_res_`, `heme_rid_`. Checkbox: `heme_checked`.

---

## Workflow

### Step 1 — Upload PDB & Select Chains

1. Navigate to `https://www.charmm-gui.org/?doc=input/pdbreader`.
2. Take a snapshot to see the upload form.
3. For RCSB download: enter the PDB ID and select the download radio option.
   For local upload: use `mcp_chrome-devtoo_upload_file`.
4. Click Next Step and wait for the page to load.
5. Take a snapshot to identify chains.
6. **Select chains by segment ID** — checkboxes have `name="chains[{SEGID}][checked]"`. Select directly by name, no table parsing:
   ```js
   () => {
     const keep = ['PROA'];  // segment IDs to keep
     document.querySelectorAll('input[type="checkbox"][name^="chains["]').forEach(cb => {
       const match = cb.name.match(/chains\[([A-Z0-9]+)\]\[checked\]/);
       if (!match) return;
       const segid = match[1];
       const shouldCheck = keep.includes(segid);
       if (cb.checked !== shouldCheck) cb.click();
     });
     return 'Selected: ' + keep.join(', ');
   }
   ```
7. Click Next Step.

### Step 2 — Modifications

This is the main step. All modifications happen on one page in a single evaluate_script call.

**Pattern for each section:**
1. Enable the section checkbox
2. Call the add function with parameters (or set dropdowns for mutation/complex sections)
3. Repeat for all entries

**Example: one evaluate_script that does everything:**
```js
() => {
  const results = {};

  // --- Terminal patches ---
  document.querySelector('select[name="terminal[PROA][first]"]').value = 'NNEU';
  document.querySelector('select[name="terminal[PROA][last]"]').value = 'CNEU';
  results.terminals = 'NNEU/CNEU';

  // --- Enable checkboxes for needed sections ---
  ['phos_checked', 'ptm_checked', 'missing_checked'].forEach(id => {
    const cb = document.getElementById(id);
    if (cb && !cb.checked) cb.click();
  });

  // --- Phosphorylation (2 entries) ---
  // Each call COMMITS current active row, creates new row with params.
  // First call commits the unwanted default row. Last entry stays as active row.
  add_phos('PROA', 'SER', '10', 'SP2');
  add_phos('PROA', 'THR', '3', 'THPB');
  results.phos = 2;

  // --- Lys/Arg PTMs (1 entry) ---
  add_ptm('PROA', 'LYS', '9', 'KAC');
  results.ptm = 1;

  // --- Protonation (1 entry) ---
  add_prot('PROA', 'HSP', '37');
  results.prot = 1;

  // --- Disulfide bonds (1 entry) ---
  add_ssbond('PROA', '23', 'PROA', '88');
  results.ss = 1;

  // --- Delete default committed rows from all sections used ---
  ['id_phos_table', 'id_ptm_table', 'id_prot_table'].forEach(id => {
    document.getElementById(id)?.querySelector('input[value="-"]')?.click();
  });

  return JSON.stringify(results);
}
```

**Key points:**
- Call `add_*()` N times for N entries. The first call commits the unwanted default row.
- After all calls, delete the first committed row (the default) by clicking the first `-` button in that section's table (`id_{section}_table`).
- The last entry stays as the active row and will be submitted with Next Step.
- The agent fills in ONLY the sections and entries from the user's spec. Omit any section not requested.

### Step 2b — Validate

After applying modifications, run one more evaluate_script to read back committed rows and verify:

```js
() => {
  const rows = [];
  document.querySelectorAll('table tr').forEach(row => {
    const cells = Array.from(row.querySelectorAll('td')).map(c => c.textContent.trim());
    if (cells.length >= 3) rows.push(cells.join(' | '));
  });
  const nter = document.querySelector('select[name="terminal[PROA][first]"]');
  const cter = document.querySelector('select[name="terminal[PROA][last]"]');
  return JSON.stringify({
    terminals: { nter: nter?.value, cter: cter?.value },
    committed_rows: rows.filter(r => r.length > 5)
  }, null, 2);
}
```

If validation passes, click Next Step in the same call or a follow-up call.

### Step 3 — Generated Output

1. Wait for CHARMM processing. **Timeout: 300000 ms (5 min)** for proteins with many modifications.
2. Take a snapshot and check for errors — look for "error", "Error", "FATAL", or red banners.
3. If the page shows the output file listing, inform the user that processing is complete and the archive is ready for download.
4. **Do NOT attempt to click download.tgz or extract files.** The user will download and extract the archive manually. This is a deliberate human-in-the-loop checkpoint — the user should inspect the output before proceeding to the Force Field Converter.
5. **Key output files the user should expect in the archive:**
   - `step1_pdbreader.psf` — CHARMM protein structure file
   - `step1_pdbreader.crd` — CHARMM coordinate file
   - `step1_pdbreader.pdb` — Modified PDB file
   - `step1_pdbreader.out` — CHARMM output log (user should check for warnings)
   - `step1_pdbreader.inp` — CHARMM input script
   - `toppar/` — Force field topology/parameter files
   - `toppar.str` — Stream file that loads all toppar files
   - `toppar/toppar_all36_ptm_imlab.str` — PTM parameters (if PTMs applied)
6. If the user wants to continue to Force Field Converter, they should upload the PSF, CRD, and relevant toppar files and invoke the FF Converter skill.

---

## Retry and Recovery

### Retry Logic
- If page doesn't load after Next Step: take a snapshot, check for progress indicator, extend wait by 120s.
- If error banner appears: capture full text, report to user.
- Retry Next Step up to 2 times with 30s waits.

### Session Expiry
- If redirected to login page: inform user, ask them to log in again.
- After re-login, use **Job Retriever** at `https://www.charmm-gui.org/?doc=input/retriever` to attempt recovery.

### Common Errors

| Error | Meaning | Recovery |
|---|---|---|
| "Atom X not found" | PDB atom naming mismatch | User needs to clean PDB |
| "Parameter not found" | Missing FF parameters for a modification | Check if correct toppar files are included |
| "Duplicate atom" | Alternate conformations in PDB | Pre-process PDB to remove altlocs |
| "Session expired" | Inactivity timeout | Re-login + Job Retriever |

---

## MCP Tools Used

| Tool | Purpose |
|------|---------|
| `mcp_chrome-devtoo_navigate_page` | Navigate to CHARMM-GUI URLs |
| `mcp_chrome-devtoo_take_snapshot` | Read current page state |
| `mcp_chrome-devtoo_click` | Click buttons, radio buttons |
| `mcp_chrome-devtoo_fill` | Fill text inputs |
| `mcp_chrome-devtoo_upload_file` | Upload PDB files |
| `mcp_chrome-devtoo_evaluate_script` | Run JavaScript for all operations |
| `mcp_chrome-devtoo_wait_for` | Wait for page loads |

---

## Reference

| Resource | URL |
|----------|-----|
| PDB Reader & Manipulator | `https://www.charmm-gui.org/?doc=input/pdbreader` |
| Job Retriever | `https://www.charmm-gui.org/?doc=input/retriever` |
| PDB Manipulator paper | Park et al., J Mol Biol 435:167995, 2023 |
| Covalent ligand paper | Park et al., J Mol Biol 436:168655, 2024 |
