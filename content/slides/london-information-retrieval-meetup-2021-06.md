# Lucene-Grep a.k.a. `lmgrep`

---

# `whoami`

```json no-exec id=ecb5068e-2d50-4696-ba83-755ef6d198c1
{
  "name": "Dainius Jocas",
  "company": {
    "name": "Vinted",
    "mission": "Make second-hand the first choice worldwide"
  },
  "role": "Staff Engineer",
  "website": "https://www.jocas.lt",
  "twitter": "@dainius_jocas",
  "github": "dainiusjocas",
  "author_of_oss": ["lucene-grep", "ket"]
}
```

---

# Agenda

1. Intro
2. Whats inside Lucene-Grep?
3. Use cases
4. Future work
5. Discussion

---

# Intro

* **`lmgrep`** is a CLI full-text search tool
* Interface is similar to **grep**
* Based on **Lucene**
* Lucene Monitor library is the main building block
* Compiled with the GraalVM **`native-image`**
* Single binary file, no external dependencies
* Supports Linux, MacOS, Windows

--- 

## Origin

* Used **Elasticsearch Percolator** for some basic named entity recognition (NER)
* Needed to deploy to AWS Lambda, Elasticsearch was not an option
* However, I really liked the idea of expressing entities as full-text queries
* Found the **Luwak** library, deployed on AWS Lambda, however it ran on JVM
* **Gunnar Morling** blog post about GraalVM native-image Lucene on AWS Lambda
* Convinced Red Hat devs to open source and release **quarkiverse/quarkus-lucene**
* Hacked Lucene Grep

---

## `grep` vs `lmgrep`

```bash id=6011bd03-c09a-46f6-b664-3b0fd31946e4
 echo "Lucene is awesome" | grep Lucene
```

```bash id=9ac214c9-b5e9-47b9-8c86-238756e4b33b
 echo "Lucene is awesome" | lmgrep Lucene
```

---

## Installing the `lmgrep`

**`brew`** or a shell script on Linux

```bash id=0d2cde56-d973-4049-ba70-25f56c4c1df7
wget https://github.com/dainiusjocas/lucene-grep/releases/download/v2021.05.23/lmgrep-v2021.05.23-linux-static-amd64.zip
unzip lmgrep-v2021.05.23-linux-static-amd64.zip
mv lmgrep /usr/local/bin
```

**`brew`** on MacOS

```bash id=00d83f4f-db54-4f94-9c55-c188c0f11cb7
brew install dainiusjocas/brew/lmgrep
```

**`scoop`** on Windows

```bash id=af8eba1d-7df2-4260-b96e-ddb022900283
scoop bucket add scoop-clojure https://github.com/littleli/scoop-clojure
scoop bucket add extras
scoop install lmgrep
```

---

# Whats inside?

* Reading from file(s)
* Searching for files with **GLOB, e.g. '**`**/*.txt`**'**
* Reading from **STDIN**
* Writing to **STDOUT** in various formats, e.g. JSON
* Text analysis pipeline
* Multiple query parsers
* Text tokenization with **`--only-analyze`**flag
* Loading multiple queries from a file
* Full-text search
* **`lmgrep -h`** for the full list of available options

---

# Text Analysis

* The same good ol' **`lucene`** text analysis
* 45 predefined analyzers available, e.g. **`LithuanianAnalyzer`**
* 5 character filters
* 14 tokenizers
* 113 token filters
* However, not everything that Lucene provides is available in **`lmgrep`** because of limitations of the **GraalVM native-image**
* <https://github.com/dainiusjocas/lucene-grep/blob/main/docs/analysis-components.md>

---

## Custom Text Analysis Issue

* At first exposed several CLI flags for text analysis
  * a problem with order of execution
* Lucene analyzers are Java classes
* For a CLI tool, exposing Java classes is not a good option
* Something similar to Elasticsearch analysis syntax is needed

---

## Text Analysis Definition

```json no-exec id=ed867748-ecb1-4528-ab4c-05fc6a442561
{
    "char-filters": [
      {"name": "htmlStrip"},
      {
        "name": "patternReplace",
        "args": {
          "pattern": "foo",
          "replacement": "bar"
        }
      }
    ],
    "tokenizer": {"name": "standard"},
    "token-filters": [
      {"name": "englishMinimalStem"},
      {"name": "uppercase"}
    ]
  }
```

---

# Various Query Parsers `--query-parser`

---

## `--query-parser`=classic

* The default one
* When googling for the `Lucene query syntax`, the first hit

```bash id=17899a72-33de-491b-901d-e84dc92c9ab9
echo "Lucene is awesome" | lmgrep --query-parser=classic "lucene is aweso~"
```

```bash id=2342aefa-edd9-4274-8aee-fa229d27e5eb
echo "Lucene is awesome" | lmgrep --query-parser=classic "\"lucene is\""
```

---

## `--query-parser=`complex-phrase

* similar to the **`classic`** query parser
* but phrase queries are more expressive

```bash id=b40c55d8-05c3-4a0e-bf77-6f65b8147318
echo "jonathann jon peterson" | lmgrep --query-parser=complex-phrase "\"(john jon jonathan~) peters*\""
```

---

## `--query-parser=`simple

* similar to the **`classic`** query parser
* **BUT** any errors in the query syntax will be ignored and the parser will attempt to decipher what it can
* E.g. given `term1\* `searches for the term `term1*`
* Probably should be the default query parser in **`lmgrep`**

---

## `--query-parser=`standard

* Implementation of the [Lucene classic query parser](https://javadoc.io/static/org.apache.lucene/lucene-queryparser/8.9.0/org/apache/lucene/queryparser/classic/package-summary.html) using the flexible query parser frameworks
* There must be a reason why it comes with the default **`lucene`** dependency

---

## `--query-parser=`surround

* Constructs span queries that use positional information

```bash id=09fccaa9-72f6-490d-8e2d-df19048a611d
echo "Lucene is awesome" | lmgrep --query-parser=surround "2W(lucene, awesome)"
```

* if the term order is **NOT** important: **W->N**

```bash id=f9c1c889-6a12-4a11-8781-319ee9b69a38
echo "Lucene is awesome" | lmgrep --query-parser=surround "2N(awesome, lucene)"
```

* WARNING: query terms are not analyzed

---

# `--only-analyze`

* Just **apply** the **text analyzer** on the **input text** and **output the list(s) of tokens**

---

## `--only-analyze`: basic example

```bash id=d0d84e54-a3b0-4437-8090-b8886b9585ea
echo "Lucene is awesome" | lmgrep --only-analyze
```

---

## **`--only-analyze`**: custom text analysis pipeline

```bash id=cd1d1a54-be1e-4a32-baa4-4d6495c13572
echo "<p>foo bars baz</p>" | lmgrep --only-analyze --analysis='
  {
    "char-filters": [
      {"name": "htmlStrip"},
      {
        "name": "patternReplace",
         "args": {
           "pattern": "foo",
           "replacement": "bar"
        }
      }
    ],
    "tokenizer": {"name": "standard"},
    "token-filters": [
      {"name": "englishMinimalStem"},
      {"name": "uppercase"}
    ]
  }
  '
```

```json no-exec id=3109d090-1872-4788-ae47-10aed96fd6d7
["BAR","BAR","BAZ"]
```

---

## `--only-analyze` with `--explain`

```bash id=30c23b70-7b03-4ed3-8fe4-c3d7aac6c7fa
echo "Dogs and CAt" | lmgrep --only-analyze --explain | jq
```

```json no-exec id=4181e7ac-8b90-422d-b65e-de8ecb110259
[
  {
    "token": "dog",
    "position": 0,
    "positionLength": 1,
    "type": "<ALPHANUM>",
    "end_offset": 4,
    "start_offset": 0
  },
  {
    "end_offset": 8,
    "positionLength": 1,
    "position": 1,
    "start_offset": 5,
    "type": "<ALPHANUM>",
    "token": "and"
  },
  {
    "position": 2,
    "token": "cat",
    "positionLength": 1,
    "end_offset": 12,
    "type": "<ALPHANUM>",
    "start_offset": 9
  }
]
```

* The idea is similar to the Elasticsearch's `_analyze` API
* No need to recreate an index on every custom analyzer change 

---

## `--only-analyze`: output for graphviz 

* **TODO**

![tokensSyns.png][nextjournal#file#9e3a426c-5354-4378-9cd9-a079f7760c23]


---

# Loading queries from a file

```bash no-exec id=400756d4-7904-4fbb-916f-6db143d4fd0f
echo "I have two dogs" | lmgrep --queries-file=dog-lovers.json
```

```json no-exec id=fa294f41-8259-4938-a8ed-a2a12a6cb4cf
[
  {
    "id": "german_language",
    "query": "hund",
    "stemmer": "german"
  },
  {
    "id": "english_language",
    "query": "dog",
    "stemmer": "english"
  }
]
```

* load all queries once
* **100K** queries takes about **1s** to load on my laptop

---

# Full-text search

```bash id=790cefed-de1e-49fa-85b9-a0151bb9cc6b
mkdir demo
cd demo
echo "Lucene is awesome" > lucene.txt
echo "Grep is awesome" > grep.txt
lmgrep lucene **.txt
```

---

# Full-text File Search with Score

```bash id=3da7f5f5-c87b-4ccf-84d2-0486291a2a4d
cd
mkdir full-text-search || true
cd full-text-search

echo "Lucene is awesome" > lucene.txt
echo "Lucene Grep is build on Lucene Monitor library" > lucene-grep.txt

lmgrep "Lucene" '**.txt' --no-split --with-score --format=json | jq -s -c 'sort_by(.score)[]' | tac | head -3 | jq

```

---

# Source Code Search

* Specify a custom analyzer for you programming language
* E.g. **WordDelimiterGraphFilter** that **"MyFooClass" => \["My", "Foo", "Class"\]**
* Enable scoring
* Output hyperlinks in a (supported) terminal emulator to the specific line number

---

# Alternative to Elasticsearch Percolator

* Start a **`lmgrep`** with open **STDIN**, **STDOUT**, and **STDERR** pipes for inter-process communication

```ruby no-exec id=51a22eb8-4ffd-4e47-8f0e-05ffd18eee2a
require 'open3'

@stdin, @stdout, @stderr, @wait_thr = Open3.popen3("lmgrep lucene")

@stdin.puts "Lucene is awesome"

@stdout.gets
```

* <https://github.com/dainiusjocas/lucene-grep/tree/main/examples/ruby-percolator>

---

# Future work

* Your issues <https://github.com/dainiusjocas/lucene-grep/issues>
* Machanism for shared analysis components
  * now only inlined text analysis config is supported
* LMGREP_HOME for keeping all the resources in one place
* Release analyzer construction code as a standalone library
* Melt your CPU
  * Use all CPU cores to the max for as short as possible
  * Do not preserve the input order
* Optimize **`--with-scored-highlights`** option
  * Sort output by score
* Analysis components with inlined data
  * E.g. inlines stopwords list, not a file

---

# Discussion

[nextjournal#file#9e3a426c-5354-4378-9cd9-a079f7760c23]:
<https://nextjournal.com/data/QmW9eMZvaoJZhgykEbhMEGmRqAq9Qx4GrKcTa8Kfwmj1Zk?content-type=image/png&node-id=9e3a426c-5354-4378-9cd9-a079f7760c23&filename=tokensSyns.png&node-kind=file> (<p>Token graph</p>)

<details id="com.nextjournal.article">
<summary>This notebook was exported from <a href="https://nextjournal.com/a/P3v43aPLhVdSZ3BS8NaX3?change-id=CxhWFVu6LhGvq85956D1fu">https://nextjournal.com/a/P3v43aPLhVdSZ3BS8NaX3?change-id=CxhWFVu6LhGvq85956D1fu</a></summary>

```edn nextjournal-metadata
{:article
 {:settings {:numbered? false},
  :nodes
  {"00d83f4f-db54-4f94-9c55-c188c0f11cb7"
   {:id "00d83f4f-db54-4f94-9c55-c188c0f11cb7",
    :kind "code",
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "09fccaa9-72f6-490d-8e2d-df19048a611d"
   {:compute-ref #uuid "f5c9e0e1-e2b7-472f-8af7-c54f043dcb35",
    :exec-duration 758,
    :id "09fccaa9-72f6-490d-8e2d-df19048a611d",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "0d2cde56-d973-4049-ba70-25f56c4c1df7"
   {:compute-ref #uuid "a22031b2-8c74-49f3-b240-f57854eeb5f2",
    :exec-duration 4615,
    :id "0d2cde56-d973-4049-ba70-25f56c4c1df7",
    :kind "code",
    :output-log-lines {:stdout 436},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"],
    :stdout-collapsed? true},
   "17899a72-33de-491b-901d-e84dc92c9ab9"
   {:compute-ref #uuid "e3f64704-22b2-41b6-9e86-2a50792e62e8",
    :exec-duration 881,
    :id "17899a72-33de-491b-901d-e84dc92c9ab9",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "2342aefa-edd9-4274-8aee-fa229d27e5eb"
   {:compute-ref #uuid "68f5460c-5c3d-422a-9a65-2ea459c4566c",
    :exec-duration 901,
    :id "2342aefa-edd9-4274-8aee-fa229d27e5eb",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "30c23b70-7b03-4ed3-8fe4-c3d7aac6c7fa"
   {:compute-ref #uuid "12efa625-d503-4984-8bd6-79536e1d2c1c",
    :exec-duration 799,
    :id "30c23b70-7b03-4ed3-8fe4-c3d7aac6c7fa",
    :kind "code",
    :output-log-lines {:stdout 27},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"],
    :stdout-collapsed? true},
   "3109d090-1872-4788-ae47-10aed96fd6d7"
   {:id "3109d090-1872-4788-ae47-10aed96fd6d7", :kind "code-listing"},
   "3da7f5f5-c87b-4ccf-84d2-0486291a2a4d"
   {:compute-ref #uuid "75ece01b-2eff-4ce7-aa94-7a213cd6cbf8",
    :exec-duration 974,
    :id "3da7f5f5-c87b-4ccf-84d2-0486291a2a4d",
    :kind "code",
    :output-log-lines {:stdout 14},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "400756d4-7904-4fbb-916f-6db143d4fd0f"
   {:id "400756d4-7904-4fbb-916f-6db143d4fd0f", :kind "code-listing"},
   "4181e7ac-8b90-422d-b65e-de8ecb110259"
   {:id "4181e7ac-8b90-422d-b65e-de8ecb110259", :kind "code-listing"},
   "4534e627-a3df-4694-97c9-39b3fbbe9f90"
   {:environment
    [:environment
     {:article/nextjournal.id
      #uuid "5b45dad0-dfdf-4576-9b8c-f90892e74c94",
      :change/nextjournal.id
      #uuid "5df5da3f-c83a-4296-bc41-0e6e394499d4",
      :node/id "dab15041-47f1-4ca7-84e2-b4532a4a2f70"}],
    :id "4534e627-a3df-4694-97c9-39b3fbbe9f90",
    :kind "runtime",
    :language "bash",
    :type :nextjournal},
   "51a22eb8-4ffd-4e47-8f0e-05ffd18eee2a"
   {:id "51a22eb8-4ffd-4e47-8f0e-05ffd18eee2a", :kind "code-listing"},
   "6011bd03-c09a-46f6-b664-3b0fd31946e4"
   {:compute-ref #uuid "1db28be9-449b-417d-b8e4-0b5457056aff",
    :exec-duration 1040,
    :id "6011bd03-c09a-46f6-b664-3b0fd31946e4",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "790cefed-de1e-49fa-85b9-a0151bb9cc6b"
   {:compute-ref #uuid "a2a52cb4-c860-4b68-b2f9-ff45e568d2f1",
    :exec-duration 1069,
    :id "790cefed-de1e-49fa-85b9-a0151bb9cc6b",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "9ac214c9-b5e9-47b9-8c86-238756e4b33b"
   {:compute-ref #uuid "d79de0c9-05d6-44d9-9bd8-09e35d3216b8",
    :exec-duration 992,
    :id "9ac214c9-b5e9-47b9-8c86-238756e4b33b",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "9e3a426c-5354-4378-9cd9-a079f7760c23"
   {:id "9e3a426c-5354-4378-9cd9-a079f7760c23", :kind "file"},
   "af8eba1d-7df2-4260-b96e-ddb022900283"
   {:compute-ref #uuid "27541092-284e-4136-afd0-046259eb264c",
    :exec-duration 2964,
    :id "af8eba1d-7df2-4260-b96e-ddb022900283",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"],
    :stdout-collapsed? true},
   "b40c55d8-05c3-4a0e-bf77-6f65b8147318"
   {:compute-ref #uuid "df503be9-fc5a-478e-bf3f-623015c2bf4c",
    :exec-duration 1129,
    :id "b40c55d8-05c3-4a0e-bf77-6f65b8147318",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "cd1d1a54-be1e-4a32-baa4-4d6495c13572"
   {:compute-ref #uuid "ef429098-d2fe-4484-a681-fdefabacec7f",
    :exec-duration 980,
    :id "cd1d1a54-be1e-4a32-baa4-4d6495c13572",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"],
    :stdout-collapsed? true},
   "d0d84e54-a3b0-4437-8090-b8886b9585ea"
   {:compute-ref #uuid "d6e6fc75-6f70-4443-8a70-09304cb6180d",
    :exec-duration 1100,
    :id "d0d84e54-a3b0-4437-8090-b8886b9585ea",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "ecb5068e-2d50-4696-ba83-755ef6d198c1"
   {:id "ecb5068e-2d50-4696-ba83-755ef6d198c1", :kind "code-listing"},
   "ed867748-ecb1-4528-ab4c-05fc6a442561"
   {:id "ed867748-ecb1-4528-ab4c-05fc6a442561", :kind "code-listing"},
   "f9c1c889-6a12-4a11-8781-319ee9b69a38"
   {:compute-ref #uuid "f3f310ba-317e-456b-802b-3b089c2553a5",
    :exec-duration 811,
    :id "f9c1c889-6a12-4a11-8781-319ee9b69a38",
    :kind "code",
    :output-log-lines {:stdout 2},
    :runtime [:runtime "4534e627-a3df-4694-97c9-39b3fbbe9f90"]},
   "fa294f41-8259-4938-a8ed-a2a12a6cb4cf"
   {:id "fa294f41-8259-4938-a8ed-a2a12a6cb4cf", :kind "code-listing"}},
  :nextjournal/id #uuid "031428c0-72a8-44c8-a5aa-10fba8f8e34a",
  :article/change
  {:nextjournal/id #uuid "60db64ef-6234-4eef-bec7-a9f972100980"}}}

```
</details>
