# Pyserini: A Deeper Dive into Dense and Sparse Representations

In a [previous guide](conceptual-framework.md), we introduced a conceptual framework for a representational approach to information retrieval that integrates dense and sparse representations into the same underlying (bi-encoder) architecture.
This guide offers a deeper dive that connects the high-level concepts with the actual code implementation.

If you're a Waterloo student traversing the [onboarding path](https://github.com/lintool/guide/blob/master/ura.md) (which [starts here](https://github.com/castorini/anserini/blob/master/docs/start-here.md)),
make sure you've first done the previous step, [reproducing a dense retrieval baseline for NFCorpus](experiments-nfcorpus.md).
In general, don't try to rush through this guide by just blindly copying and pasting commands into a shell;
that's what I call [cargo culting](https://en.wikipedia.org/wiki/Cargo_cult_programming).
Instead, really try to understand what's going on.

Following the onboarding path, this lesson does **not** introduce any new concepts.
Rather, the focus is to solidify previously introduced concepts and to connect the bi-encoder architecture to implementations in Pyserini.
Informally, we're "peeling back the covers".

**Learning outcomes** for this guide, building on previous steps in the onboarding path, are divided into three parts.
With respect to dense retrieval models:

1. Be able to materialize and inspect dense vectors stored in Faiss.
2. Be able to encode documents and queries with the Contriever model and manipulate the resulting vector representations.
3. Be able to compute query-document scores (i.e., retrieval scores) "by hand" for dense retrieval, by directly manipulating the vectors.
4. Be able to perform retrieval "by hand" given a query, by directly manipulating the document vectors stored in the index.

With respect to sparse (i.e., bag-of-words) retrieval models:

1. Be able to materialize and inspect BM25 document vectors from a Lucene inverted index.
2. Be able to compute query-document scores (i.e., retrieval scores) "by hand" for bag-of-words retrieval, by directly manipulating the vectors.
3. Be able to perform retrieval "by hand" given a query, by directly manipulating the document vectors materialized from the inverted index.

And putting the two together:

+ Understand how dense retrieval and sparse (bag-of-words) retrieval are different realizations of the same bi-encoder architecture.
+ Be able to connect key concepts in the bi-encoder architecture to Pyserini implementations.
+ Be able to "trace" retrieval with dense and sparse representations through the encoding and top-_k_ retrieval phases.

## Recap

As a recap from [here](conceptual-framework.md), this is the "core retrieval" problem that we're trying to solve:

> Given an information need expressed as a query _q_, the text retrieval task is to return a ranked list of _k_ texts {_d<sub>1</sub>_, _d<sub>2</sub>_ ... _d<sub>k</sub>_} from an arbitrarily large but finite collection
of texts _C_ = {_d<sub>i</sub>_} that maximizes a metric of interest, for example, nDCG, AP, etc.

And this is the bi-encoder architecture for tackling the above challenge:

<img src="images/architecture-biencoder.png" width="400" />

It's all about representations!
BM25 generates bag-of-words sparse lexical vectors where the terms are assigned BM25 weights in an unsupervised manner.
Contriever, which is an example of a dense retrieval model, uses transformer-based encoders, trained on large amounts of supervised data, that generate _dense_ vectors. 

## Dense Retrieval Models

Let's start by first peeking inside the Faiss index we built: 

```python
import faiss

index = faiss.read_index('indexes/faiss.nfcorpus.contriever-msmacro/index')
num_vectors = index.ntotal
```

We see, from `num_vectors`, that there are 3633 vectors in this index.
That's a vector (or alternatively, embedding) for each document.

We can print out each vector:

```python
for i in range(num_vectors):
    vector = index.reconstruct(i)
    print(f"Vector {i}: {vector}")
```

Pyserini stores the `docid` corresponding to each vector separately.
In the code snippet below, we load in the mapping data and then look up the vector corresponding to `MED-4555`.

```python
docids = []
with open('indexes/faiss.nfcorpus.contriever-msmacro/docid', 'r') as fin:
    docids = [line.rstrip() for line in fin.readlines()]

v1 = index.reconstruct(docids.index('MED-4555'))
```

So, `v1` now holds the dense vector representation (i.e., embedding) of document `MED-4555`.

Now, where did this vector come from?
Well, it's the output of the encoder.
Let's verify this by first encoding the contents of the document, which is in `doc_text`:

```python
# This is the string contents of doc MED-4555
doc_text = 'Analysis of risk factors for abdominal aortic aneurysm in a cohort of more than 3 million individuals. BACKGROUND: Abdominal aortic aneurysm (AAA) disease is an insidious condition with an 85% chance of death after rupture. Ultrasound screening can reduce mortality, but its use is advocated only for a limited subset of the population at risk. METHODS: We used data from a retrospective cohort of 3.1 million patients who completed a medical and lifestyle questionnaire and were evaluated by ultrasound imaging for the presence of AAA by Life Line Screening in 2003 to 2008. Risk factors associated with AAA were identified using multivariable logistic regression analysis. RESULTS: We observed a positive association with increasing years of smoking and cigarettes smoked and a negative association with smoking cessation. Excess weight was associated with increased risk, whereas exercise and consumption of nuts, vegetables, and fruits were associated with reduced risk. Blacks, Hispanics, and Asians had lower risk of AAA than whites and Native Americans. Well-known risk factors were reaffirmed, including male gender, age, family history, and cardiovascular disease. A predictive scoring system was created that identifies aneurysms more efficiently than current criteria and includes women, nonsmokers, and individuals aged <65 years. Using this model on national statistics of risk factors prevalence, we estimated 1.1 million AAAs in the United States, of which 569,000 are among women, nonsmokers, and individuals aged <65 years. CONCLUSIONS: Smoking cessation and a healthy lifestyle are associated with lower risk of AAA. We estimated that about half of the patients with AAA disease are not eligible for screening under current guidelines. We have created a high-yield screening algorithm that expands the target population for screening by including at-risk individuals not identified with existing screening criteria.'

from pyserini.encode import AutoDocumentEncoder
encoder = AutoDocumentEncoder('facebook/contriever-msmarco', device='cpu', pooling='mean')

v2 = encoder.encode(doc_text)
```

Minor detail here: the encoder is designed to work on batches of input, so the actual vector representation is `v2[0]`.

We can verify that the vector we generated using the encoder is identical to the vector that is stored in the index by computing the L2 norm (which should be almost zero):

```python
import numpy as np
np.linalg.norm(v2[0] - v1)
```

Let's push this further and work through a query.
Consider the query "How to Help Prevent Abdominal Aortic Aneurysms", which is `PLAIN-3074`.
We can perform interactive retrieval as follows:

```python
from pyserini.search.faiss import FaissSearcher, AutoQueryEncoder

encoder = AutoQueryEncoder('facebook/contriever-msmarco', device='cpu', pooling='mean')
searcher = FaissSearcher('indexes/faiss.nfcorpus.contriever-msmacro', encoder)
hits = searcher.search('How to Help Prevent Abdominal Aortic Aneurysms')

for i in range(0, 10):
    print(f'{i+1:2} {hits[i].docid:7} {hits[i].score:.6f}')
```

And the result will be:

```
 1 MED-4555 1.472201
 2 MED-3180 1.125014
 3 MED-1309 1.067153
 4 MED-2224 1.059536
 5 MED-4423 1.038440
 6 MED-4887 1.032622
 7 MED-2530 1.020758
 8 MED-2372 1.016142
 9 MED-1006 1.013599
10 MED-2587 1.010811
```

Let's go ahead and encode the query, producing the query vector `q_vec`:

```python
from pyserini.encode import AutoQueryEncoder

q_encoder = AutoQueryEncoder('facebook/contriever-msmarco', device='cpu', pooling='mean')
q_vec = q_encoder.encode('How to Help Prevent Abdominal Aortic Aneurysms')
```

Then, we compute the dot product between the query vector `q_vec` and the document vector `v1` (which is the representation of document `MED-4555` generated by the document encoder):

```python
np.dot(q_vec, v1)
```

We should arrive at the same score as above (`1.472201`).
In other words, the query-document score (i.e., the relevance score of the document with respect to the query) is exactly the dot product of the two vector representations.
This is as expected!

We can take this a step further and manually perform retrieval by computing the dot product between the query vector and _all_ document vectors.
The corpus is small enough that this is practical:

```python
from tqdm import tqdm

scores = []
# Iterate through all document vectors and compute dot product.
for i in tqdm(range(num_vectors)):
    vector = index.reconstruct(i)
    score = np.dot(q_vec, vector)
    scores.append([docids[i], score])

# Sort by score descending.
scores.sort(key=lambda x: -x[1])

for s in scores[:10]:
    print(f'{s[0]} {s[1]:.6f}')
```

In a bit more detail, we iterate through all document vectors in the index, compute its dot product with the query vector, and append the results in `scores`.
After going through the entire corpus in this manner, we sort the results and print out the top-10.
This sorting operation corresponds to top-_k_ retrieval.

We can see that the output is the same as search with `FaissSearcher` above.
This is exactly as expected.

## Sparse Retrieval Models

Now, we're going to basically do the same thing, but with BM25.
The point here is to illustrate how dense and sparse retrieval are conceptually identical &mdash; they're both instantiations of the bi-encoder architecture.
The primary difference is the encoder representation, i.e., the vectors that the encoders generate.

We have to start with a bit of data munging, since the Lucene indexer expects the documents in a slightly different format.
Start by creating a new sub-directory:

```bash
mkdir collections/nfcorpus/pyserini-corpus
```

Now run the following Python script to munge the data into the right format:

```python
import json

with open('collections/nfcorpus/pyserini-corpus/corpus.jsonl', 'w') as out:
    with open('collections/nfcorpus/corpus.jsonl', 'r') as f:
        for line in f:
            l = json.loads(line)
            s = json.dumps({'id': l['_id'], 'contents': l['title'] + ' ' + l['text']})
            out.write(s + '\n')
```

We can now index these documents as a `JsonCollection` using Pyserini:

```bash
python -m pyserini.index.lucene \
  --collection JsonCollection \
  --input collections/nfcorpus/pyserini-corpus/ \
  --index indexes/lucene-index.nfcorpus \
  --generator DefaultLuceneDocumentGenerator \
  --storePositions --storeDocvectors --storeRaw
```

Perform retrieval:

```bash
python -m pyserini.search.lucene \
  --index indexes/lucene-index.nfcorpus \
  --topics collections/nfcorpus/queries.tsv \
  --output runs/run.beir-bm25.nfcorpus.txt \
  --hits 1000 --bm25 \
  --threads 4 --batch-size 16
```

And evaluate the retrieval run:

```bash
python -m pyserini.eval.trec_eval \
  -c -m ndcg_cut.10 collections/nfcorpus/qrels/test.qrels \
  runs/run.beir-bm25.nfcorpus.txt
```

The expected results are:

```
Results:
ndcg_cut_10           	all	0.3218
```

We can also perform retrieval interactively:

```python
from pyserini.search.lucene import LuceneSearcher

searcher = LuceneSearcher('indexes/lucene-index.nfcorpus')
hits = searcher.search('How to Help Prevent Abdominal Aortic Aneurysms')

for i in range(0, 10):
    print(f'{i+1:2} {hits[i].docid:7} {hits[i].score:.4f}')
```

The results should be as follows:

```
 1 MED-4555 11.9305
 2 MED-4423 8.4771
 3 MED-3180 7.1896
 4 MED-2718 6.0102
 5 MED-1309 5.8181
 6 MED-4424 5.7448
 7 MED-1705 5.6101
 8 MED-4902 5.3639
 9 MED-1009 5.2533
10 MED-1512 5.2068
```

So far, none of this is new:
We did exactly the same thing for [the MS MARCO passage ranking test collection](experiments-msmarco-passage.md), but now we're doing it for NFCorpus.

Next, let's generate the BM25 document vector for doc `MED-4555`, the same document we examined above.

```python
from pyserini.index.lucene import IndexReader
import json

index_reader = IndexReader('indexes/lucene-index.nfcorpus')
tf = index_reader.get_document_vector('MED-4555')
bm25_weights = \
    {term: index_reader.compute_bm25_term_weight('MED-4555', term, analyzer=None) \
     for term in tf.keys()}

print(json.dumps(bm25_weights, indent=4, sort_keys=True))
```

The variable `bm25_weights` is a Python dictionary holding the BM25 weights for the document.

We're going to now perform retrieval "by hand" with BM25, similar to what we did above with the dense retrieval model.
Let's start by encoding the query, which is a multi-hot vector where the non-zero items correspond to the query terms:

```python
from pyserini.analysis import Analyzer, get_lucene_analyzer

analyzer = Analyzer(get_lucene_analyzer())
query_tokens = analyzer.analyze('How to Help Prevent Abdominal Aortic Aneurysms')
multihot_query_weights = {k: 1 for k in query_tokens}
```

The variable `multihot_query_weights` is a Python dictionary where the keys correspond to the query tokens, each with a value of one.

Now let's compute the dot product of the two vectors.

```python
sum({term: bm25_weights[term] \
     for term in bm25_weights.keys() & \
     multihot_query_weights.keys()}.values())
```

The dot product is `11.9305`.

Again, this isn't anything new.
We did all of this in the [conceptual framework guide](conceptual-framework.md) with MS MARCO passage; we're just now doing it on NFCorpus.

The above expression for computing a dot product &mdash; let's wrap in a Python function, and then verify it gives the same output:

```python
def dot(q_weights, d_weights):
    return sum({term: d_weights[term] \
                for term in d_weights.keys() & \
                q_weights.keys()}.values())

dot(multihot_query_weights, bm25_weights)
```

With this setup, we can now perform end-to-end retrieval for a query "by hand", by directly manipulating the index structures:

```python
from pyserini.search.lucene import LuceneSearcher
from pyserini.index.lucene import IndexReader
from tqdm import tqdm

searcher = LuceneSearcher('indexes/lucene-index.nfcorpus')
index_reader = IndexReader('indexes/lucene-index.nfcorpus')

scores = []
# Iterate through all docids in the index.
for i in tqdm(range(0, searcher.num_docs)):
    docid = searcher.doc(i).get('id')
    # Reconstruct the BM25 document vector.
    tf = index_reader.get_document_vector(docid)
    bm25_weights = \
        {term: index_reader.compute_bm25_term_weight(docid, term, analyzer=None) \
         for term in tf.keys()}
    # Compute and retain the query-document score.
    score = dot(multihot_query_weights, bm25_weights)
    scores.append([docid, score])

# Sort by score descending.
scores.sort(key=lambda x: -x[1])

for s in scores[:10]:
    print(f'{s[0]} {s[1]:.4f}')
```

The code snippet above should be self-explanatory.
We iterate through all documents, reconstruct the BM25 document vectors (as weights in a Python dictionary), compute the dot product with the query vector, and retain the scores.
Once we've gone through all documents in the corpus in this manner, we sort the scores and print out the top-_k_.

The output should match the results from `LuceneSearcher` above.

To recap, what's the point for this exercise?

+ We see that dense retrieval and sparse retrieval are both instantiations of a bi-encoder architecture. The only difference is the output of the encoder representations.
+ For both a dense index (Faiss) and a sparse index (Lucene), you now know how to reconstruct the document vector representations.
+ For both a dense retrieval model and a sparse retrieval model, you now know how to encode a query into a query vector.
+ For both a dense retrieval model and a sparse retrieval model, you know how to compute query-document scores: they're just dot products.
+ Finally, for both a dense retrieval model and a sparse retrieval model, you can perform retrieval "by hand". This can be accomplished by iterating through all document vectors in the index and computing its dot product with the query vector in a brute force manner. By sorting the scores, you're performing top-_k_ retrieval, which gives exactly the same output as `FaissSearcher` and `LuceneSearcher` (although not as efficient).

Okay, that's it for this lesson.
Before you move on, however, add an entry in the "Reproduction Log" at the bottom of this page, following the same format: use `yyyy-mm-dd`, make sure you're using a commit id that's on the main trunk of Pyserini, and use its 7-hexadecimal prefix for the link anchor text.

## Reproduction Log[*](reproducibility.md)

+ Results reproduced by [@sahel-sh](https://github.com/sahel-sh) on 2023-08-07 (commit [`9dab30f`](https://github.com/castorini/pyserini/commit/9dab30f1ac2b7672ffc65477f0d4279d30e97ad4))
+ Results reproduced by [@Andrwyl](https://github.com/Andrwyl) on 2023-08-26 (commit [`d9da49e`](https://github.com/castorini/pyserini/commit/d9da49eb3a23fb9daa26399a2e27a5efc73beb71))
+ Results reproduced by [@yilinjz](https://github.com/yilinjz) on 2023-08-30 (commit [`42b3549`](https://github.com/castorini/pyserini/commit/42b354914b230880c91b2e4e70605b472441a9a1))
+ Results reproduced by [@UShivani3](https://github.com/UShivani3) on 2023-09-02 (commit [`42b3549`](https://github.com/castorini/pyserini/commit/42b354914b230880c91b2e4e70605b472441a9a1))
+ Results reproduced by [@Edward-J-Xu](https://github.com/Edward-J-Xu) on 2023-09-05 (commit [`8063322`](https://github.com/castorini/pyserini/commit/806332286d6eacea23061c04205a71698e6a6208))
+ Results reproduced by [@mchlp](https://github.com/mchlp) on 2023-09-09 (commit [`d8dc5b3`](https://github.com/castorini/pyserini/commit/d8dc5b3a1f32fd5d0cebeb711ba148ea967fadbe))
+ Results reproduced by [@lucedes27](https://github.com/lucedes27) on 2023-09-10 (commit [`54014af`](https://github.com/castorini/pyserini/commit/54014af8fe4bf4ba75daba9119acac94c7191cdb))
+ Results reproduced by [@MojTabaa4](https://github.com/MojTabaa4) on 2023-09-14 (commit [`d4a829d`](https://github.com/castorini/pyserini/commit/d4a829d18043783ef3dec2a8adce50e4061ba99a))
+ Results reproduced by [@Kshama](https://github.com/Kshama33) on 2023-09-24 (commit [`7d18f4b`](https://github.com/castorini/pyserini/commit/7d18f4bd3f98d4f901dc061ffd93a1c656e32d0d))
+ Results reproduced by [@MelvinMo](https://github.com/MelvinMo) on 2023-09-24 (commit [`7d18f4b`](https://github.com/castorini/pyserini/commit/7d18f4bd3f98d4f901dc061ffd93a1c656e32d0d))
+ Results reproduced by [@ksunisth](https://github.com/ksunisth) on 2023-09-28 (commit [`142c774`](https://github.com/castorini/pyserini/commit/142c774a303c906ee245913bc7e714b165074b77))
+ Results reproduced by [@maizerrr](https://github.com/maizerrr) on 2023-10-01 (commit [`bdb9504`](https://github.com/castorini/pyserini/commit/bdb9504b1757ab88247924b55a8fde3e5c1a3d20))
+ Results reproduced by [@Mofetoluwa](https://github.com/Mofetoluwa) on 2023-10-02 (commit [`88f1f5b`](https://github.com/castorini/pyserini/commit/88f1f5b653021e249f45bb85c3297bb6af862c3d))
+ Results reproduced by [@gituserbs](https://github.com/gituserbs) on 2023-10-19 (commit [`e0a0d35`](https://github.com/castorini/pyserini/commit/e0a0d354ccbd055b42413b1eed911858d68a01fc))
