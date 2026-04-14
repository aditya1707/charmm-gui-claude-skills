### USER PROMPT FOR CLAUDE - Manipulate PDB 1KX5 TO ADD PTMS ###

Open a new browser tab and navigate to the CHARMM-GUI website. Verify that I am logged in by looking for a Logout button. If I am not logged in, tell me and stop. If I am logged in, navigate to PDB Reader and Manipulator.

I want to process PDB 1KX5 (nucleosome core particle). Download it directly from RCSB using the PDB ID field — do not upload a file. After the PDB loads and the chain selection page appears, select only the first histone H3 chain (PROA) and deselect everything else. Do this in a single JavaScript call, not by clicking individual checkboxes. Proceed to the next step.

On the manipulation options page, apply everything below in a single JavaScript call. Remember that each section's checkbox must be enabled before calling the add functions, otherwise the server will ignore the modifications.

Terminal patches: Set both termini for PROA to neutral — NNEU for the N-terminal and CNEU for the C-terminal.

Model missing residues: Enable this option.

Phosphorylation: Enable the phosphorylation checkbox, then add these four entries:
- PROA, SER, 10, SP2
- PROA, SER, 28, SP1
- PROA, THR, 3, THPB
- PROA, THR, 11, THP1

Lysine acetylation: Enable the Lys/Arg PTM checkbox, then add these four entries:
- PROA, LYS, 9, KAC
- PROA, LYS, 14, KAC
- PROA, LYS, 18, KAC
- PROA, LYS, 27, KAC

After applying everything, validate by reading back the committed rows from the page and the terminal patch dropdown values. Report what you find. If it looks correct, click Next Step.

Wait for CHARMM to finish processing — this may take several minutes. When the output page loads, check for any error messages on the page and let me know. I will download the archive myself.

Do not ask me to confirm at each step. Do not pause to present options. Do not summarize what you see on each page. Apply the spec and proceed. Only stop if something fails.
