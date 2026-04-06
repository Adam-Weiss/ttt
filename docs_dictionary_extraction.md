# Building and Extracting Standalone Dictionaries (TTT)

This repo’s standalone dictionary format is a 3-file set:

- `NAME.wrd` (binary trie of syllables/entries)
- `NAME.def` (binary UTF definition strings)
- `NAME.dic` (plain text dictionary list and abbreviations)

The generator is `org.thdl.tib.scanner.BinaryFileGenerator`.

## 1) Build a standalone dictionary from TSV

`BinaryFileGenerator` reads `*.txt` inputs by basename and supports tab-separated input via `-tab`.

### Single TSV dictionary

If your source file is `mydict.tsv`, do:

```bash
cp mydict.tsv mydict.txt
java -cp dist/lib-vanilla/DictionarySearchStandalone.jar \
  org.thdl.tib.scanner.BinaryFileGenerator -tab mydict
```

This generates:

- `mydict.wrd`
- `mydict.def`

If you want UI dictionary labels (used by selectors), create `mydict.dic` manually, for example:

```text
My Dictionary,mydict
```

### Combine multiple sources into one standalone dictionary

You can mix separators in one build; first arg is destination basename:

```bash
java -cp dist/lib-vanilla/DictionarySearchStandalone.jar \
  org.thdl.tib.scanner.BinaryFileGenerator combined \
  -tab dict_a \
  -tab dict_b \
  dict_dash
```

This writes `combined.wrd` + `combined.def`.

If you use the GUI "CreateDatabaseWizard", it also writes `.dic` (dictionary descriptions/abbreviations).

## 2) Access all internal dictionaries in text form

`.dic` is already plain text and lists dictionary descriptions + tags (abbreviations), one per line:

```text
Description,tag
```

Example from shipped data:

- first line is `Term ids, TID`
- following lines are human dictionary names and short tags.

So, to list internal dictionary labels:

```bash
cat dist/data/thl.dic
```

## 3) Extract all terms and term IDs

Yes, this is possible.

### Important detail

In `thl.dic`, dictionary index 0 is `Term ids, TID`. During lookup, “default definitions” (dictionary 0) are only included when `includeDefaultDefinition=true` (used in JSON/remote path). That is where term IDs come from.

In practice: each term node may include one “TID” definition string (often numeric) plus normal dictionary definitions.

### Fast practical extraction path (Python)

Use this script to dump the standalone `.wrd/.def` into TSV with:

- term (EWTS)
- term_id (if present from dictionary 0 / `TID`)
- all dictionary-tagged definitions

```python
#!/usr/bin/env python3
import struct
from pathlib import Path

base = Path("dist/data/thl")  # change as needed (without extension)

# ---- Java DataInput-compatible readers ----
def read_u1(f):
    b = f.read(1)
    if not b:
        raise EOFError
    return b[0]

def read_i4(f):
    b = f.read(4)
    if len(b) != 4:
        raise EOFError
    return struct.unpack(">i", b)[0]

def read_utf(f):
    # Java DataInput.readUTF(): 2-byte length + modified UTF-8 payload
    n = struct.unpack(">H", f.read(2))[0]
    data = f.read(n)
    # for dictionary content this usually decodes with utf-8 in practice
    return data.decode("utf-8", errors="replace")

# ---- load .dic tags ----
with open(base.with_suffix(".dic"), "r", encoding="utf-8", errors="replace") as f:
    dic_rows = [line.rstrip("\n") for line in f if line.strip()]

def split_dic_row(r):
    if "," in r:
        a, b = r.split(",", 1)
        return a.strip(), b.strip()
    return r.strip(), r.strip()

dict_meta = [split_dic_row(r) for r in dic_rows]
# dict_meta[i] = (description, tag)

# ---- collect definition strings from .def by byte offset ----
def_cache = {}
with open(base.with_suffix(".def"), "rb") as fdef:
    def read_def_at(offset):
        if offset in def_cache:
            return def_cache[offset]
        fdef.seek(offset)
        s = read_utf(fdef)
        def_cache[offset] = s
        return s

    # ---- parse .wrd recursively ----
    with open(base.with_suffix(".wrd"), "rb") as fwrd:
        size = fwrd.seek(0, 2)
        fwrd.seek(size - 8)
        root_pos = read_i4(fwrd)
        fwrd.seek(size - 1)
        version = read_u1(fwrd)
        if version != 3:
            raise RuntimeError(f"Expected v3 .wrd, got {version}")

        rows = []

        def read_byte_dict_source(f):
            # ByteDictionarySource.read():
            # first byte: 0..63 defs count, +64 if has brother
            n = read_u1(f)
            has_brother = bool(n & 64)
            defs_count = n & 63
            defs_dicts = []
            for _ in range(defs_count):
                dset = []
                while True:
                    x = read_u1(f)
                    dset.append(x & 63)
                    if not (x & 64):
                        break
                defs_dicts.append(dset)
            return has_brother, defs_dicts

        def walk_list(pos, prefix):
            fwrd.seek(pos)
            while True:
                child_pos = read_i4(fwrd)
                syll = read_utf(fwrd)
                has_brother, defs_dicts = read_byte_dict_source(fwrd)

                def_offsets = []
                for _ in range(len(defs_dicts)):
                    def_offsets.append(read_i4(fwrd))

                token = (prefix + " " + syll).strip()

                if defs_dicts:
                    defs = []
                    term_id = ""
                    for i, dset in enumerate(defs_dicts):
                        txt = read_def_at(def_offsets[i])
                        tags = [dict_meta[d][1] if d < len(dict_meta) else f"dict_{d}" for d in dset]
                        defs.append("|".join(tags) + ":" + txt)
                        if 0 in dset and not term_id:
                            term_id = txt
                    rows.append((token, term_id, " || ".join(defs)))

                if child_pos != -1:
                    walk_list(child_pos, token)

                if not has_brother:
                    break

        walk_list(root_pos, "")

# output TSV
print("term\tterm_id\tdefinitions")
for term, tid, defs in rows:
    print(f"{term}\t{tid}\t{defs}")
```

Run:

```bash
python extract_terms.py > all_terms.tsv
```

## 4) Notes on IDs and limitations

- There is no separate dedicated numeric ID field in `.wrd`; IDs are stored as definition text associated with dictionary 0 (`TID`) when present.
- If a dictionary build does not include a "Term ids" source, `term_id` can be blank.
- Definitions can map to multiple dictionary tags (the binary format supports grouped sources per definition).
