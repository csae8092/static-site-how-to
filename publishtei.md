# Static Page

## Prerequisites

The goal of this tutorial is a contemporary and sustainable online publication of your digital edition without own infrastructure like servers or databases, only with the help of GitHub Pages and GitHub Actions. 

To follow this tutorial you need XML/TEI coded text, an XML capable text editor like oXygen and ideally basic knowledge in XSLT and HTML. Basic knowledge of GIT as well as GitHub account is also required.

The data basis for this tutorial comes from the research project "Aratea Digital" by Ivana Dobcheva. This is a collection of XML/TEI encoded descriptions of early medieval manuscripts, supplemented by XML/TEI encoded bibliographic summaries of the texts mentioned in the manuscripts. This is an ongoing project, the current state of the data can be viewed in this [GitHub repo](https://github.com/ivanadob/aratea-data).

## Workflow and Architecture

The workflow for publishing the XML/TEI data via [GitHub Pages](https://pages.github.com/) and [GitHub Actions](https://docs.github.com/en/actions) can be roughly summarized as follows:

- Data is edited locally with any text editor.
- Changes in the data are regularly `commited` and then `pushed` to the GitHub repo
- Every time a script is pushed to the GitHub repo, a so-called `GitHub Action` is triggered, i.e. a script runs on the GitHub servers and performs the following tasks:
  - The XML/TEI files are converted to HTML files using XSL stylesheets
  - The generated HTML files are copied to a separate directory
  - All these files are finally published as so-called GitHub pages and together form the online edition of the XML/TEI files.

## Actual implementation

### Directory Structure

To make the above more concrete, let's start with the directory structure of the repo. By convention, the XML/TEI files go into a directory called `data` and the XSL stylesheets go into a directory called `xsl`. In this example, the `data` directory includes other subdirectories, and these directories correspond to the different document types. However, currently only the files from `descriptions` and `texts` as well as `meta` are of interest. `descriptions` contains the manuscript descriptions, `texts` the summaries of the (ancient) texts and in `meta` is the file `about.xml`, which, also encoded in XML/TEI, contains general information about the specific research project.

Finally, there needs to be a directory for so-called static content. In this case this directory is called `html` and contains (for the time being) only another folder called `css`, which is used to store self-written CSS stylesheets. In addition, the `html` directory serves as the target directory or storage location for our XML/TEI to HTML transformations. 
> NICE TO HAVE: eine graphische Darstellung der oben beschriebenen Collection-Hierarchy

### XSLTs

Since this is an evolved project, the `xsl` directory also contains XSLTs for data curation and cleansing. In addition, the project uses an adapted version of a stylesheet for converting XML/TEI encoded manuscript descriptions to HTML, written by Thorsten Schaßan at the Herzog August Bibliothek. Specifically, this is `desc_tei_to_html.xsl` and the files in the subdirectory `./xsl/hab-xsl`. However, a look at the names of the individual XSLTs should quickly provide clarity about their task.

`desc_tei_to_html.xsl` converts the XML/TEI encoded descriptions of the manuscripts stored in the `data/descriptions` folder to HTML and the XSLT `text_tei_to_html.xsl` converts the XML/TEI encoded documents in the `data/texts` directory to HTML. Attentive readers might recognize a pattern here ... .

This leaves three more XSLTs relevant for converting our data to HTML, namely `make_index.xsl`, `make_index_mss.xsl` and `nav_bar.xsl`.

`make_index.xsl` transforms the already mentioned XML/TEI encoded `data/meta/about.xml` into an HTML document named `index.html`, which - following the GitHub Pages conventions - serves as the entry or start page of our website. 

With `make_index_mss.xsl` an HTML is generated that represents a table with structured information extracted from the XML/TEI encoded information about the individual manuscripts. This page serves as a navigation menu to the individual descriptions.

Finally, `nav_bar.xsl` contains that part of the code which is responsible for displaying the navigation menu of the website. This code part, or more specifically the xsl-template `nav_bar` in `nav_bar.xsl` is also required by previously mentioned XSLTs and is included using `<xsl:import href="nav_bar.xsl"/>` and called using `<xsl:call-template name="nav_bar"/>`. This technique allows you to edit code that is used in several places in the project in only one place.

Since this is not an XSLT tutorial, the stylesheets mentioned above will not be discussed in detail. The only really important thing for the fulltext search to work though is that you generate well-formed XHTML files with you XSLTs.

### BUILD Scripts

Having the XML/TEI data and the needed XSLTs in place, we need a way to bring those together to create HTML files. In more technical term we would speak of some kind of "build process" to create or *build* all folders and files needed for our website to work. If you are familiar with oXygen you can *build* your website in a semi automatic way by 

- defining the needed scenarios to
  - transform all files from folder `data/descriptions` with `xsl/desc_tei_to_html.xsl` and save the result as `html/{name-of-source-xml}.html`
  - transform all files from folder `data/texts` with `xsl/text_tei_to_html.xsl` and save the result as `html/{name-of-source-xml}.html` into the `html` directory.
  - transform `data/meta/about.xml` with `xsl/make_index.xsl` and save the result as `html/index.html` 
  - transform `data/meta/about.xml` with `xsl/make_index_mss.xsl` and save the result as `html/index-mss.html` 
- run those scenarios

Although those scenarios are not needed for our GitHub Pages served digital edition it is nice to have those in place to check your XSLTs and the generated HTML output quickly while developing the XSLTs.

#### ANT

For the actual automated build process we need some kind of script which basically performs the tasks/scenarios mentioned above. In principle, this script can be written in any programming language, but in our project we decide to define and process the individual tasks using [Apache Ant](https://ant.apache.org/). The reasons for this are of a more pragmatic nature: 
- Ant processes are defined in XML format
- Ant is supported by oXygen
- and Ant is based on Java, as well as the XSL processor [Saxon](http://saxon.sourceforge.net/) which is used to transform XML/TEI documents to HTML.

By convention such a process definition file is called `build.xml`. The `build.xml` in for our project looks as follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project basedir="." name="tei2html">
    <!-- some variables so we spare us some typing and/or copy-pasting later -->
    <property name="source_desc" value="./data/descriptions"/>
    <property name="source_text" value="./data/texts"/>
    <property name="target" value="./html"/>
    <property name="stylesheet_desc" value="./xslt/desc_tei_to_html.xsl"/>
    <property name="stylesheet_text" value="./xslt/text_tei_to_html.xsl"/>
    <property name="index.source" value="./data/meta/about.xml"/>
    <property name="index.style" value="./xslt/make_index.xsl"/>
    <property name="index-mss.style" value="./xslt/make_index_mss.xsl"/>
    
    <!-- here we delete all html-files in our target folder -->
    <delete>
        <fileset dir="${target}" includes="*.html"/>
    </delete>
    <delete dir="${target}/static-search"/>
    
    <!-- here we define the transformation scenarious to bulk process files in a directory-->
    <xslt style="${stylesheet_desc}"  basedir="${source_desc}" destdir="${target}" includes="*.xml">
        <factory name="net.sf.saxon.TransformerFactoryImpl"/>
        <classpath location="${basedir}/saxon/saxon9he.jar"/>
    </xslt>
    <xslt style="${stylesheet_text}" basedir="${source_text}" destdir="${target}" includes="*.xml">
        <factory name="net.sf.saxon.TransformerFactoryImpl"/>
        <classpath location="${basedir}/saxon/saxon9he.jar"/>
    </xslt>

    <!-- here we define the transformation scenarious for single files -->
    <xslt in="${index.source}" out="${target}/index.html" style="${index.style}">
        <factory name="net.sf.saxon.TransformerFactoryImpl"/>
        <classpath location="${basedir}/saxon/saxon9he.jar"/>
    </xslt>
    <xslt in="${index.source}" out="${target}/index-mss.html" style="${index-mss.style}">
        <factory name="net.sf.saxon.TransformerFactoryImpl"/>
        <classpath location="${basedir}/saxon/saxon9he.jar"/>
    </xslt>
    
    <!-- here some cleanup in our generated HTML files to get rid of some namesspace chaos -->
    <replace dir="${target}" value="">
        <include name="*.html"/>
        <replacetoken> xmlns=""</replacetoken>
    </replace>
</project>
```

The actual definition of the transformation processes is done in the `<xslt>` elements. Since this is not an ANT tutorial, we will not go into more detail about the actual syntax, which is already comprehensively documented [here](https://ant.apache.org/manual/Tasks/style.html). Important for the current context is only that

- the attribute `style` contains the path to the XSL stylesheet
- the attributes `basedir` and `includes` define which data are to be transformed (i.e. all files in `data/descripitions` ending with `.xml`)
- and the attribute `destdir` determines the directory where the results of the transformation should be saved. 
By default, the generated HTML documents are conveniently saved under the same file name as the source file, but with the file extension `.html`.

A similar syntax can be used to define the transformation of individual files. Here the `in` and `out` attributes define the input and output document.

If you want to run the build script locally, you also have to consider the elements `<factory name="net.sf.saxon.TransformerFactoryImpl"/>` and `<classpath location="${basedir}/saxon/saxon9he.jar"/>`. In the `location` attribute of the `classpath` element the path to the Saxon jar file used for transformation must be specified.

#### GitHub Actions

With the source data (XML/TEI), the XSLTs and the just mentioned `build.xml` all necessary building blocks for our digital edition are available. What is still missing is an automated workflow that puts the individual parts together. This is where the GitHub Actions come into play.

With the help of a configuration file `.github/workflows/build.yml` we define, that every time pushes are made against the GitHub repo,

- the software needed to process the data is installed and configured
- the ANT process is started
- and publish the generated HTML data via GitHub Pages.

The above translated into the necessary GitHub Actions syntax results in the following YAML configuration file:

```yaml
name: Build and publish

on: 
  push:

jobs:
  build_pages:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
    - name: Perform Checkout
      uses: actions/checkout@v2
    - name: Install Saxon and ANT
      run: |
        apt-get update && apt-get install openjdk-11-jre-headless ant -y --no-install-recommend
        wget https://sourceforge.net/projects/saxon/files/Saxon-HE/9.9/SaxonHE9-9-1-7J.zip/download && unzip download -d saxon && rm -rf download
    - name: Build
      run: |
        ant
    - name: Deploy
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: ./html
```

Again, this is not a tutorial on GitHub Actions, so just a few notes on this configuration. 
We define in this file only one job (`build_pages`), give it a human readable name (`name: Build and Deploy`) and define that all further steps should run under Linux/Ubuntu (`runs-on: ubuntu-latest`). We then define individual sequential steps that need to be worked through. This corresponds to a large extent to the typing of successive commands in the command line

Crucial for our project is the installation of missing programs or program libraries (`name: Install Saxon and ANT`) into the operating system used by GitHub Actions here (`ubuntu-latest`). This takes place concretely with the command `apt-get update && apt-get install openjdk-11-jre-headless ant -y --no-install-recommend`, with which with the help of the Advanced Packaging Tool, briefly apt, Java and Ant are installed.

The next command `wget https://sourceforge.net/projects/saxon/files/Saxon-HE/9.9/SaxonHE9-9-1-7J.zip/download && unzip download -d saxon && rm -rf download`

- downloads the already mentioned XSLT processor Saxon (`wget {some-url}`),
  - unpacks the zip file and copies the files it contains into the directory `saxon` (`unzip download -d saxon`), i.e. exactly the place we have designated for it in our `build.xml` (`<classpath location="${basedir}/saxon/saxon9he.jar"/>`)
- and removes the original downloaded zip (`rm -rf download`) 

The next step, building our HTML files, is very simple thanks to our preliminary work. The command `ant` reads the configuration file `build.xml` and starts the build process. This means that the transformations we defined are processed and the results are stored in the `html` directory.

What follows is the "Deploy" step. For this we use the already existing GitHub action `peaceiris/actions-gh-pages@v3` which roughly speaking checks in our `html` folder into its own Git branch and puts it online via GitHub Pages. Mostly self-explanatory is the parameter `./html` which defines the directory to be published, i.e. `./html`. The `github_token: ${{ secrets.GITHUB_TOKEN }}` key-value pair is required by the used GitHub Action but will be set up automatically. 

With this setup, the configured pipeline is triggered with every push event (`on: push:`) and the current state of the data in the repo is published as a website.

## Wrap up and and next steps

We have come a long way and now have an automated workflow that ensures that our data is accessible online and is always up to date. Since the transformation of the data is not done on our local computer, we don't have to worry about installing and updating required programs and software libraries. And thanks to GitHub Pages, we don't have to worry about a web server making our data available to the world. Therefore focus on the actual work on the data and continue creating great content.

For some projects, what has been implemented so far may already be sufficient. Depending on the scope and content of the project, an efficient full-text search is an absolutely necessary feature. How to configure such a full-text search and integrate it into the existing workflow will be described in the next blog post.
