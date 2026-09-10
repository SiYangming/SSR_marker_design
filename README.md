SSR_marker_design
=================

Galaxy Wrappers and helpers for the MISA tools

See http://pgrc.ipk-gatersleben.de/misa/

Publications
-----------

* Thiel et al 2003 http://www.ncbi.nlm.nih.gov/pubmed/12589540
* Baldwin et al 2012 http://dx.doi.org/10.1007/s11032-012-9727-6

Local additions in this fork
----------------------------

`misa_v2.1/`
: MISA v2.1 release (misa.pl, release date 25/08/2020; misa.ini with the
  optional `GFF: true` output switch). Newer than the v1.0 scripts at the
  repository root.

`misa_primer3/`
: Custom SSR-to-Primer3 pipeline.
  - `misa_primer3.pl` - extracts the flanking sequences of each SSR from the
    `.misa` result and designs primers with `primer3_core`
    (usage: `perl misa_primer3.pl genome.fasta.misa genome.fasta`).
  - `p3_settings.txt` - Primer3 settings used with this script.

SHA-256:

```
4712f29a0c57ff4aee1cc248cba20bcc879ef5d3b2ea6f5f1162cbc391c59abb  misa_v2.1/misa.pl
2dc25894c4d85314ae53eb7e615474285424c667e3b985d2ccde9b31edb4a6a2  misa_v2.1/misa.ini
c451d1d00aee00d2f7a1ac3c03b297bcc9ed7cd1dcddceec4f665abbaed80d60  misa_primer3/misa_primer3.pl
3222c34aab3accf8d937600cb39bd1c50618ee4259fba8abfc5ca572f7e1d53b  misa_primer3/p3_settings.txt
```


 

