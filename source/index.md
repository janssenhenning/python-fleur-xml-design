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


# Python API for Fleur XML files: Design

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
popular `ElementTree` API orgianllay from the `python stdlib` and has complete support for `XPath1.0` and `XInclude`. The latter is used to split up input files into more digestable chunks for users.
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

text = root.xpath('//child/text()')
print(f"Text of 'child' elements: {text}")
```

## Using the XML schema file

## XML getters

## Modifying XML files

## Predefined XML Parser

## Indices and tables


* {ref}`genindex`
* {ref}`modindex`
* {ref}`search`
