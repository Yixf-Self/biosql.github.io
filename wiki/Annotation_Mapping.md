---
title: Annotation Mapping
permalink: wiki/Annotation_Mapping
layout: wiki
---

This page documents the recommended BioSQL-standard of mapping sequence
annotations (through Bio\* object models) to the BioSQL relational
model.

GenBank
-------

| Genbank annotation | Mapping into the BioSQL relational model         |
|--------------------|--------------------------------------------------|
| DIVISION           | column division in table bioentry                |
| accession numbers  | secondary accession numbers go into bioentry qualifier value table. Primary accession maps to column accession in table bioentry |
| TITLE              | this is part of a publication reference, and should map to column title in table reference |
| VERSION            | in BioPerl we parse this apart into a version for the accession (which is column version in table bioentry) and the GI number, which maps to column identifier in table bioentry |
| REFERENCE          | table reference (and bioentry\_reference for association with the bioentry) |
| KEYWORDS           | map to bioentry\_qualifier\_value                |
| GI                 | column identifier in table bioentry              |
| SIZE               | if it is the length of the sequence, it should map to column length in table biosequence |
| DEFINITION         | column description in table bioentry             |
| ORGANISM           | this is the organism and maps to the table taxon (and taxon\_name), with a foreign key in bioentry pointing to the taxon |
| JOURNAL            | this is part of a reference, see 'references'    |
| ACCESSION          | the primary accession, maps to column accession in table bioentry |
| LOCUS              | in the file itself this is an entire line consisting of multiple fields; BioPerl/bioperl-db maps the locus name (the first token after the literal token LOCUS) to column name in table bioentry |
| SOURCE             | this is the organism, see 'ORGANISM'             |
| PUBMED             | this is part of a literature reference, and maps to a foreign key in the reference table (reference.dbxref) to a dbxref entry with PUBMED or PMID as the database and the pubmed ID as the accession |
| AUTHORS            | part of a literature reference, maps to column authors in table reference |
| TYPE               | if this is the alphabet, it maps to table biosequence, column alphabet |
| CIRCULAR           | this at present maps to bioentry\_qualifier\_value, though there are plans to make it a column in table biosequence. |

GFF3
----

**Work in progress** - Use the information in this section at your own
risk!

See biosql-list threads: [Importing GFF3 files into a BioSQL
database?](http://lists.open-bio.org/pipermail/biosql-l/2009-February/001492.html)
following the [SO's GFF3
specification](http://www.sequenceontology.org/gff3.shtml) and [this
blog
entry](http://bcbio.wordpress.com/2009/02/22/exploring-bioperl-genbank-to-gff-mapping/).

### Major Columns

The table below describes each column of the GFF3 file, where the
content should reside in the BioSQL schema (if it is known), a summary
of the field from the original GFF3 spec, and notes on the BioSQL
mapping. The remaining columns should provide some indication of how
this data is mapped into the various Bio-\* language bindings (please
add version numbers/binding model where known if you fill in these).

| Column Number | GFF Field Name | BioSQL Table | Column | Description (summarised from [GFF3 spec](http://www.sequenceontology.org/gff3.shtml)) | Mapping Notes |
|---------------|----------------|--------------|--------|--------------|----------------------|
| 1    | seqid          | BIOENTRY     | accession | Sequence Identifier. The BioEntry being annotated. *Interpretation depends of GFF file's context* |
| 2    | source         | TERM         | name    | The source is a free text qualifier intended to describe the algorithm or operating procedure that generated this feature. In effect, the source is used to extend the feature ontology by adding a qualifier to the type creating a new composite type that is a subclass of the type in the type column. | BioSapiens have gone to some length to create a controlled vocabulary for the source term - but it is in no way complete (*TODO*). |
| 3    | type           | TERM         | name    | The SO provides the most complete type ontology, and type terms are constrained to be either: (a) a term from the "lite" sequence ontology, SOFA; or (b) a SOFA accession number. The latter alternative is distinguished using the syntax SO:000000. | The content of the type field is, in practice, usually human readable rather than a term ID, and even worse - is usually some (un)controlled vocabulary often dictated by the source term. |
| 4 & 5 | start & end    | LOCATION     | start & end | The start and end of the feature, in 1-based integer coordinates, relative to the landmark given in column 1, where start &lt;=end. For zero-length features, such as insertion sites, start equals end and the implied site is to the right of the indicated base in the direction of the landmark. | **TODO** This field is a source of ambiguity. Previous versions of GFF arbitrarily allow '-', or '' for *'start* and **end** to indicate either non-positional annotation, or annotation with an undefined start or end. The BioSQL schema supports this (see LOCATION documentation) but BioPerl bindings do not (see [this thread](http://lists.open-bio.org/pipermail/biosql-l/2009-February/001483.html)). One way of handling this is to map non-positional features directly to bioentry attributes (name, description, dbxrefs, references) - but this approach is not feasible for all non-positional features. |
| 6    | score          | SEQFEATURE\_QUALIFIER\_VALUE | value       | The score of the feature, a floating point number. As in earlier versions of the format, the semantics of the score are ill-defined. It is strongly recommended that E-values be used for sequence similarity features, and that P-values be used for ab initio gene prediction features. |
| 7    | strand         | LOCATION     | strand      | The strand of the feature. + for positive strand (relative to the landmark), - for minus strand, and . for features that are not stranded. In addition, ? can be used for features whose strandedness is relevant, but unknown. |
| 8    | phase          | LOCATION (?)                 | **TODO**    | For features of type "CDS", the phase indicates where the feature begins with reference to the reading frame. The phase is one of the integers 0, 1, or 2, indicating the number of bases that should be removed from the beginning of this feature to reach the first base of the next codon. In other words, a phase of "0" indicates that the next codon begins at the first base of the region described by the current line, a phase of "1" indicates that the next codon begins at the second base of this region, and a phase of "2" indicates that the codon begins at the third base of this region. This is NOT to be confused with the frame, which is simply start modulo 3. For forward strand features, phase is counted from the start field. For reverse strand features, phase is counted from the end field. The phase is REQUIRED for all CDS features. | This is a field specific to DNA. Most GFF files related to protein sequences specify PHASE as '0'. Some overload it in some way. |
| 9     | attributes     | *See below*                  | *See Below* | A list of feature attributes in the format tag=value. Multiple tag=value pairs are separated by semicolons. URL escaping rules are used for tags or values containing the following characters: ",=;". Spaces are allowed in this field, but tabs must be replaced with the %09 URL escape. | The content of the attributes field can be used to specify a host of properties - it is frequently used to specify URLs, database accessions or addtional references associated with a feature. It is also essential for the representation of alignments. |

**TODO - Format below and merge in with interpretations from [this blog
entry](http://bcbio.wordpress.com/2009/02/22/exploring-bioperl-genbank-to-gff-mapping/)**

    These tags have predefined meanings:

        ID     Indicates the name of the feature.  IDs must be unique
           within the scope of the GFF file.

        Name   Display name for the feature.  This is the name to be
               displayed to the user.  Unlike IDs, there is no requirement
           that the Name be unique within the file.

        Alias  A secondary name for the feature.  It is suggested that
           this tag be used whenever a secondary identifier for the
           feature is needed, such as locus names and
           accession numbers.  Unlike ID, there is no requirement
           that Alias be unique within the file.

        Parent Indicates the parent of the feature.  A parent ID can be
           used to group exons into transcripts, transcripts into
           genes, an so forth.  A feature may have multiple parents.
           Parent can *only* be used to indicate a partof 
           relationship.

        Target Indicates the target of a nucleotide-to-nucleotide or
           protein-to-nucleotide alignment.  The format of the
           value is "target_id start end [strand]", where strand
           is optional and may be "+" or "-".  If the target_id 
           contains spaces, they must be escaped as hex escape %20.

        Gap   The alignment of the feature to the target if the two are
              not collinear (e.g. contain gaps).  The alignment format is
          taken from the CIGAR format described in the 
          Exonerate documentation.
          (http://cvsweb.sanger.ac.uk/cgi-bin/cvsweb.cgi/exonerate
              ?cvsroot=Ensembl).  See "THE GAP ATTRIBUTE" for a description
          of this format.

        Derives_from  
              Used to disambiguate the relationship between one
              feature and another when the relationship is a temporal
              one rather than a purely structural "part of" one.  This
              is needed for polycistronic genes.  See "PATHOLOGICAL CASES"
          for further discussion.

        Note   A free text note.

        Dbxref A database cross reference.  See the section
           "Ontology Associations and Db Cross References" for
           details on the format.

        Ontology_term  A cross reference to an ontology term.  See
               the section "Ontology Associations and Db Cross References"
           for details.

    Multiple attributes of the same type are indicated by separating the
    values with the comma "," character, as in:

           Parent=AF2312,AB2812,abc-3

    Note that attribute names are case sensitive.  "Parent" is not the
    same as "parent".

    All attributes that begin with an uppercase letter are reserved for
    later use.  Attributes that begin with a lowercase letter can be used
    freely by applications.
