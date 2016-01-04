# FEL
Fast Entity Linker Core

This library performs query segmentation and entity linking to the Yahoo Knowledge base. Its current version is tailored towards query entity linking (alternatively, short fragments of text). The main goal was to have an extremely fast linker 
(< 1 or 2 ms/query on average on a standard laptop) that is completely unsupervised, so that more sophisticated approaches can work on top of it with a decent time budget available. A side effect of this is that the datapack used by the linker 
occupies <3GB making it suitable to run on the grid (and making the footprint on server machines very low).

 Future releases will contain

intent detection (what aspects of an entity the query is referring to), either linking to yo/isearch taxonomy or YK predicates
Answer type prediction 
Entity linking in long text 
Entity linking with an additional supervised layer 

The library returns a confidence score (~log likelihood) that should be (more or less) comparable across queries of different length so one can use a global threshold for linking.

If you use this library for research purposes, please cite the following paper:

@inproceedings{Blanco:WSDM2015,
        Address = {New York, NY, USA},
        Author = {Blanco, Roi and Ottaviano, Giuseppe and Meij, Edgar},
        Booktitle = {Proceedings of the Eight ACM International Conference on Web Search and Data Mining},
        Location = {Shanghai, China},
        Numpages = {10},
        Publisher = {ACM},
        Series = {WSDM '15},
        Title = {Fast and Space-Efficient Entity Linking in Queries},
        Year = {2015},
}



Code:

GitHub repo: https://git.corp.yahoo.com/roi/FEL
DEPRECATED:
Artifactory project space (jars)
https://artifactory.ops.yahoo.com:9999/artifactory/webapp/search/artifact/?12&q=pimel 

(data)
https://artifactory.ops.yahoo.com:9999/artifactory/webapp/search/artifact/?14&q=pimel-models 

Alternative datapack location (should move to artifactory in the next few days).
gwta6000.tan.ygrid.yahoo.com:/grid/0/tmp/roi/FNELPIMEL-0.0.1-SNAPSHOT-fat.jar
ENTITIES.PHRASE.model
PHRASE.model
alias.hash

MODELS:

research-hm6.corp.gq1.yahoo.com:/mnt/scratch3/roi/FEL

Stand-alone entity linking

There are a number of different rankers/linkers that use different conceptual models. The overall description of the algorithm with some implementation details is at:

https://git.corp.yahoo.com/emeij/techpulse2014-wsdm/blob/master/WSDM/paper.pdf

The two main classes to use are 
com.yahoo.semsearch.fastlinking.FastEntityLinker (no context)
com.yahoo.semsearch.fastlinking.EntityContextFastEntityLinker (with context)

The classes can be called with --help for the input option list.
They provide interactive linking through stdin (edit the code or extend for custom output format).

Example usage calls (you don't need rlwrap but it's nice to have):
rlwrap java -Xmx10G com.yahoo.semsearch.fastlinking.EntityContextFastEntityLinker -h data/alias.hash -u data/PHRASE.model -e data/ENTITIES.PHRASE.model

rlwrap java -Xmx10G com.yahoo.semsearch.fastlinking.FastEntityLinker data/alias.hash



Grid based linking
The following command works (it doesn't use the context-aware version):

hadoop jar FEL-0.1.0-SNAPSHOT-jar-with-dependencies.jar -Dmapred.map.tasks=100 -Dmapreduce.map.java.opts=-Xmx3g -Dmapreduce.map.memory.mb=3072 -Dmapred.job.queue.name=adhoc \
 -files /grid/0/tmp/roi/alias.qsi#hash  <inputfile>

The jar reads files that have one query per line - it splits on <TAB> and takes the first element. 
At the moment it outputs the following format (which is PIG-friendly) although it could print out anything else if needed to. 

(a beka summer books,{(summer,Summer,1768785,678,-4.572464942932129)(books,Book,1882136,998,-5.444955348968506)(a beka,A_Beka_Book,2889283,926,-2.051509857177734)})    1
(air cooled gasoline engines,{(gasoline engines,Gas_engine,12299268,0,-2.340573310852051)(air cooled,Air-cooled_engine,2494691,15,-1.2365872859954834)})        1

This is:
 (query, {(span, entity_yk_id, entity_fnel_id(ignore), entity_type, entity_score)} ) (entities are -not- ordered by score).

In general you should rely on thresholding and possibly sticking to the top-1 entity retrieved but this depends on how you are going to use it.


Fiddling with word embeddings


Trains and quantizes word2vec vectors for uni/bigrams and entities. An entity is anything that has 1) an identifier 2) a sequence of words describing it - a document, sentence could match 
this definition.

JAVA VERSION

--word vectors
First, compute word embeddings using whatever software you prefer, and output the word vectors in word2vec C format.

To quantize + compress the vectors:
java com.yahoo.semsearch.fastlinking.w2v.Quantizer -i <word_embeddings> -o <output> -h

java -Xmx5G it.cnr.isti.hpc.Word2VecCompress <quantized_file> <output>

--entity vectors
The program accepts a file with the following format (one per line)

entity_id <TAB> number <TAB> word sequence

The class com.yahoo.semsearch.fastlinking.w2v.EntityEmbeddings computes entity embeddings (on the grid or on a single machine).

hadoop jar FEL-0.1.0-fat.jar com.yahoo.semsearch.fastlinking.w2v.EntityEmbeddings -Dmapreduce.input .fileinputformat.split.minsize=100 -Dmapreduce.input.fileinputformat.split.maxsize=90000 -Dmapreduce.job.maps=3000
-Dmapred.child.java.opts=-XX:-UseGCOverheadLimit -Dmapreduce.map.java.opts=-Xmx3g -Dmapreduce.map.memory.mb=3072 \
-Dmapreduce.job.queuename=adhoc \ -files /grid/0/tmp/roi/vectors#vectors E2W.text entity.embeddings


Then, you can quantize + compress the resulting vectors, just like we did for word embeddings.

If you have a large number (likely) of entity vectors, it might take a while to find the right quantization factor, and you might want to use one straight ahead:

java com.yahoo.semsearch.fastlinking.w2v.Quantizer -i <embeddings> -o <output> -d 8 

To compress the vectors, it's better to use the following class (it scales up to millions of vectors):
com.yahoo.semsearch.fastlinking.w2v.EfficientWord2VecCompress




