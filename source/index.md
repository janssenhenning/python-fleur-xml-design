---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.11.4
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---

<!-- Python Fleur XML design documentation master file, created by
     sphinx-quickstart on Fri Mar 11 16:07:58 2022.
     You can adapt this file completely to your liking, but it should at least
     contain the root `toctree` directive. -->


# Python API for Fleur XML files

```{toctree}
   :maxdepth: 2
```

This site aims to explain the design of the current functionality for working with `inp.xml` and `out.xml` files in both `masci-tools` and `aiida-fleur`

Ultimately this document explains the background behind the following code blocks for loading information from `.xml` files

`````{tabs}

````{group-tab} AiiDA

   ```python
   from aiida import plugins

   FleurinpData = plugins.DataFactory('fleur.fleurinp')
   fleurinp = FleurinpData('inp.xml')

   structure = fleurinp.get_structuredata()
   lapw_parameters = fleurinp.get_parameterdata()
   ```
````
````{group-tab} non-AiiDA

   ```python
   from masci_tools.io.io_fleurxml import load_inpxml
   from masci_tools.util.xml.xml_getters import get_structure_data, get_parameter_data

   xmltree, schema_dict = load_inpxml('inp.xml')

   structure_information = get_structure_data(xmltree, schema_dict)
   lapw_parameters = get_parameter_data(xmltree, schema_dict)
   ```
````
`````

and modifying `.xml` files

`````{tabs}

````{group-tab} AiiDA

   ```python
   from aiida_fleur.data.fleurinpmodifier import FleurinpModifier

   modifier = FleurinpModifier(fleurinp)
   modifier.set_inpchanges({'kmax': 5.0, 'l_soc': True})
   modifier.set_species('all-Fe', {
                           'cutoffs': {'lmax': 18}
                        },
                        filters={'species':{
                           'mtSphere': {'radius': {'>': 2.0}}
                        }})

   fleurinp = modifier.freeze()
   ```
````
````{group-tab} non-AiiDA

   ```python
   from masci_tools.io.fleurxmlmodifier import FleurXMLModifier

   modifier = FleurXMLModifier()
   modifier.set_inpchanges({'kmax': 5.0, 'l_soc': True})
   modifier.set_species('all-Fe', {
                           'cutoffs': {'lmax': 18}
                        },
                        filters={'species':{
                           'mtSphere': {'radius': {'>': 2.0}}
                        }})

   xmltree, _ = modifier.modify_xmlfile('inp.xml')
   ```
````
`````

```{note}
This document does not try to explain the AiiDA framework itself but only the parts relevant
for the XML functionality. For more information on AiiDA refer to it's [documentation](https://aiida.readthedocs.io/projects/aiida-core/en/latest/)
```

## Problem

The main Fleur input/output files are in a XML format since version `v27`. This provides
external tools with the ability to retrieve information from these or modify the input
file in a well defined manner.

However, changes in the structure of these XML formats produces a non-trivial amount of
maintenance that needs to happen in external packages. Especially for the AiiDA plugin for
the Fleur code was required to do a release with a lot of small compatibility changes for each
release of the code (TODO: provide example), limiting the ability to support more Fleur releases with the same version
of the plugin.

The XML functionality in `masci-tools` aims to provide a way to centralize this effort and making the extension and implementation of new features easier.

## Design principles

- The previous implementation of XML file parsers in `aiida-fleur` made use of XML `XPath` to
  retrieve information from XML files. For performance reasons these are always absolute paths
  completely definining the location of the desired information. The use of absolute XPaths
  should also be preferred in the new implementation
- Users should not be required to know the complete `XPath` to be able to retrieve
  information, but ideally and if possible only the name of the last node in the path.
  The complete Path would already special knowledge and new functionality should be
  much closer to the way `set_inpchanges` works
- Maintenance for including changes in the XML file structure with new Fleur releases should
  be minimal
- All AiiDA functionality directly manipulating/reading XML files should have an equivalent
  way of doing the same thing without AiiDA. `aiida-fleur` should only provide a light wrapper
  adding AiiDA specific functionality around the functions in `masci-tools` for these cases.
  This not only reduces the barrier of entry for people experimenting with it since no large
  AiiDA environment has to be set up. It also enables the reusage of this code for adding
  Fleur IO capability to other packages

## Fleur XML file structure

The structure of the Fleur XML files is defined in so called XML schema files. There are two
files

- `FleurInputSchema.xsd` defines the complete structure of the `inp.xml`
- `FleurOutputSchema.xsd` defines the complete structure of the `out.xml`

```{note}
Since the `out.xml` contains a copy of the used input to make it useful as a standalone file
the `FleurOutputSchema.xsd` also explicitly includes the `FleurInputSchema.xsd`
```

The structure of the XML files is very specific.
- A large amount of the data is stored in XML attributes
- The `out.xml` file makes more use of the text for XML tags.
- The XML files are non-recursive (with the exception of the `timing` section in the `out.xml`)
- The data types in the XML are limited to:
   - `switches`: Booleans expressed as either `T` or `F` strings
   - `strings`
   - `integers`
   - `floats`
   - `mathematical expressions`: Simple strings that Fleur can evaluate
   - `complex numbers`

## Loading XML files

The first step in doing anything with the Fleur XML files is parsing them into a datastructure
in the python runtime. For this we use the `lxml` library. This library provides the very
popular `ElementTree` API originally from the `python stdlib` and has complete support for
`XPath1.0` and `XInclude`. The latter is used to split up input files into more digestable chunks for users.
In addition it uses the `libxml2` library under the hood, which is the same library used by
the Fleur code itself.

```{admonition} Using lxml for untrusted input

For XML files from untrusted sources special care must be taken to avoid security problems.
A general guide for using `lxml` in these cirumstances can be found [here](https://lxml.de/FAQ.html#how-do-i-use-lxml-safely-as-a-web-service-endpoint)

Some features used by the Fleur XML files have to be limited in these cases:
- `XInclude`: Can in principle retrieve files from other sources on the web

At the moment there are only a couple measures taken to make the API in `masci-tools` safer
but it is definitely not safe for use directly as a web service
```

Example of parsing XML files with `lxml`

```{literalinclude} test.xml
   :language: xml
   :caption:
```

```{code-cell} ipython3
from lxml import etree

xmltree = etree.parse('test.xml')

#The xmltree now contains the complete informationg from the XML file
root = xmltree.getroot()
print(f'Root Tag: {root.tag}')

text = [node.text for node in root.findall('child')]
print(f"Text of 'child' elements: {text}")

text = root.xpath('//child/text()')
print(f"Text of 'child' elements: {text}")
```

The code cell above uses two equivalent methods of searching the XML tree. First we use
a method of the `ElementTree` API called `findall` which searches for tags with the given
name. In the second way we use a the called [XPath](https://www.w3schools.com/xml/xpath_syntax.asp) standard for specifying a path through the XML tree simlar to a file
system path. 

The latter is used in the majority of cases in the following implementations. It allows us
to be very specific with what we want to find and it allows for more complex syntax to
[filter the results](#filtering-results). It also allows us to express a unique location
in the XML file with a single string, enabling us to define mappings between the locations
in the XML file and their attached properties. Further as long as a XML file is not
recursive, i.e. a tag cannot contain itselve, the number of the unique locations defined by
absolute XPaths without complex syntax is finite.
## Using the XML schema file

The XML Schema files provide a great way of programmatically discovering the structure of the
`.xml` files and the types of the used attributes/text. The Schema files can be used in two
ways by the `lxml` library. First an `etree.XMLSchema` object can be constructed to validate
given XML files against this schema.

```{code-cell} ipython3
from lxml import etree

schema = etree.XMLSchema(file='schemas/FleurInputSchema.xsd')

xmltree = etree.parse('example_files/inp_valid.xml')
print(f'Is this file valid? {schema(xmltree)}')
```

```{code-cell} ipython3
from lxml import etree

schema = etree.XMLSchema(file='schemas/FleurInputSchema.xsd')

xmltree = etree.parse('example_files/inp_invalid.xml')
print(f'Is this file valid? {schema(xmltree)}')
```

Or the file can be used like a normal XML file. The following example retrieves how many
different complex types are defined in the schema.

```{note}
In Schema files all tags are prefixed with a special namespace `xsd`
```

```{code-cell} ipython3
from lxml import etree

xmltree = etree.parse('schemas/FleurInputSchema.xsd')

namespaces = {'xsd': 'http://www.w3.org/2001/XMLSchema'}
n_types = xmltree.xpath('count(/xsd:schema/xsd:complexType)', namespaces=namespaces)

print(f'The input schema has {int(n_types)} complex types')
```
We employ the second way of handling XML Schemas to discover their structure. The 
found information is stored in dictionary like structures called `InputSchemaDict` and `OutputSchemaDict`

```{tikz} General structure of extracting information from XML Schema files. Blue nodes each refer to small functions working on the XML tree of the schema directly
   :include: tikz/schema_structure.tikz
   :align: center
```

```{admonition} Performance implications
   Looking at this structure one problem you might spot is that we dramatically increased the number
   of operations needed, just to get the path to a given tag. Previously the path would just be hardcoded
   meaning no performance impact at all. With the new implementation a XML file maybe larger than the actual
   input file has to be parsed before gaining access to e.g. the path to a tag.

   To manage the impact of this there are multiple layers of reusing results, i.e. caching in the construction
   of the `SchemaDict` objects.

   1. The individual functions parsing the schema file use a lot of similar `XPath` expressions over and over again (on the order of ~1000 queries).
      The results of these queries are cached
   2. If a `SchemaDict` is constructed using the `fromVersion` method the SchemaDict is only constructed on the first
      run. On subsequent calls with the same version string the cached object is returned
```

The `load_inpxml` and `load_outxml` functions in `masci_tools.io.io_fleurxml` provide this schema dictionary together with the
parsed xml file just by giving the filepath to a given XML file. The `aiida-fleur` `FleurinpData` class has a wrapper method
for loading and validating `inp.xml` files, which is called, when any instance is instanitated with a file named `inp.xml`.

All functionality in the following sections is built on top of these objects to remove the need for hardcoding the structure of the
Fleur XML files in multiple places

### Selecting XPaths
Below is a simple example of getting the complete Xpath for a given tag name
```{code-cell} ipython3
from masci_tools.io.parsers.fleur_schema import InputSchemaDict

schema_dict = InputSchemaDict.fromVersion('0.35')
print(schema_dict.tag_xpath('xcfunctional'))
```

```{code-cell} ipython3
from masci_tools.io.parsers.fleur_schema import InputSchemaDict

schema_dict = InputSchemaDict.fromVersion('0.31')
print(schema_dict.tag_xpath('xcfunctional'))
```

```{note}
   The tag name given in the method is case-insensitive `XCFUNCTIONAL` would also work
```

### Working with types

Accessing the collected types of attribute/text

```{code-cell} ipython3
from masci_tools.io.parsers.fleur_schema import InputSchemaDict

schema_dict = InputSchemaDict.fromVersion('0.35')
print(schema_dict['attrib_types']['alpha'])
print(schema_dict['text_types']['kpoint'])
```

`masci_tools.util.xml.converters` provides functions for robustly using these types

```{code-cell} ipython3
from masci_tools.io.parsers.fleur_schema import InputSchemaDict
from masci_tools.util.xml.converters import convert_from_xml, convert_to_xml

schema_dict = InputSchemaDict.fromVersion('0.35')
print(convert_from_xml('16.500000000', schema_dict, 'alpha'))
print(convert_from_xml('16.500000000*Pi', schema_dict, 'alpha')) #mathematical expressions

print(convert_to_xml([0.0,1.0,2.0], schema_dict, 'kpoint', text=True))
```
```{note}
The boolean second return value indicates whether the conversion was successful
```
## Retrieving information from XML files

There are two levels of functions building on top of the schema functionality for retrieving
information from XML files

- Functions, that retrieve and convert values for one given tag/attribute from the xml file
   - `evaluate_attribute`, `evaluate_text`, `evaluate_tag`, `evaluate_parent_tag`, `evaluate_single_value`, 
     `tag_exists`, `attrib_exists`, `get_number_of_nodes`
- Functions, that return a collection of values for a clearly defined property of the calculation
   - `get_fleur_modes`: Dictionary with general switches and numbers identifying the _kind_ of Fleur calculation
   - `get_cell`: Return the braivais matrix and periodicity
   - `get_structure_data`: Return the atom position, symbols, braivais matrix and periodicity
   - `get_parameter_data`: Return dict with additional LAPW parameters (cutoffs, ...)
   - `get_kpoints_data`: Get Kpoint information in arrays
   - `get_nkpts`: Get the numnber of kpoints used in the calculation (mainly for optimizing parallelization)
   - `get_special_kpoints`: Get labelled kpoints in a kpoint set
   - `get_symmetry_information`: Get the symmetry operations in the calculation

The second set of functions uses the functions above to avoid hardcoding `XPath`

### Usage

### Relative XPaths

### XML getter Versioning
## Modifying XML files

When modifying the XML files more care has to be taken. The changes made could leave the
XML file in a state inconsistent with the XML schema if the modification is done in multiple
steps. In the previous implementation this was solved in `aiida-fleur` by bundling multiple
changes together in the `FleurinpModifier` class, which are then executed together and changes
are only validated at the end.

This approach is also taken in the new implementation making use of the `SchemaDict` 
functionalities. However, we introduce a much more strict separation of concerns of the
underlying functions modifying the XML tree to make extension and reasoning about
modifications easier. The three main categories of functions are shown below.

```{tikz} Hierachy of XML setters. The width indicates that most if not all functionality should be introduced in the higher levels and reuse code in the levels below extensively
   :include: tikz/xml_setter_hierachy.tikz
   :align: center
```

These categories are mainly identified by their function signature of their first arguments:

- `Basic` setter functions: These take the XML tree and an explicit `XPath` expression and no `SchemaDict`
- `Intermediate` setter functions: These take the XML tree, the corresponding `SchemaDict` and two `XPath` expressions. One is a `XPath` that is "simple", i.e. it is absolute and has no predicates (conditions in `[]`). The second `XPath` is unrestricted but has to produce a subset of the results the "simple" `XPath` produces
- `Name` setter functions: These take the XML tree, the corresponding `SchemaDict` and require no `XPath` expression. They can however take a `XPath` expression as an optional argument

When implementing XML setters it is allowed to call different functions on the same level of
modification function or a level below (according to the pyramid). It is never allowed to go
back up in the levels. The reason for this is that each higher level provides more
functionality that cannot be reproduced consistently when going back up. For example the
`Intermediate` functions are allowed to create nested tags if they are not present in the
XML file. This is possible since these functions have access to the `SchemaDict` to make sure
that the order of tags is always correct and no invalid tags are created. The `Basic` functions
have no guarantees for this since they do not have access to the `SchemaDict`

## Filtering Results

## Predefined complete file Parsers

## Error handling

## Indices and tables


* {ref}`genindex`
* {ref}`modindex`
* {ref}`search`
