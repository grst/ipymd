[![Build Status](https://travis-ci.org/grst/ipymd.svg?branch=travis)](https://travis-ci.org/grst/ipymd)
[![Coverage Status](https://coveralls.io/repos/grst/ipymd/badge.svg)](https://coveralls.io/r/grst/ipymd)

# Scope of this fork
This fork provides an RMarkdown content manager for Jupyter Notebook, *i.e.* it allows you to edit `.Rmd` documents
as if they were notebooks in Jupyter. 

Inputs are stored in Rmd, outputs in a corresponding `.nb.html` which attempts to be compatible with Rstudio. 

**Unless you are particularly interested in the `.nb.html` output, 
you should probably use [mwouts/jupytext](https://github.com/mwouts/jupytext) instead, which is more actively developed. 
It also supports storing cell outputs in a dedicated `.ipynb` file. **

# Store Jupyter notebooks in markdown format. 
This package provides an alternative content manager for jupyter.
It allows to store notebooks in text-based formats, replacing the native json-based `.ipynb`. 

This combines the advantages of a simple, text-based format (vi and git-friendly) with jupyter's powerful UI for interactively editing code and text. 


## Overview of formats
Ipymd currently supports the following formats:

| Format       | Extension          | vi | git | images |
| ------------ | ------------------ | -- | --- | ------ |
| [notebook](#ipython-notebook-ipynb) | `.ipynb`           |    |     | ✔      | 
| [rmarkdown](#rmarkdown-rmd--rnotebook-nbhtml) | `.Rmd`, `.nb.html` | ✔  | ✔   | ✔      |
| [markdown]()     | `.md`              | ✔  | (✔) |        |
| [atlas](#oreilly-atlas-md) | `.md`              | ✔  | (✔) |        | 
| [opendocument](#opendocument-odt) | `.odt`             |    |     |        |
| [python](#python-py)       | `.py`              | ✔  | (✔) |        |

✔ works; (✔) works with limitations


## Usage
Ipymd hooks into jupyter, enabling to open the files directly in jupyter notebook. 

Alternatively, you can use ipymd to convert between the formats from command line: 

```
ipymd my_notebook.ipynb --from notebook --to markdown
```

Additional options:
```
  -h, --help            show this help message and exit
  --output OUTPUT       output folder
  --extension EXTENSION
                        output file extension
  --overwrite           overwrite target file if it exists (false by default)
```


## Installation
There are two possibilities to use ipymd:

1. **Within a virtual environment, for testing and developing**
    ```shell
    git clone https://github.com/grst/ipymd
    make jupyter
    ```
    will setup a virtual environment and run a `jupyter notebook` instance with ipymd activated.

    You can choose the format by editing `.jupyter/jupyter_notebook_config.py`:
    ```python
    c.IPymdContentsManager.format = 'rmarkdown'  # choose the format here
    ```


2. **Integrated into your local jupyter installation**
    * Install ipymd
        ```shell
        pip install  https://github.com/grst/ipymd/archive/master.zip
        ```

    * Open your `jupyter_notebook_config.py`. Here's how to find it:
        ```
        jupyter notebook --generate-config  # generate a default config file
        jupyter --config-dir  # find out the path to the config file
        ```

    * Add the following in `jupyter_notebook_config.py`:
        ```python
        c.NotebookApp.contents_manager_class = 'ipymd.IPymdContentsManager'
        c.IPymdContentsManager.format = 'rmarkdown'  # choose the format here
        ```

    * (re)start jupyter

**Optional:**
To interact with `.odt` files:

```shell
pip install git+https://github.com/eea/odfpy
```

## Caveats

**WARNING**: use this library at your own risks, backup your data, and version-control your notebooks and Markdown files!

* Renaming doesn't work yet (issue #4)
* New notebook doesn't work yet (issue #5)
* Only nbformat v4 is supported currently (IPython 3.0)



## Formats
### IPython notebook (`.ipynb`)
Jupyter's default notebook format. It stores cells as json-objects. 
The main downsides of this format are
* not git-friendly
* cannot easily edit in a text editor
* cannot easily edit on GitHub's web interface

[Format documentation](http://nbformat.readthedocs.io/en/latest/)

### RMarkdown (`.Rmd`) / RNotebook (`.nb.html`)
RMarkdown is propagated by rstudio and widely adopted within the R community. 
Unlike the name suggests, it can very well be used with python. 

The clue about this format is, that it strictly separates source code from output. 
This makes it the format of choice when working with version control. 

While the source code is stored as markdown in a `.Rmd` file, the results go into a
`.nb.html` file which can also be viewed in a browser. 

[Format documentation](http://rmarkdown.rstudio.com/r_notebooks.html)


#### Known issues
See [grst/ipymd/issues](https://github.com/grst/ipymd/issues) for issues related to rmarkdown. Major issues:
* HTML formatting can be improved
* Some output is not compatible with rstudio

#### Implementation of `.Rmd` format
* markdown cells are saved as plain markdown
* code cells are saved as code chunks, separated by a newline
  ~~~
  ```{python, some="meta", data=True}
  print("Hello World!")
  ```
  ~~~
  Note the curly braced `{}` which distinguish an executed code chunk from 
  a code chunk within markdown. 
* metadata is saved as chunk options.
    * Both python and R literals are supported (`NULL`, `None`, `TRUE`, `True`, `FALSE`, `False`),
    but always saved as R literals to maintain compatibility with rstudio. 
    * Both single and double quoted strings are supported. 
    * We try to parse unquoted options as literal, then as integer, then as float. If all three fail a `TypeError` is raised. 
    
#### Implementation of `nb.html` format. 
* This format stores the outputs of the notebook in a way that
  * the outputs can be read from jupyter
  * the entire notebook can be viewed from a browser
* a html templated is used, which is filled using `jinja2`. 
* markdown cells are saved within `<!--rnb-text-begin -->...<!--rnb-text-end -->` tags
* code cells are saved within `chunk` tags:
  ```
  <!--rnb-chunk-begin -->
    <!--rnb-source-begin {base64}-->
      <pre><code>...</pre/code>
    <!--rnb-source-end -->
    <!--rnb-output-begin {base64}-->
      ...
    <!--rnb-output-end -->
    <!--rnb-plot-begin {base64}-->
      <img src=... />
    <!--rnb-plot-end -->
  <!--rnb-chunk-end -->  
  ```
* tags cannot be nested
* a `chunk` may hold an arbitrary number of `outputs`
* tags hold data as base64 encoded json dictionaries as follows:
  * rnb-source-begin: 
    ~~~
    {'data': '```python\n chunk as markdown```'}
    ~~~
  * rnb-output-begin/rnb-plot-begin
    ~~~
    {'data': '<plain text representation of output>',   # fallback for rstudio
     'ipymd.data': {'text/plain': ...,                  # output['data'] from jupyter nbformat
                    'image/png': ...,
                    ... },
     'ipymd.metadata': {},                              # output['metadata'] from jupyter nbformat
     'ipymd.output_type': 'display_data'}               # output['output_type'] from jupyter nbformat
    ~~~


### Markdown (`.md`)
* By convention, a **notebook code cell** is equivalent to a **Markdown code block with explicit `python` syntax highlighting**:

  ```
  >>> print("Hello world")
  Hello world
  ```

* **Notebook metadata** can be specified in [YAML](http://yaml.org/) inside
  Jekyll-style [front-matter](http://jekyllrb.com/docs/frontmatter/) dashes
  at the beginning of a document:

  ```markdown
  ---
  kernelspec:
    name: some-non-native-kernel
  ---

  First cell content
  ```

  Native kernel metadata will be elided by default: non-python kernels haven't
  been tested yet, but support is planned.

* **Cell metadata** is specified with YAML stream documents with dashes and
  periods, such as to create slides:

  ```markdown
  # Previous slide

  ---
  slideshow:
    slide_type: slide
  ...

  # Some Slide Content
  ```

  > NOTE: You probably shouldn't use `---` to mean an `<hr/>`: `***`
  could be a suitable substitute.

* Null metadata (i.e. splitting a markdown cell) can be created with just
  three dashes. This is useful when adding slideshow notes or skipped cells.

  ```markdown
  A cell

  ---

  Another cell
  ```

* The back-and-forth conversion is not strictly the identity function:
    * Extra line breaks in Markdown are discarded
    * Text output and standard output are combined into a single text output (stdout lines first, output lines last)



### O'Reilly Atlas (`.md`)
* `.md` with special HTML tags for code and mathematical equations
[Format documentation](http://odewahn.github.io/publishing-workflows-for-jupyter/#1)  (

### Python (`.py`)
* code cells are delimited by double line breaks.
* Markdown cells = Python comments. 
* [TODO: this doesn't work well, see #28 and #31]

### Opendocument (`.odt`).
* You need to install the [development version of odfpy](https://github.com/eea/odfpy/).



## Implementing your own format
ipymd uses a modular architecture that lets you define new formats. The following formats are currently implemented, and can be selected by modifying `~/.ipython/profile_<whichever>/ipython_notebook_config.py`:

You can convert from any supported format to any supported format. This works by converting to an intermediate format that is basically a list of notebook cells.

### ipymd cells

An **ipymd cell** is a Python dictionary with the following fields:

* `cell_type`: `markdown`, `code` or `notebok_metadata` (if implemented)
* `input`: a string with the code input (code cell only)
* `output`: a string with the text output and stdout (code cell only)
* `source`: a string containing Markdown markup (markdown cell only)
* `metadata`: a dictionary containing cell (or notebook) metadata

### Kernel Metadata

By default, notebook metadata for the native kernel (usually `python2` or
`python3`) won't be written to markdown. Since ipymd doesn't yet support other
kernels, this doesn't matter much, but if you would like to pick a non-native
python kernel to be interpreted as the default for ipymd, and store
`kernelspec` and `language_info` for the other, you can add this to your
`ipython_notebook_config.py` file:
  * `c.IPymdContentsManager.default_kernel_name = 'python2'`

Or, to always remember all notebook-level metadata:
  * `c.IPymdContentsManager.verbose_metadata = True`

### Customize the Markdown format

You can customize the exact way the notebook is converted from/to Markdown by deriving from `BaseMarkdownReader` or `MarkdownReader` (idem with writers). Look at `ipymd/formats/markdown.py`.

### Implement your own format

You can also implement your own format by following these instructions:

* Create a `MyFormatReader` class that implements:
    * `self.read(contents)`: yields ipymd cells from a `contents` string
* Create a `MyFormatWriter` class that implements:
    * `self.write(cell)`: append an ipymd cell
      * (optional) `self.write_notebook_metadata(cell)`: write the notebook
        metadata dictionary
    * `self.contents`: return the contents as a string

* To activate this format, call this at Notebook launch time (not in a kernel!), perhaps in your `ipython_notebook_config.py`:

```python
  from ipymd import format_manager
  format_manager().register(
      name='my_format',
      reader=MyFormatReader,
      writer=MyFormatWriter,
      file_extension='.md',  # or anything else
      file_type='text',  # or JSON
  )
```

* Now you can convert contents: `ipymd.convert(contents, from_='notebook', to='my_format')` or any other combination.

### Contributing a new ipymd format
* To further integrate your format in ipymd, create a `ipymd/formats/my_format.py` file.
* Put your reader and writer class in there, as well as a top-level variable:

```python
  MY_FORMAT = dict(
      reader=MyFormatReader,
      writer=MyFormatWriter,
      file_extension='.md',
      file_type='text',
  )
```

* In `setup.py`, add this to `entry_points`:

```python
      ...
      entry_points={
          'ipymd.format': [
              ...
              'my_format=myformat:MY_FORMAT',
              ...
          ]
      }
```

  > Note that the `entry_point` name will be used by default. you may override
    it, if you like, but Don't Repeat Yourself.

* Add some unit tests in `ipymd/formats/tests`.
* Propose a PR!

Look at the existing format implementations for more details.


### Packaging a format
* If you want to be able to redistribute your format without adding it to ipymd proper (i.e. in-house or experimental), implement all your code in a real python module.
* Someplace easy to import, e.g. `myformat.py` or `myformat/__init__.py`, add:

```python
  MY_FORMAT = dict(
      reader=MyFormatReader,
      writer=MyFormatWriter,
      file_extension='.md',  # or anything else
      file_type='text',  # or JSON
  )
```

  and this to your `setup.py`:

```python
  ...
      entry_points={
          'ipymd.format': [
              'my_format=myformat:MY_FORMAT',
              ],
          },
  ...
```

  * Publish on pypi!
  * Your users will now be able to `pip install myformat`, then configure their Notebook to use your format with the name `my_format`.
  
