# WN-MCR-Transform

Transform the MCR 3.0 database so that it can be loaded using the NLTK WordNet reader.

This file has a quick user guide to transform and use the MCR 3.0 files, as well as a description of the overall transformation process and its limitations.

This software was created by Grupo de Procesamiento de Lenguaje Natural from Instituto de Computación, Facultad de Ingeniería, Universidad de la República, Uruguay. Any questions or comments can be addressed at Luis Chiruzzo (luischir@fing.edu.uy).

## Transforming the MCR 3.0 corpus:

The compressed files can be downloaded directly from here, but if you prefer to generate them, do the following.

Download the needed files:

* [The Multilingual Central Repository 3.0 corpus](http://adimen.si.ehu.es/web/MCR/)
* [The NLTK toolkit](http://www.nltk.org/)
* The WordNet corpus for NLTK, which can be downloaded using `nltk.download()` after NLTK is installed.

Then:

1. Find your WordNet 3.0 database files. They can be typically found in ~/nltk_data/corpora/wordnet, but the location of these files might vary in your installation. We will refer to the WordNet 3.0 database folder as WORDNET_EN_ROOT. We will refer to folder you just created, where the transformed files will be, as RESULT_ROOT.
2. Extract the MCR 3.0 files to a folder. This will create a series of folders containing WordNet versions for different languages. We will refer to this folder as MCR_ROOT.
3. Run the command:

```shell
$ ./generate_all.sh <path to MCR_ROOT> <path to WORDNET_EN_ROOT>
```

After step 3, the data.* and index.* files contained in RESULT_ROOT will be replaced with new versions containing the MCR 3.0 information for the desired language.

The available languages so far and their codes are:

* Catalan - cat
* English - eng
* Euskara - eus
* Galician - glg
* Spanish - spa

## Using the transformed MCR 3.0 corpus:

```python
>>> import nltk
>>> wncr = nltk.corpus.reader.wordnet.WordNetCorpusReader(<path to RESULT_ROOT>)
>>> print(wncr.synset("entidad.n.01").definition)
```

## Exporting and importing the glosses:

MCR is a work in progress and not all the contents have been fully translated. This is specially true about glosses, for example only around 15% of the glosses in the Spanish version have been translated. For some applications this might be an issue.
However, if you have another source where you can get the glosses in your language (for example using a machine translation process) you can import that data so it can be merged with the MCR 3.0 during the transformation.

1. In a Python shell, import the transform module
2. Execute the following command:
```python
>>> transform.export_glosses(<path to WORDNET_EN_ROOT>, <path to EN_GLOSSES_FILE>)
```
This creates the file EN_GLOSSES_FILE, which will contain the English glosses for all synsets. The file format is straightforward. Each line contains the gloss for one synset in this format: <id> | <gloss>, where <id> is a concatenation of the offset in the WordNet 3.0 database file and the part of speech of the synset. For example, the synset corresponding to "entity" in English has this line: 00001740n | that which is perceived...
3. Translate the glosses using any means you can. As long as the format is honored and the identifiers are kept, the process will be able to get the translated glosses and merge them with the rest of the data. We will assume that you have created a new file TRANSLATED_GLOSSES_FILE containing the translated glosses.
4. Execute the following command:
```shell
$ python3 transform.py <path to MCR_ROOT> <path to WORDNET_EN_ROOT> <LANGUAGE> <path to RESULT_ROOT> <path to TRANSLATED_GLOSSES_FILE>
```

## Description of the process and limitations:

The transformation process is straightforward. The synsets are loaded, variants and relations files from MCR. Then that information is used to create data and index files respecting the constraints of the WordNet database files.

In particular, we must respect the constraint that the synset with numeric id XX must start in the offset XX of the data file. This is mandatory in order to use the NLTK WordNet reader, as it uses this fact to speed up the queries. However, this has an unfortunate consequence: the numeric id of the English and the transformed MCR files do not match. For example, the synset for "dog.n.01" has the offset 02084071 but the synset for "perro.n.01" has the offset 01295142 in the Spanish MCR. This also means that it is no longer possible to match "perro.n.01" back to its MCR identifier spa-30-02084071-n, which points to the English file offset.

The definition of relations in MCR and WordNet differ. In MCR there are more relations defined, and it is not always easy to know which corresponds to which. However, the most usual relations (such as hyponym/hypernym and meronym/holonym) have been correctly mapped. The mapping of MCR relations that are transformed to WordNet relations follows (note that some of the mappings could be wrong and change in the future):

MCR                                 WordNet

| Id | Name LR | Name RL | LR | Name LR | RL | Name RL |
| -- | ------- | ------- | -- | ------- | -- | ------- |
| 1 | be_in_state | state_of | = | Attribute | = | Attribute |
| 2 | causes | is_caused_by | > | cause | - | - |
| 4 | has_derived | is_derived_from | \ | derived from adjective | - | - |
| 6 | has_holo_madeof | has_mero_madeof | #s | substance holonym | %s | substance meronym |
| 7 | has_holo_member | has_mero_member | #m | member holonym | %m | member meronym |
| 8 | has_holo_part | has_mero_part | #p | part holonym | %p | part meronym |
| 12 | has_hyponym | has_hyperonym | ~ | hyponym | @ | hypernym |
| 19 | has_subevent | is_subevent_of | * | entailment | - | - |
| 33 | near_antonym | - | ! | antonym | ! | antonym |
| 34 | near_synonym | - | & | similar | & | similar |
| 49 | see_also_wn15 | - | ^ | also see | - | - |
| 52 | verb_group | - | $ | verb group | $ | verb group |
| 63 | category_term | category | -c | member - topic | ;c | domain - region |
| 64 | related_to | - | + | deriv. related form | + | deriv. related form |
| 66 | region_term | region | -r | member - region | ;r | domain - region |
| 68 | usage_term | usage | -u | member - usage | ;u | domain - usage |

There is a difference between the way hypernyms are defined in MCR and WordNet. Also, in the original WordNet the antonym relation holds between two lemmas (the NLTK corpus reader browses the antonyms this way), while in MCR the relation is between synsets. Because of this, we consider that an antonym relation between synsets S1 and S2 in MCR will correspond, in the transformed version, to a set of antonym relations between lemmas L1 and L2, for all L1 in S1 and all L2 in S2.