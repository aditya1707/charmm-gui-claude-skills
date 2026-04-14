# CHARMM-GUI PDB Reader — AI Agent Skill

An AI agent skill that automates [CHARMM-GUI](https://www.charmm-gui.org/) PDB Reader & Manipulator through browser automation. Instead of manually clicking through dozens of dropdowns to set up post-translational modifications, protonation states, and other structural modifications, you describe what you want in natural language and the agent does it.

## The Problem

Setting up a protein structure in CHARMM-GUI PDB Reader for MD simulation involves:

- Selecting chains from multi-chain complexes (e.g., nucleosome has 8 protein chains + 2 DNA chains + waters + ions)
- Setting terminal patches
- Applying phosphorylation to specific residues with specific protonation states
- Applying Lys/Arg post-translational modifications (acetylation, methylation, etc.)
- Setting protonation states for titratable residues
- Defining disulfide bonds
- Configuring up to 19 different modification sections

For a protein with many PTMs, this means hundreds of mouse clicks on dropdowns. Mistakes require starting over. Exploring different protonation states means repeating the entire process.

## The Solution

This skill teaches an AI agent the exact DOM structure, JavaScript function signatures, and form mechanics of CHARMM-GUI's PDB Reader. The agent can:

- Select specific chains from a PDB in a single JavaScript call
- Apply any combination of the 19 modification types available in PDB Reader
- Call CHARMM-GUI's internal functions (`add_phos`, `add_ptm`, `add_prot`, `add_ssbond`) directly with parameters
- Validate committed entries before submission
- Handle the default-row cleanup that CHARMM-GUI's form mechanics require

All patch values in the skill are **verified from CHARMM-GUI's live `charmm_patch` JavaScript object** — not guessed or copied from documentation.

## What's in This Repo

| File | Description |
|------|-------------|
| `SKILL.md` | The complete skill file — install this in your Claude Project |
| `example_prompt.md` | Example prompt that processes histone H3 from PDB 1KX5 with 4 phosphorylations + 4 lysine acetylations |

## Prerequisites

### 1. Claude with Chrome DevTools MCP

The skill requires Claude to have access to Chrome DevTools MCP tools (prefixed `mcp_chrome-devtoo_*`). Setup depends on your Claude environment:

**Claude.ai (Web/Desktop App)**
- Add Chrome DevTools as an MCP integration in your Claude.ai settings/developer panel

**Claude Code (Terminal)**
- Install the Chrome DevTools MCP server:
  ```bash
  claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
  ```
- Or configure manually in `~/.claude.json`:
  ```json
  {
    "mcpServers": {
      "chrome-devtools": {
        "command": "npx",
        "args": ["-y", "chrome-devtools-mcp@latest"]
      }
    }
  }
  ```
- Launch Chrome with remote debugging:
  ```bash
  # macOS
  /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug

  # Linux
  google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
  ```

### 2. CHARMM-GUI Account

- Register at https://www.charmm-gui.org/ (free for academic use)
- Log in **before** starting the agent workflow — the skill checks for a Logout button to confirm login

### 3. Install the Skill

- Create a Claude Project
- Add `SKILL.md` to the project's skills directory
- For the Claude desktop app, add the `SKILL.md` file to the Customize/Skills panel

## Supported Modifications

The skill covers all 19 modification sections in CHARMM-GUI PDB Reader Step 2:

| Section | Automation Method |
|---------|-------------------|
| Terminal patches | Direct dropdown setting |
| Model missing residues | Checkbox toggle |
| Preserve hydrogen coordinates | Checkbox toggle |
| Phosphorylation (SER/THR/TYR/ARG) | `add_phos()` with parameters |
| Lys/Arg PTMs | `add_ptm()` with parameters |
| Protonation state | `add_prot()` with parameters |
| Disulfide bonds | `add_ssbond()` with parameters |
| Mutation | Dropdown setting + `add_mutation()` |
| Ubiquitylation / SUMOylation | Popup editor |
| GPI anchor | Checkbox + dropdown |
| Glycosylation | Popup editor |
| Heme coordination | Checkbox + dropdown |
| Lipidation (Lipid-tail) | Dropdown + `add_ltl()` |
| Peptide stapling | Dropdown + `add_stapling()` |
| FRET/LRET labels | Dropdown + `add_ret()` |
| LBT loops | Dropdown + `add_lbt()` |
| MTS spin labels | Dropdown + `add_epr()` |
| MTS chemical modifier | Dropdown + `add_mts()` |
| Non-standard AA/RNA substitution | Popup image selector |

## Example Usage

See `example_prompt.md` for a complete example that:
1. Downloads PDB 1KX5 (nucleosome) from RCSB
2. Selects only the histone H3 chain
3. Neutralizes termini (NNEU/CNEU)
4. Enables model missing residues
5. Phosphorylates S10 (SP2), S28 (SP1), T3 (THPB), T11 (THP1)
6. Acetylates K9, K14, K18, K27 (KAC)
7. Validates all entries
8. Submits for processing

## License

MIT

## References

- [CHARMM-GUI PDB Manipulator paper](https://doi.org/10.1016/j.jmb.2023.167995) — Park et al., J Mol Biol 435:167995, 2023
- [CHARMM-GUI Covalent Ligand paper](https://doi.org/10.1016/j.jmb.2024.168655) — Park et al., J Mol Biol 436:168655, 2024
- [Chrome DevTools MCP](https://github.com/anthropics/chrome-devtools-mcp)
