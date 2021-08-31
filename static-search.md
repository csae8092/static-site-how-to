# Static-Search

## Prerequisites

This post picks up where [publishtei.mdx](publishtei.mdx) ended. We managed to publish a direcotry of HTML files as GitHub Pages which we generated via XSLT from XML/TEI Documents with the help of GitHub Actions. But now we want to implement a full text search for our data allowing

- a fuzzy-search
- some custom filters/facets
- a KWIC (KeyWordInContext) preview of the potential matches
- a scoring mechanism to rank potential matches by their importance. 

Since we don't want to (or can't) fall back on any of the usual suspects in the area of search, full-text search or information retrieval, such as Solr or Elastic Search, we have to find a "static" alternative. Such an alternative called [staticSearch](https://github.com/projectEndings/staticSearch) is being developed as part of the [projectEndings](https://endings.uvic.ca/) project. And here we show how to integrate staticSearch into an existing project.

### staticSearch - what it does

staticSearch is basically responsible for two things. On the one hand for the creation of a search index based on HTML files and on the other hand for the implementation of a search interface. The first consists of a collection of small `JSON` files in which it is recorded in a structured form where, in which file and at which position the searched token, the searched word occurs. The latter is a website, in which the desired term (and previously configured additional filters can be set) is entered in a search slot and the results are displayed.

### install staticSearch

To use staticSearch it must be downloaded and unzipped. We can do this with the similar commands we used to download and install Saxon in the previous blog post.

```shell
wget https://github.com/projectEndings/staticSearch/archive/refs/tags/v1.3.0.zip
unzip v1.3.0.zip -d tmp
mv ./tmp/staticSearch-1.3.0 ./staticSearch && rm -rf ./tmp
```

The result of these commands is a directory `staticSearch` in the same folder where the commands were executed. To avoid unnecessarily bloating our git repo, we put this directory on our `.gitignore` list. Or rather, we don't even need to download staticSearch to our local machine, since we have our static website built by GitHub anyway.

For this we add a new "job" to `.github/workflows/build.yml` which we call something like "Install staticSearch": 

```yaml
  - name: Install StaticSearch
    run: |
      wget https://github.com/projectEndings/staticSearch/archive/refs/tags/v1.3.0.zip && unzip v1.3.0.zip -d tmp
      mv ./tmp/staticSearch-1.3.0 ./staticSearch && rm -rf ./tmp
```

We also need to download and install an extension for [ant](https://ant.apache.org/) called [ant-contrib](http://ant-contrib.sourceforge.net/) required by staticSearch. This is done for us by the following commands:
```shell 
wget https://repo1.maven.org/maven2/ant-contrib/ant-contrib/1.0b3/ant-contrib-1.0b3.jar
mv ant-contrib-1.0b3.jar /usr/share/ant/lib
```

### configuration of staticSearch

The next step is to configure staticSearch. How and what has to be done concretely and which optional settings are possible can be read in the documentation of staticSearch - which by the way uses the search function of staticSearch itself!

To make staticSearch work, we need to create three files, namely `words.txt`, `stopwords.txt` and `ss_config.xml`. For our minimal example it is enough if there are the first two files, even if they are empty files. Of incomparably greater importance, however, is `ss_config.xml`. As the name suggests, it contains the information needed by staticSearch to create the search index and the search interface. 

```
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns="http://hcmc.uvic.ca/ns/staticSearch">
    <params>
        <searchFile>./html/search.html</searchFile>
        <recurse>false</recurse>
        <linkToFragmentId>true</linkToFragmentId>
        <scrollToTextFragment>false</scrollToTextFragment>
        <phrasalSearch>true</phrasalSearch>
        <wildcardSearch>true</wildcardSearch>
        <createContexts>true</createContexts>
        <resultsPerPage>5</resultsPerPage>
        <minWordLength>3</minWordLength>
        <!--NOTE: If phrasalSearch is set to TRUE, then
        maxContexts prop will be ignored-->
        <maxKwicsToHarvest>5</maxKwicsToHarvest>
        <maxKwicsToShow>5</maxKwicsToShow>
        <totalKwicLength>15</totalKwicLength>
        <kwicTruncateString>...</kwicTruncateString>
        <verbose>false</verbose>
        <stopwordsFile>stopwords.txt</stopwordsFile>
        <dictionaryFile>words.txt</dictionaryFile>
        <indentJSON>false</indentJSON>
        <outputFolder>static-search</outputFolder>
    </params>
</config>
```

The meaning of the individual parameters listed therein can also be read at staticSearch. Therefore, only the absolutely necessary settings will be discussed here. 
The element `<searchFile>./html/search.html</searchFile>` determines the directory in which the HTML files are to be found, which we want to search finally. The file `search.html` referenced here does not have to exist at all, since this is created by staticSearch in the context of the index creation. 

If our HTML files are scattered over several subdirectories of `html`, the `<recurse>` parameter must be set to `true`. However, in order not to complicate the navigation within our static website unnecessarily, no subdirectories for HTML files should be created anyway.

For our minial setup the elements `<stopwordsFile>stopwords.txt</stopwordsFile>`, <dictionaryFile>words.txt</dictionaryFile> and <outputFolder>static-search</outputFolder> are relevant. The first two parameters point to the stopword and word lists discussed earlier, although these can also be empty files. `outputFolder` on the other hand specifies the name of the directory that is created by staticSearch to store the files necessary for the search index and for the search interface.  

### Building the Search Index and Search Interface

Now that all components have been submitted, the build process for the index creation still needs to be triggered. For this we call ant again, pass the program with the parameter `-f ./staticSearch/build.xml` on the one hand the path for the `build.xml` file supplied by `staticSeach` and with the parameter `-DssConfigFile=${PWD}/ss_config.xml` the path to the configuration file just discussed. 

Translated into the YAML format required by GitHub Actions this results in: 

```yaml
- name: Build Index
  run: |
    ant -f ./staticSearch/build.xml -DssConfigFile=${PWD}/ss_config.xml
```

Depending on the size of the project and the vocabulary used in the HTML files, the build process can take several minutes. After a successful completion, however, both a file named `search.html` and a directory named `static-search` should have been created in the `html` folder. 

Our website published via GitHub Pages can now be searched in full text at `https://{github-user-name}.github.io/{git-repo}/search.html`.
