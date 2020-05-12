# Sling

A collection of elasticsearch command line tools, with query based filtering capabilities.
- Bulk **ingest** large datasets files (currently supports .csv) with basic fields transformations and mapping. 
- **create/delete** an index with mapping and analysers. 
- **dump** an index to bulk files : data, mapping and analysers.
- **index** .bulk files : index data, mapping and analysers.

## Version Warning
Sling works with ElasticSearch 6.x and higher.

## Use of a query filter to dump, delete or count
Sling can use a query to filter what you want to. The query used is an ElasticSearch DSL query :
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "John"
          }
        }
      ]
    }
  }
}
```
Use the query as an argument of the command like this
```bash
sling command -q '{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "John"
          }
        }
      ]
    }
  }
}'  index
```
>**Windows users**, we recommend that you copy the query in a json file and then execute the command in a cmd window : 
    ```
    sling command -q query.json 
    ```
>Or you can use it as an argument of the command by escaping the json like this :
    ```
    sling command -q "{\"query\": {\"bool\": {\"must\": [{\"match\": {\"name\": \"John\"}}]}}}'
    ```

## Sling's featured commands
### Ingest

A pretty straightforward method to define your data mapping, apply basic transformations and bulk ingest your datasets from files into existing or new indexes. 
 
#### Quick starting guide :

1. Create the config file for your dataset.csv
    ```
    sling ingest create-config -s ',' dataset.csv
    ```
2. Edit your config file according to your needs (datatypes, basic transformations, add more csv files...)
3. bulk ingest you data into Elasticsearch, using the config.json parameters. 
    ```
    sling ingest config.json
    ```

#### 1. Generating a configuration file
Use the sub command `create-config` to generate a .json configuration file from a CSV structure.
```bash
# create a config file named metadata.json (default name) based on dataset.csv structure with default separator (;)
sling ingest create-config dataset.csv
```
>Only one CSV file can be passed as a parameter. 

Edit the metadata.json to adapt mapping to your needs. (see below).


>The command generates a template describing anything you need to ingest your dataset.csv into ES. Edit the template to apply the desired mapping (default field datatype is "keyword") and  desired transformations (see below ).

Optional parameters are :
* -s (or --separator) (default is ;) to specify the separator used in the data file
* -i (or --index) (default TODO) to specify the name of the index
* -o (or --output-file) to specify the output for the configuration file

A more complete example :
```bash
# create a config file named config.json based on dataset file.csv with index my-index. The data file uses separator ,
sling ingest create-config -s ',' -i my-index -o config.json file.csv
```

#### 2. Edit the configuration file

The configuration file is a JSON file, describing the metadata of the files (name, headers type, separator).
Below is a simple configuration file :

```json
{
    "inputs": [
        {
            "files": [
                "path/file.csv"
            ],
            "index": "test-index",
            "schema": {
                "fields": [
                    {
                        "id": "Column1",
                        "type": "keyword"
                    },
                    {
                        "id": "Column2",
                        "type": "integer"
                    },
                    {
                        "id": "Column3",
                        "type": "date",
                        "format": {
                            "input": "yyyyMM",
                            "output": "epoch_millis"
                        }
                    }
                ],
                "separator": ";",
                "withHeaders": true
            }
        }
    ]
}
```

`inputs` is the main key of the JSON file, it describes an index and the data used to fill it. It's an array, meaning we can specify differents index with different files structures.
An `input` has the following keys
* `files`
* `index`
* `schema`

`files` is the list of files that will be indexed into your ElasticSearch. All of the files __must__ be of the same type.
`index` is the name of the index into which the data will be written.
`schema` describes how the files are formatted :

Inside the `schema` object, we have those keys : 
* `fields`
* `separator`
* `withHeaders`

`fields` is a list of all the headers (for a CSV), keys (for XML and JSON) we want to retrieve from the files and their types. (skip what you do not need)

__Required keys to describe a field :__

* An `id` representing the name of the column/key in the file (case sensitive)

* A `type` representing the type of the column/key. The type must match the ElasticSearch field datatypes (https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html) .
* A `rename` (optional) key can be added to rename the field in ElasticSearch.

```json
{
    "id": "Column1",
    "type": "keyword",
    "rename": "myColumn"
}
```

__Some specific types offer more options.__

- for `date` datatype you __must__ specify a `format` :

    ```json
    {
       "id": "Column3",
        "type": "date",
        "format": {
            "input": "yyyyMM",
            "output": "epoch_millis"
        }
    }
    ```
    * `input` is the date format in the file. Many common types are supported. 

    * `output` is how you want the date to be stored in ElasticSearch, it must match ElasticSearch date datatype (https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html)


* for `number` : basic formatting with `format` option.

    If the datatype is a number (float, integer, long, ...), you can add a formatter to perform simple operations on it.

    ```json
    {
       "id": "quantity",
        "type": "float",
        "rename": "quantity10",
        "format": {
            "output": "`quantity`/10"
        }
    }
    ```

    * The name of the field in `output` must be in backquote '`'. 
    * You can only perform operations on the current field, not between fields.
"\`quantity\`*\`price\`*10" will not work

#### 3.  Bulk ingest datasets
Once you are satisfied with you configuration file, you are good to go!
```bash
# Ingest the data as described in file config.json 
sling ingest config.json
```
Optional parameters are :
* -w (or --workers) (default is 3) to specify the number of workers that will performs tasks in parallel
* -s (or --size) (default is 2MB) to specify the size in MB of the bulk sent to ElasticSearch
* -a (or --append) to specify if the data will be added to an existing index, if not present, then it will recreate the index

> Use options to push performance. Remember that "Race" settings require a skilled pilot!
![Manettino](https://secure.gravatar.com/avatar/95530e6f8342c5cde314cedd69f27526?s=96&d=blank&r=g)

---
### Count
Count the number of documents in one or more indexes. Sums up the number of documents.
```bash
# In this example, it will count all of the documents of idx and idx2
sling count idx idx2
```
Optional parameter is :
* -q (or --query) to specify a query to filter the counted documents

---
### Index

Create or recreate an index with it's mapping and analyser. If the index already exists and the recreate options is not provided, the index won't be created.
You can also delete an index.
One or more index can be specified in both commands.

```bash
# Create index idx with the mapping described in file mapping.json and the analyser described in file analyser.json
sling index create -m mapping.json -a analyser.json idx

# Delete the indexes idx and idx2
sling index delete idx idx2
```

Parameters must be :
* __create__ to create an index
* __delete__ to delete an index

Optional parameters for __create__ are :
* -r (or --recreate) to create the index (delete, then create)
* -m (or --mapping) to specify the mapping of the index (can be a JSON file, a JSON text or the mapping of another index). JSON mapping must be the as follow : https://www.elastic.co/guide/en/elasticsearch/reference/7.6/mapping.html#create-mapping
* -a (or --analyser) to specify the analyser of the index (can be a JSON file, a JSON text or the analyser of another index). JSON mapping must be the as follow : https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-custom-analyzer.html
* -q (or --query) to specify a query to filter the counted documents

Optional parameter for __delete__ is :
* -q (or --query) to specify a query to filter the counted documents
---
### Dump
Dump the data, the mapping, the analyser or everything of an index.

```bash
# dump the data of index idx into files named file.bulk. if one file is not enough to contain all the data, a file rotation will be performed until all the data is dumped. The max-size indicates that each output file will be roughly 5MB
sling dump -f data --max-size 5 -o file.bulk idx

# dump everything (data, mapping and analyser) of the index idx. All of the data will be dump in the single (as there is no parameters of size or lines) file file.bulk. Mapping and analyser will be dumped into respectively mapping-idx.json and analyser-idx.json
sling dump -f all -o file.bulk idx
```

Optional parameters are :
* -f (or --for) to specify what will be dump, possible values are [analyser and/or data and/or mapping] or [all]
* -s (or --size) to specify the size of the scoll used to paginate the data from ElasticSearch (default is 1000 documents)
* -k (or --keep-alive) to specify the duration for which the scroll will be kept alive (default is 5m)
* -d (or --destination-index) to specify a output index. For example, we are dumping the index idx and want to create bulk files for the index idx2
* -o (or --output-file) to specify the file in which the dump will be performed
* --max-size to specify the maximum size in MB of the output file. If the size of the data is bigger than this, then a file rotation is performed
* --max-line to specify the number of lines of the output file. If the number of line of the data is bigger than this, then a file rotation is performed. Priority is on max-size if both options are used
* -a (or --append) to specify if we append the data to the output file or no. If option is not present, then it's not appended.
* -g (or --gzip) to sepcify if we create an archive of the whole dump (data + mapping + analyser)
---
### Bulk

Bulk the files into Elasticsearch

```bash
# bulk all the file that match test_*.bulk
sling bulk test_*.bulk
```

Optional parameter is :
* -g (or --gzip) to specify a zip file, extract it and bulk all the non json files
