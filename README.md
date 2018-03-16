## Hoot

Hoot is an [OWL Ontology]() in RDF/XML format that describes English [phonetic](https://en.wikipedia.org/wiki/Phonetics) information as a domain of knowledge. Each word of English is present as an instance of the OWL class `Word`; each `Word` has a unique identifier (`IRI`), spelling (`rdfs:label`), and phonetic transcription (`hoot:asIPA`). There is also a `Phone` class, which identifies each attested phone in the English language. The English words and their phonetic transcriptions come from the [CMU Pronouncing Dictionary](http://www.speech.cs.cmu.edu/cgi-bin/cmudict).

There are a number of implications for a graph-model-defined phonetic dictionary, and one of them is that we can leverage the [SPARQL](https://www.w3.org/TR/sparql11-query/) RDF query language to find interesting phonetic facts and relationships. 

The following demonstrates just a few example SPARQL queries of Hoot: 

* View all phonetic symbols that represent vowels
```sparql
PREFIX hoot: <http://mphilli.github.io/hoot#>
SELECT (strafter(str(?phone), str(hoot:)) as ?vowels) {
      ?phone hoot:hasPhoneType hoot:vowel.
  }
```
vowels|
--- |
æ |
aɪ |
aʊ |
ɑ |
e |
ə |
ər |
ɛ |
i |
ɪ |
oʊ |
ɔ |
ɔɪ |
u |
ʊ |

* Find a word, its IRI, and its phonetic transcription
```SPARQL
PREFIX hoot: <http://mphilli.github.io/hoot#>

SELECT ?word ?IRI ?phones WHERE {
      ?IRI rdfs:label ?word ; 
             hoot:asIPA ?phones .
        FILTER regex(str(?word), "^hoot$")
  }
```
word | IRI | phones
--- | --- | ---
hoot | <http://mphilli.github.io/hoot#w055894> | hut

* Find all words that end with the phonemes for "owl" (aʊl)
```sparql
PREFIX hoot: <http://mphilli.github.io/hoot#>

SELECT ?word ?ipa WHERE {
      ?id rdfs:label ?word ; 
             hoot:asIPA ?ipa .
        FILTER regex(str(?ipa), "aʊl$")
  }
```
word | ipa |
--- | --- |
afoul | əˈfaʊl
foul | faʊl
fowl | faʊl
growl | graʊl
howl | haʊl
cowl | kaʊl
peafowl | ˈpiˌfaʊl
owl | aʊl
jowl | ʤaʊl
prowl | praʊl
waterfowl | ˈwɔtərˌfaʊl
scowl | skaʊl
towel | taʊl


* View words that end with *-ology* and start with a nasal
```sparql
PREFIX hoot: <http://mphilli.github.io/hoot#>

SELECT ?word  {
  ?id rdfs:label ?word ;
      hoot:asIPA ?d .
  BIND(REPLACE(?d, "ˈ", "") as ?e) # remove primary stress marker
  BIND(REPLACE(?e, "ˌ", "") as ?f) # remove secondary stress marker
  BIND(IRI(concat(str(hoot:), substr(?f, 1, 1))) as ?phone) 
  FILTER regex(str(?word), "ology$")
  ?phone hoot:hasPhoneType hoot:nasal .
}
```
word | 
--- |
mixology |
meteorology |
methodology |
nanotechnology |
microbiology |
micropaleontology |
mycology |
morphology |
mythology |
neurology |
numerology |
necrology |

* Get words that rhyme with "maple"
```sparql
PREFIX hoot: <http://mphilli.github.io/hoot#>

SELECT ?word WHERE {
      ?source rdfs:label ?word ; 
       hoot:asIPA ?rhyme .
    {SELECT (strafter(str(?phones), "m") as ?n) ?literal ?phones {
              ?w rdfs:label ?literal ;
              hoot:asIPA ?phones . 
               FILTER regex(str(?literal), "^maple$")
     }}
         FILTER regex(str(?rhyme), concat(str(?n), "$"))
         FILTER(str(?word) != str(?literal)) # prevent a word from rhyming itself (spelling-wise)
         FILTER(str(?phones) != str(?rhyme)) # prevent a word from rhyming itself (phonetically)
  }
  ```
  word |
  ---|
capel |
caple |
papal |
yaple |
staple |


#### Goals 

These are just a few examples of the things we can do with Hoot. What we could do with Hoot is ultimately a reflection of how we expand it. There are two primary ways two improve upon Hoot: 
  * Add more information to words at different linguistic levels beyond phonetics (e.g., syntax & semantics)
  	* This would enable us to find other, even more complex relations between words. 
  * Add more languages
  	* This would provide us with a knowledge base to discover interesting and complex relationships cross-linguistically, and allow us to perform all kinds of typological-analysis in a semantically-informed way. 
  	
(Keep in mind that [other linguistic ontologies](http://linguistics-ontology.org/info/about) exist, although the goals may be different). 
  
#### Usage and installation

Hoot exists in RDF/XML format as an `.rdf` file. I recommend placing the file in a graph database (triple store) in order to interact with it. If you have your own way of doing this, then by all means. If you need a simple way, I would recommend [Blazegraph](https://wiki.blazegraph.com/wiki/index.php/Main_Page). Simple installation instructions are described here: 

* Download the [Blazegraph .jar file](https://www.blazegraph.com/download/)
* With Java (7 or greater) installed on your machine, type `java -server -Xmx4g -jar blazegraph.jar`. This starts the Blazegraph server at `http://192.168.0.2:9999/blazegraph/`. Navigate there on your web browser. 
* In Blazegraph, select the UPDATE tab, and select the hoot.rdf file by clicking the `Browse...` button. Then hit `Update`. You should get a little report of how many triples are in the database, and how long it took to make. For example: 
    
      Modified: 401574
      Milliseconds: 7288
      
* Now navigate to the QUERY tab; here you can query the database with SPARQL. Test out some of the queries described above to get started. 
* Also, you can treat the server URL `http://192.168.0.2:9999/blazegraph/sparql` as a SPARQL endpoint. That means you can query the database in other ways. Here's an example of retrieving all English fricatives using `SPARQLWrapper` in Python: 

```Python
from SPARQLWrapper import SPARQLWrapper, JSON

sparql = SPARQLWrapper("http://localhost:9999/blazegraph/sparql")
sparql.setQuery("""
        PREFIX hoot: <http://mphilli.github.io/hoot#>
        SELECT (strafter(str(?phone), str(hoot:)) as ?vowels) {
                ?phone hoot:hasPhoneType hoot:fricative.
        }""")
sparql.setReturnFormat(JSON)
results = sparql.query().convert()
if results:
    fricatives = [(r["fricatives"]["value"])
                     for r in results["results"]["bindings"]]
    print(str(fricatives))
    # prints: ['ð', 'f', 's', 'ʃ', 'v', 'z', 'ʒ', 'θ']
```
  
