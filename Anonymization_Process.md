# EUROLAN Network Graph Preparation

## Motivation

A graph of participants, tutors, themes, courses, has been built at the end of the EUROLAN 21 Tutorial on Linguistic Linked Data. We decided that this graph will prefigure a formal description of the LLOD Network.

Hence, we setup a git project under nexuslinguarum github group and we decided to add this graph there.

## Anonymization

Until each participant and tutor agrees formally to be included in the graph, we decided that it will be anonimzed.

For this I am using Fuseki and SPARQL queries.

### Loading the graph into Fuseki

Before loading the graph into fuseki, I changed the uri prefix to http://linguistic-lod.org/network#

I also removed all \\" that was remaining in labels. And fixed several UTF-8 encoding errors.

### Querying the people

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?person
FROM <https://linguistic-lod.org/eurolan_final>
WHERE {
  ?person a foaf:Person.
}
```

### Associating each people with a number

I choose to associate each people with a number (her rank), but there is no RANK function in SPARQL 1.1. A solution found on stackoverflow consists in sorting the elements and counting the number of elements that are less than or equal to the element we have to rank.

*This is quite inefficient, but given the small size of the dataset, it's no problem*

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>

SELECT ?person (COUNT(*) as ?ranking) 
FROM <https://linguistic-lod.org/eurolan_final>
WHERE {  ?person a foaf:Person .
  ?other a foaf:Person .
  FILTER( str(?other) <= str(?person) )
}
GROUP BY ?person
ORDER BY ?ranking
```

This gives the list of persons in alphabetical order, along with it's rank.

*Note* : I could have used the UUID function to directly create a blank unique uuid.

### Creating an anonymousURI for each user

First, dry run a construct query to cretae an anonymous uri for each person.

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>

CONSTRUCT {?person owl:sameAs ?anonid}
FROM <https://linguistic-lod.org/eurolan_final>
WHERE
{
  {
    SELECT ?person (COUNT(*) as ?ranking) 
	WHERE {  ?person a foaf:Person .
  		?other a foaf:Person .
  		FILTER( str(?other) <= str(?person) )
	}
	GROUP BY ?person
    ORDER BY ?ranking
  } 
  BIND (str(?person) as ?peopleUri)
  BIND (strafter(str(?person), "#") as ?peopleName)
  BIND (strbefore(str(?person), "#") as ?prefix)
  BIND(URI(CONCAT(?prefix, "#", "anonId_", str(?ranking))) as ?anonid)
}
```

This query serves as a base for the creation of a graph containing people identity mapping.

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>

WITH <https://linguistic-lod.org/eurolan_identities>
INSERT {?person owl:sameAs ?anonid} 
USING <https://linguistic-lod.org/eurolan_final>
WHERE
{
  {
    SELECT ?person (COUNT(*) as ?ranking) 
	WHERE {  ?person a foaf:Person .
  		?other a foaf:Person .
  		FILTER( str(?other) <= str(?person) )
	}
	GROUP BY ?person
    ORDER BY ?ranking
  } 
  BIND (str(?person) as ?peopleUri)
  BIND (strafter(str(?person), "#") as ?peopleName)
  BIND (strbefore(str(?person), "#") as ?prefix)
  BIND(URI(CONCAT(?prefix, "#", "anonId_", str(?ranking))) as ?anonid)
}
```

### Copy all triples that are not bound to a person into the anonymous resulting graph

``` SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>

WITH <https://linguistic-lod.org/eurolan_anonymous>
INSERT {?s ?p ?o} 
USING <https://linguistic-lod.org/eurolan_final>
WHERE {
  ?s ?p ?o.
  MINUS { {?s a foaf:Person} UNION {?o a foaf:Person} }
} 
```

### Copy all person triples to a temporary graph

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>

WITH <https://linguistic-lod.org/eurolan_people>
INSERT {?s ?p ?o} 
USING <https://linguistic-lod.org/eurolan_final>
WHERE {
  ?s ?p ?o.
  { {?s a foaf:Person} UNION {?o a foaf:Person} }
}
```

### COPY all person triples substituting the anonymous id (filtering out personal data)

So, we just filter out literals as the rdfs:label is the only property having literal in the dataset.

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX llodnet: <https://linguistic-lod.org/network#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

WITH <https://linguistic-lod.org/eurolan_anonymous>
INSERT {?anonid ?p ?o} 
USING <https://linguistic-lod.org/eurolan_people>
USING NAMED <https://linguistic-lod.org/eurolan_identities>
WHERE {
  ?s ?p ?o.
  ?s a foaf:Person
  FILTER ( ! isLiteral(?o) )
  GRAPH <https://linguistic-lod.org/eurolan_identities> { ?s owl:sameAs ?anonid }
}
```

### Export the anonymized result

```SPARQL
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX llodnet: <https://linguistic-lod.org/network#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

CONSTRUCT {?s ?p ?o} 
FROM <https://linguistic-lod.org/eurolan_anonymous>
WHERE {
  ?s ?p ?o.
}
```
