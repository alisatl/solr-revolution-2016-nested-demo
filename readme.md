#Working with Deeply Nested Documents
### Talk at Lucene/Solr Revolution 2016

Demo queries and command  that accompany the talk Working with Deeply Nested Documents in Apache Solr (to be) presented at SolrRevolution2016 conference in Boston, MA on Oct 14, 2016.

Link to the talk slides to be provided...

### Data Pre-processing

All necessary data is provided in the ./data folder in all formats and with different pre-processing options.

Example how to run a script:

```bash
$ python ./scripts/convert_data2solrjson -i ./data/example-data.json -o ./data/example-data-solr-facet.json
```

###Creating Collection

```bash
$ bin/solr create -c demo
$ bin/post -c demo ./data/example-data-solr.json -format solr
```

###Schema Modification

```
curl -X POST -H 'Content-type:application/json' --data-binary '{
  "replace-field":{
     "name":"text",
     "type":"text_general",
     "stored":true }
}' http://localhost:8983/solr/demo/schema
```

## Queries

Query base:
http://localhost:8983/solr/demo/query?

###"Flat queries"
Find all comments and replies that mention Trump:

```
q=(path:2.posts.comments OR path:3.posts.comments.replies) AND text:Trump
```

Find all documents at level 2 that mention Hillary

```
q=path:2.* AND (text:Hillary OR Clinton)
```



### Cross-Level Querying

Block Join Query: Parent Domain
```
q={!parent%20which="path:1.posts"}path:*.keywords%20AND%20text:Hillary%20AND%20sentiment:negative
```

Block Join Query: Child Domain

```
q={!child of="path:2.posts.comments"}path:2.posts.comments AND sentiment:negative&fq=path:3.posts.comments.replies
```


ChildDocTransformer

```
q=path:2.posts.comments AND sentiment:negative&fl=id,text,path,sentiment,[child parentFilter=path:2.* childFilter=path:3.posts.comments.replies fl=text]
```


Cousin Queries
```
q={!parent which="path:2.post.comments"}path:3.posts.comments.keywords AND text:Trump&fq={!parent which=”path:2.post.comments”}path:3.posts.comments.replies AND sentiment:positive&fl=*,[child parentFilter=”path:2.*” childFilter=”path:3.*”]
```


##FACETING

### JSON Faceting API

JSON faceting API: Parent Domain  

```
curl http://localhost:8983/solr/demo/query -d 'q=path:2.posts.comments AND sentiment:positive&fl=path,text,sentiment&
json.facet={
 most_liked_authors : {
 type: terms,
 field: author,
 domain: { blockParent : "path:1.posts"}
 }}'
```
JSON faceting API: Child Domain

```
curl http://localhost:8983/solr/demo/query -d 'q=path:1.posts&rows=0&
json.facet={
 filter_by_child_type :{
   type:query,
   q:"path:*comments*keywords",
   domain: { blockChildren : "path:1.posts" },
   facet:{
     top_keywords : {
       type: terms,
       field: text,
       sort: "counts_by_posts desc",
       facet: {
         counts_by_posts: "unique(_root_)"
 }}}}}'
```

JSON faceting API: Child Faceting by intermediate ancestors

```
curl http://localhost:8983/solr/demo-facet/query -d  'q=path:2.posts.comments&rows=0&
json.facet={
 filter_by_child_type :{
   type:query,
   q:"path:*comments*keywords",
   domain: { blockChildren : "path:2.posts.comments" },
   facet:{
     top_keywords : {
       type: terms,
       field: text,
       sort: "counts_by_comments desc",
       facet: {
         counts_by_comments: "unique(2.posts.comments-id)"
 }}}}}'
```


###Block Join Faceting

Changes to solrconfig.xml of demo index:

```
<searchComponent name=”bjqFacetComponent” class=”org.apache.solr.search.join.BlockJoinFacetComponent”/>

<requestHandler name=”/bjqfacet” class=”org.apache.solr.handler.component.SearchHandler”>
  <lst name=”defaults”>
    <str name=”shards.qt”>/bjqfacet</str>
  </lst>
  <arr name=”last-components”>
    <str>bjqFacetComponent</str>
  </arr>
</requestHandler>
```

```
bjqfacet?q={!parent which=path:1.posts}path:*.comments*keywords&rows=0&facet=true&child.facet.field=text&wt=json&indent=true
```

```
bjqfacet?q={!parent which=path:2.posts.comments}path:*.comments*keywords&rows=0&facet=true&child.facet.field=text&wt=json&indent=true
```
