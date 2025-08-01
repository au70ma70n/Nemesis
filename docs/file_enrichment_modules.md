## Adding File Enrichment Modules

File enrichment modules for the main enrichment workflow are located in [libs/file_enrichment_modules/file_enrichment_modules/](https://github.com/SpecterOps/Nemesis/tree/main/libs/file_enrichment_modules).

To add a new module, create a new folder matching Python's [PEP8 naming conventions](https://peps.python.org/pep-0008/#package-and-module-names):
>Modules should have short, all-lowercase names. Underscores can be used in the module name if it improves readability. Python packages should also have short, all-lowercase names, although the use of underscores is discouraged.

Create a main `analyzer.py` file with your enrichment logic. The easiest method for this (and enrichment modules are fairly small) is to find an example module, and use it as a base with a LLM to help draft your code.

If your module needs additional dependencies, you have two options. Before either, first [install Poetry](https://python-poetry.org/).

For the first option, you can `cd` to `projects/file_enrichment` or `libs/file_enrichment_modules/` and run `poetry add X` for the needed library.

Alternatively (and easier) you can create a `pyproject.yaml` in the new module module folder. An example is:

```toml
[tool.poetry]
name = "module"
version = "0.1.0"
description = "Enriches things"
authors = ["harmj0y <will@harmj0y.net>"]
package-mode = false

[tool.poetry.dependencies]
python = "^3.9"
```

Then in this folder, run `poetry add X` to add a new library. The dynamic module loader will install the necessary dependencies in a Poetry env for just that module.

## Tips / Tricks

The `should_process()` function determines if the module should run on a file. You can either check the name or any other component of the base enriched file with `file_enriched = get_file_enriched(object_id)`:

```python
...
def should_process(self, state_key: str) -> bool:
    """Determine if this module should run based on file type."""
    file_enriched = get_file_enriched(state_key)
    # Check if file appears to be a VNC config file
    should_run = (
        file_enriched.file_name.lower().endswith(".ini")
        and "vnc" in file_enriched.file_name.lower()
        and "text" in file_enriched.magic_type.lower()
    )
    logger.debug(f"VncParser should_run: {should_run}, magic_type: {file_enriched.magic_type.lower()}")
    return should_run
...
```

Or you can use a Yara rule (or you could do both!):

```python
...
    # Yara rule to check for DPAPI blob content
    self.yara_rule = yara_x.compile("""
rule has_dpapi_blob
{
    strings:
        $dpapi_header = { 01 00 00 00 D0 8C 9D DF 01 15 D1 11 8C 7A 00 C0 4F C2 97 EB }
        $dpapi_header_b64_1 = "AAAA0Iyd3wEV0RGMegDAT8KX6"
        $dpapi_header_b64_2 = "AQAAANCMnd8BFdERjHoAwE/Cl+"
        $dpapi_header_b64_3 = "EAAADQjJ3fARXREYx6AMBPwpfr"
    condition:
        $dpapi_header or $dpapi_header_b64_1 or $dpapi_header_b64_2 or $dpapi_header_b64_3
}
    """)

def should_process(self, object_id: str) -> bool:
    file_enriched = get_file_enriched(object_id)
    if file_enriched.size > self.size_limit:
        logger.warning(
            f"[dpapi_analyzer] file {file_enriched.path} ({file_enriched.object_id} / {file_enriched.size} bytes) exceeds the size limit of {self.size_limit} bytes, only analyzing the first {self.size_limit} bytes"
        )

    num_bytes = file_enriched.size if file_enriched.size < self.size_limit else self.size_limit
    file_bytes = self.storage.download_bytes(file_enriched.object_id, length=num_bytes)

    should_run = len(self.yara_rule.scan(file_bytes).matching_rules) > 0
    logger.debug(f"[dpapi_analyzer] should_run: {should_run}")
    return should_run
...
```

## On Transforms

File transforms require a `type` (used as a title for display) and an `object_id` to reference the data to display.

Optional metadata is:

| Metadata Field            | Type         | Description                                                            |
| ------------------------- | ------------ | ---------------------------------------------------------------------- |
| file_name                 | string       | Name of the file (i.e., for downloads)                                 |
| display_type_in_dashboard | display_type | How to display in the dashboard                                        |
| display_title             | string       | Title to display for the transform in the dashboard                    |
| default_display           | bool         | `true` to set this transform as the default display                    |
| offer_as_download         | bool         | If set to `true` offered as a download tab, downloading as `file_name` |

Display Types are:

| Value    | Description                                                                                           |
| -------- | ----------------------------------------------------------------------------------------------------- |
| monaco   | Display in a Monaco editor, using the extension from `file_name` to help determine the language type. |
| pdf      | Render as a PDF                                                                                       |
| image    | Render as an image                                                                                    |
| markdown | Render as an image                                                                                    |
| null     | Don't display content                                                                                 |

### Examples

Example of setting a text file as the default display (in file_enrichment_modules/sqlite/analyzer.py):

```python
with tempfile.NamedTemporaryFile(mode="w", encoding="utf-8") as tmp_display_file:
    display = format_sqlite_data(database_data)
    tmp_display_file.write(display)
    tmp_display_file.flush()

    object_id = self.storage.upload_file(tmp_display_file.name)

    displayable_parsed = Transform(
        type="displayable_parsed",
        object_id=f"{object_id}",
        metadata={
            "file_name": f"{file_enriched.file_name}.txt",
            "display_type_in_dashboard": "monaco",
            "default_display": True
        },
    )
enrichment_result.transforms = [displayable_parsed]
```

Example of offering a file for download (in file_enrichment_modules/dotnet/analyzer.py):

```python
decompilation = Transform(
    type = "decompilation",
    object_id = service_results["decompilation"]["object_id"],
    metadata = {
        "file_name" : f"{file_enriched.file_name}.zip",
        "offer_as_download" : True
    }
)
enrichment_result.transforms = [decompilation]
```
