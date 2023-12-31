# RoBERTa Pretraining Preprocessing Guide

This guide walks through downloading, formatting, and encoding the Wikipedia and
books corpora for pretraining RoBERTa.

This guide assumes you have installed the `llm` packages and its dependencies as described in the [Installation Guide](../installation/index.md).

Note: using both the Wikipedia and books corpora is not necessary, and you can
skip one or the other. These instructions will also work for your own text
corpora---just skip the download step.

## Download

Create a `datasets/` directory. This directory will contain all of the
files produced.
```
$ mkdir datasets/
```

Download the datasets.
```bash
$ python -m llm.preprocess.download --dataset wikipedia --output-dir datasets/downloaded/
$ python -m llm.preprocess.download --dataset bookscorpus --output-dir datasets/downloaded/
```
This will result in two files: `datasets/downloaded/wikipedia-{date}.en.txt`
and `datasets/downloaded/bookcorpus.txt`.
Each of these files has the format one sentence per line with documents
separated by blank lines.

## Shard

Next we shard the files to make them easier to work with. For small dataset this
may not be needed.

```bash
$ python -m llm.preprocess.shard datasets/downloaded/*.txt --output-dir datasets/sharded/wikibooks/ --size 250MB
```

Now we have a set of sharded files in `datasets/sharded/wikibooks/` that are
each approximately 250 MB. The format of these files is still one sentence
per line with documents separated by blank lines.

## Tokenzier

Now we train a tokenizer on the text corpus.
BPE and wordpiece tokenizers are supported as well as cased/uncased.
This example creates an uncased wordpiece tokenizer will 50,000 tokens.

```bash
$ python -m llm.preprocess.tokenizer datasets/sharded/wikibooks/* \
      --output-file datasets/tokenizers/wikibooks-wordpiece.json \
      --tokenizer wordpiece \
      --size 50000 \
      --uncased
```

The resulting JSON file can be loaded to get the trained tokenizer.
```bash
from tokenizers import Tokenizer

tokenizer = Tokenizer.from_file('datasets/tokenizers/wikibooks-wordpiece.json')
```

## Encode the Shards

Now we can encode the shards using our tokenizer.

```bash
$ python -m llm.preprocess.roberta datasets/sharded/wikibooks/* \
      --output-dir datasets/encoded/wikibooks/ \
      --tokenizer datasets/tokenizers/wikibooks-wordpiece.json \
      --max-seq-len 512 \
      --processes 4
```

Encoding a 250MB shard can take around 16GB of RAM so adjust the number of
processes (parallel workers encoding a shard) as needed to fit your shard size
and system memory.

This produces a set of encoded HDF5 files in `datasets/encoded/wikibooks`
with each encoded file corresponding to a shard. Each encoded file contains
the `input_ids`, `attention_masks`, and `special_tokens_masks` attributes
which are each numpy arrays of shape `(samples, max_seq_len)`.

In contrast to BERT encoding, the samples are not pre masked and next sentence
prediction is not done. In other words, masking must be done at runtime and
each sample represents contiguous sentences draw from the same document until
the max sequence length is reached.
