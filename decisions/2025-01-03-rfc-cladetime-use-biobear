# 2025-01-03 incoporate biobear package into Cladetime

## Context

The [Cladetime package](https://cladetime.readthedocs.io) has a feature that
uses a reference tree from a specific point in time to assign clades to
SARS-CoV-2 sequences.

One use of Cladetime is to generate target data for the
[COVID-19 Variant Nowcast Hub](https://github.com/reichlab/variant-nowcast-hub),
and its current functionality is sufficient for that purpose.

However, the clade assignment process is very slow. The largest bottleneck is
the code that filters an entire universe of of SARS-CoV-2 sequences into a
smaller subset of sequences to use for clade assignment.

The filtering process is slow because Cladetime downloads the sequence file
from S3 and then operates on it line-by-line:

- Read a sequence from the downloaded file
(using biopython's [SeqIO interface](https://biopython.org/wiki/SeqIO))
- If the sequence ID is in our list of "sequences we want for clade assignment",
write the sequence to the filtered file (again using biopython).

The result is a smaller sequence file that we pass to the Nextclade CLI for
clade assignment.

Other important context:

- The sequence data is in
[.fasta format](https://en.wikipedia.org/wiki/FASTA_format).
- Nextstrain publishes the sequence data as a single, compressed file.
- Input to the Nextclade CLI must be a file on disk (the CLI won't accept a
data stream, for example).

### Aims

- Speed up the
[clade assignment process](https://cladetime.readthedocs.io/en/latest/reference/cladetime.html#cladetime.CladeTime.assign_clades).
- Maintain Cladetime's ability to run on larger-than-memory datasets.

### Anti-Aims

- We do not want to re-write or replace the current process for doing custom
clade assignments.
- Speed of clade assignment is important, but should not take priority over code
that is understandable and maintainable.

## Decision

This RFC proposes to introduce the
[`biobear` package](https://github.com/wheretrue/biobear) to Cladetime,
as a faster mechanism for reading and filtering .fasta files.

Biobear delivers on the speed improvement (by ~65% in a limited test), but there
are some red flags about the package that merit wider consideration and
feedback (see [consequences](#consequences) below).

### About Biobear

- reads bioinformatic file formats
- is built in Rust (like Polars, uv, and other speedy Python tools used by the
lab)
- integrates with Arrow and DuckDB

### Sample code

Biobear can read a .fasta file into an Arrow
[RecordBatchReader](https://arrow.apache.org/docs/python/generated/pyarrow.RecordBatchReader.html),
allowing Cladetime to consume the sequence data in batches rather than
line by line.

Each batch becomes a Polars dataframe we can use for filtering on
sequence of interest and writing them to the smaller file.

```python
session = bb.new_session()
    sequence_batches = session.read_fasta_file(
        "/Users/rsweger/code/sequences.fasta.zst"
    ).to_arrow_record_batch_reader()
    for batch in sequence_batches:
        batch_df = pl.DataFrame(batch)
        sequence_count += len(batch_df)
        id_list = batch_df.get_column("id").to_list()
        sequence_list = batch_df.get_column("sequence").to_list()
        seq_match_list = [
            SeqRecord(Seq(sequence), id=id)
            for id, sequence in zip(id_list, sequence_list)
            if id in sequence_ids
        ]
        sequence_match_count += SeqIO.write(seq_match_list, fasta_output, "fasta")
```

### Other Options Considered

- Leave Cladetime as is (if we choose not to use biobear, this would be
my reccommendation).
- Since Cladetime already uses biopython, use biopython's built-in batch reader
(discarded because it made filtering slower, see table below).
- Roll our own reader for .fasta parsing (discarded this idea quickly, as it
adds complexity to Cladetime and duplicates what biobear provides).
- Stream Nextstrain's .fasta file from S3 instead of downloading it to disk;
filter incoming chunks (discarded because the code was slower
than current state, though I forgot to record the exact timings).

The test sequence assignment runs below used the Nextstrain sequence file
published on 2024-12-19. The subset "sequences of interest" were .48% of the
total published sequences and defined as follows:

- collection date between 2023-01-01 and 2023-01-31 (inclusive)
- sequence collected in the US
- sequence collected from a human

| Approach                  | batch sz | filtered seq   | total seq         | wall time (sec)   |
|---------------------------|----------|----------------|-------------------|-------------------|
| No change     | n/a   | 44,668     | 9,237,594     | 277.59    |
| biopython batch reader, polars as filtering intermediary    | 50,000   | 44,668     | 9,237,594 | 418.44    |
| biopython batch reader, no polars conversion    | 50,000   | 44,668     | 9,237,594 | 318.26    |
| biopython batch reader, no polars conversion    | 10,000   | 44,668     | 9,237,594 | 315.98    |
| biopython batch reader/writer, no polars conversion    | 10,000   | 44,668     | 9,237,594     | 317.98    |
| biopython batch reader/writer, no polars conversion    | 20,000   | 44,668     | 9,237,594     | 318.24    |
| biopython batch reader/writer, no polars conversion    | 50,000   | 44,668     | 9,237,594     | 320.12    |
| biobear pyarrow batch records read/polars for filtering | 8,192*   | 44,668     | 9,237,594     | 97.89     |

*batch size not configurable

## Status

proposed

## Consequences

- In the limited test, incorporating biobear reduced the filtering time by
nearly 65% 😮
- Unlike biopython, which has hundreds of contributors, is widely-used, and is
well-documented, biobear is the work of a single person, with incomplete
documentation.
- Biobear is written in Rust, thus would be a learning curve for lab staff
to fork/update if the current maintainer steps back.

## Projects

[Cladetime](https://github.com/reichlab/decisions/blob/main/project-posters/cladetime.md)
