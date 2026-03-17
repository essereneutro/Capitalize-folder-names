# Capitalize folder names

A batch renaming script.

This script renames folders and subfolders by applying capitalization rules to folder names.

It capitalizes the first letter of each word while lowercasing the remaining letters, with special handling for text inside square brackets, words following characters such as hyphens, parentheses, quotation marks, underscores, and dots, and contractions with apostrophes.

It also preserves or adjusts specific patterns, including Roman numerals (`II`, `III`, `IV`, etc.), time formats such as `9AM` and `12PM`, abbreviations such as `DJ`, `EP`, `LP`, `OST`, `USA`, and `UK`, names beginning with `Mc`, and words such as `Mix` and `Version`.

## Instructions

Open Terminal, navigate to the target folder using `cd folder_name` (or drag and drop the folder into Terminal, if supported), press Enter, then paste and run the following code:

```bash
find . -depth -type d | while IFS= read -r dir; do
    [ "$dir" = "." ] && continue

    base="${dir##*/}"
    parent="${dir%/*}"

    target=$(
        printf "%s" "$base" | LC_ALL=C awk '{
            inbr = 0
            for (i = 1; i <= NF; i++) {
                out  = ""
                word = $i

                if (inbr || word ~ /^\[/) {
                    $i = toupper(word)
                    if (word ~ /\]/) inbr = 0
                    else inbr = 1
                    continue
                }

                for (j = 1; j <= length(word); j++) {
                    c    = substr(word, j, 1)
                    prev = (j > 1 ? substr(word, j-1, 1) : "")

                    if (prev == "'"'"'" || prev == "’") {
                        c = tolower(c)
                    } else if (j == 1 ||
                               prev == "-" ||
                               prev == "(" ||
                               prev == "\"" ||
                               prev == "_" ||
                               prev == ".") {
                        c = toupper(c)
                    } else {
                        c = tolower(c)
                    }

                    out = out c
                }

                lw = tolower(out)

                if (lw ~ /^(1[0-2]|[1-9])(am|pm)$/) {
                    num = substr(lw, 1, length(lw)-2)
                    suf = toupper(substr(lw, length(lw)-1))
                    out = num suf
                }

                if (lw == "ii"  || lw == "iii" || lw == "iv"  ||
                    lw == "v"   || lw == "vi"  || lw == "vii" ||
                    lw == "viii"|| lw == "ix"  || lw == "x"   ||
                    lw == "xi"  || lw == "xii")
                    out = toupper(lw)

                if (lw == "dj" || lw == "ep" || lw == "lp" ||
                    lw == "ost"|| lw == "usa"|| lw == "uk")
                    out = toupper(lw)

                if (tolower(substr(out,1,2)) == "mc" && length(out) > 2)
                    out = substr(out,1,2) toupper(substr(out,3,1)) substr(out,4)

                if (lw == "mix")     out = "Mix"
                if (lw == "version") out = "Version"

                if (out ~ /'\''nt$/) out = substr(out,1,length(out)-2) "nt"
                if (out ~ /'\''ll$/) out = substr(out,1,length(out)-2) "ll"
                if (out ~ /'\''ve$/) out = substr(out,1,length(out)-2) "ve"
                if (out ~ /'\''re$/) out = substr(out,1,length(out)-2) "re"
                if (out ~ /'\''d$/)  out = substr(out,1,length(out)-1) "d"
                if (out ~ /'\''s$/)  out = substr(out,1,length(out)-1) "s"

                $i = out
            }

            print
        }'
    )

    [ "$base" = "$target" ] && continue

    tmp="${parent}/${target}_TMP_$$"
    final="${parent}/${target}"

    if ! mv -- "$dir" "$tmp"; then
        printf "[PHASE 1 ERROR] ORIG: \"%s\" -> TMP: \"%s\" (target: \"%s\")\n" \
            "$dir" "$tmp" "$final" | tee -a rename_errors.log
        continue
    fi

    if [ -e "$final" ]; then
        printf "[SKIP] Target already exists: \"%s\" -> \"%s\"\n" \
            "$dir" "$final" | tee -a rename_errors.log
        mv -- "$tmp" "$dir"
        continue
    fi

    if ! mv -- "$tmp" "$final"; then
        printf "[PHASE 2 ERROR] TMP: \"%s\" -> TARGET: \"%s\" (original: \"%s\")\n" \
            "$tmp" "$final" "$dir" | tee -a rename_errors.log
        mv -- "$tmp" "$dir"
    fi
done
