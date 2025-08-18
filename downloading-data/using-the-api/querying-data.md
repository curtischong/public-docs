---
description: >-
  An introduction on how to query for data with the Materials Project API
  client.
---

# Querying Data

Materials Project data can be queried through a specific (list of) [**Materials Project ID(s)**](../../frequently-asked-questions.md#what-is-a-task_id-and-what-is-a-material_id-and-how-do-they-differ), and/or through **property filters (e.g. band gap less than 0.5 eV).**

Most material property data is available as [**summary data**](https://materialsproject.github.io/api/_autosummary/mp_api.client.routes.materials.summary.SummaryRester.html#mp_api.client.routes.materials.summary.SummaryRester.search) for a specific material. **To query summary data with Materials Project IDs** the `search` method should be used:

```python
with MPRester("your_api_key_here") as mpr:
    docs = mpr.materials.summary.search(
        material_ids=["mp-149", "mp-13", "mp-22526"]
    )
```

The above returns a list of `MPDataDoc` objects with data accessible through their attributes. For example, the Material ID and formula can be obtained for a particular document with:

```python
example_doc = docs[0]
mpid = example_doc.material_id
formula = example_doc.formula_pretty
```

A list of available property fields can be obtained by examining one of these objects, or from the `MPRester` with:

```python
list_of_available_fields = mpr.materials.summary.available_fields
```

**To query summary data with property filters** use the `search` function as well. For example, below is a query to find materials containing `Si` and `O` that have a band gap greater than `0.5 eV` but less then `1.0 eV`.

```python
with MPRester("your_api_key_here") as mpr:
    docs = mpr.materials.summary.search(
        elements=["Si", "O"], band_gap=(0.5, 1.0)
    )
```

> _**NOTE:**_ The `available_fields` attribute is meant to refer to the data available from the endpoint, not necessarily which fields you can use to query that data via `search()`. See the `search()` docstrings [the client docs](https://materialsproject.github.io/api/_autosummary/mp_api.client.routes.html) for supported arguments. The section on [Advanced Usage](advanced-usage.md) might be helpful here, too.

**Note that by default ALL available property data within `MPDataDoc` objects will be populated.** If one is only interested in a few properties, limiting what data is returned will speed up data retrieval. Pass a list of the fields you are interested in to `fields` to accomplish this. For example, if we were only interested in `material_id`, `band_gap`, and `volume` for the materials from the above query, we could instead use:

```python
with MPRester("your_api_key_here") as mpr:
    docs = mpr.materials.summary.search(
        elements=["Si", "O"], band_gap=(0.5, 1.0),
        fields=["material_id", "band_gap", "volume"]
    )
```

Now, only the `material_id`, `band_gap`, and `volume` attributes of the returned `MPDataDoc` objects will be populated. A list of available fields that were not requested will be returned in the `fields_not_requested` parameter.

```python
example_doc = docs[0]
mpid = example_doc.material_id       # a Materials Project ID
formula = example_doc.formula_pretty # a formula
volume = example_doc.volume          # a volume
example_doc.fields_not_requested     # list of unrequested fields
```



### Determine which functional was used to relax structures

The Materials Project contains structures relaxed with the PBE, PBE+U, and r<sup>2</sup>scan functionals. To determine which functional was used to calculation a given property, use the `origins` field of the summary documents, which points to the DFT calculation that generated a given property. You can include the `origins` field when searching for materials of interest, and use it to identify the `task_id`. Then, you can find the thermo document with the same `task_id` and use `run_type` to identify the functional. A run\_type of `GGA` indicates a PBE GGA calculations, `GGA_U` indicates a PBE+U calculation, and `r2SCAN` indicates an r<sup>2</sup>scan calculation.&#x20;

```
from mp_api.client import MPRester

mp_id_to_task_id = {}
with MPRester() as mpr:
    summary_docs = mpr.materials.summary.search(material_ids=["mp-149", "mp-13", "mp-22526"],
                                                fields = ["material_id", "structure", "origins"])

    for doc in summary_docs:
        for prop in doc.origins:
            if prop.name == "structure":
                mp_id_to_task_id[doc.material_id] = {
                    "task_id": prop.task_id,
                    "structure": doc.structure,
                }
                break

    thermo_docs = mpr.materials.thermo.search(material_ids=["mp-149", "mp-13", "mp-22526"],
                                              fields = ["material_id","entries"])

for doc in thermo_docs:
    mp_id = doc.material_id
    for entry in doc.entries.values():
        if entry.data["task_id"] == mp_id_to_task_id[mp_id]["task_id"]:
            mp_id_to_task_id[mp_id]["run_type"] = entry.parameters["run_type"]
            break
```

To query all entries from a chemical system calculated with a specific functional, specify <kbd>additional\_criteria</kbd> when using <kbd>get\_entries\_in\_chemsys</kbd>.

```
from mp_api.client import MPRester

chemsys = "Co-N"
with MPRester() as mpr:
    entries = mpr.get_entries_in_chemsys(chemsys, additional_criteria={"thermo_types": ["GGA_GGA+U","GGA_GGA+U_R2SCAN","R2SCAN"]})
```

### Other Data

Not all Materials Project data for a given material can be obtained from the summary API endpoint. To access the remaining data, other endpoints must be used. **Consult the** [**client docs**](https://materialsproject.github.io/api/_autosummary/mp_api.client.routes.html) **for a complete list of available endpoints/routes.**

For example, the `initial_structures` used in calculations producing data for a specific material can only be found in the `materials` [endpoint](https://materialsproject.github.io/api/_autosummary/mp_api.client.routes.materials.materials.html#module-mp_api.client.routes.materials.materials). To obtain the `initial_structures` for `mp-149`, the same `search` function can be used:

```python
with MPRester("your_api_key_here") as mpr:
    docs = mpr.materials.search(
        material_ids=["mp-149"], fields=["initial_structures"]
    )

example_doc = docs[0]
initial_structures = example_doc.initial_structures
```

Below is another example which uses property filters:

```python
with MPRester("your_api_key_here") as mpr:
    docs = mpr.materials.search(
        elements=["Si", "O"], num_sites=(0, 10),
        fields=["initial_structures"]
    )
                                              
example_doc = docs[0]
initial_structures = example_doc.initial_structures
```

The same querying procedure shown above can be applied to most endpoints. See [advanced usage](advanced-usage.md) and [examples](examples.md) for more information.

### Convenience Functions

In addition to `search` function, there are a small number of top level convenience functions for frequently made queries. These aim to provide a simpler interface, and make use of `search` under the hood. A couple examples of common queries include obtaining the structure or density of states of a particular material. **These convenience functions are illustrated in the** [**examples section**](examples.md)**.**
