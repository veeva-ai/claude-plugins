# Config workspace layout

The directory `vault config pull` writes. Any skill that needs to know how a Vault is
configured reads from here before it calls the API.

```
<workspace>/
  components/
    <ComponentType>/
      <name>/
        <name>.mdl               the component's RECREATE statement
        <Subcomponent>/<subname>/<subname>.mdl
                                 one folder per subcomponent (Field/, Picklistentry/, ...)
        <attr>.js|json|xml|txt   embedded code or long text lifted out of the MDL, next to
                                 the .mdl that names it (rule.xml, step_detail.xml,
                                 page_markup.xml)
  javasdk/                       Java SDK source by package path, one file per class
  custom_pages/                  Custom Page distributions and code
  .vault/
    state.json                   md5 per tracked file plus provenance (below)
    dependencies.json            { "<Type>.<name>": { "depends_on": ["<Type>.<name>", ...] } }
                                 rebuilt on every pull
    metadata/<ComponentType>.json
                                 type schema: attributes, subcomponents, datatypes
  .vdxignore                     globs the CLI leaves alone on pull and push
```

Component type names are the Metadata Definition Language (MDL) names (`Object`, `Picklist`, `Doclifecycle`, `Workflow`,
`Pagelayout`, ...), the same strings the component-types metadata endpoint returns in
`name`. Objects are under `components/Object/<object_name>/`. The object's fields are split
out under `Field/`, one folder each, and `<object_name>.mdl` holds the object-level attributes.
A list attribute keeps the form the pull wrote it in.
The CLI writes `.vault/metadata/<ComponentType>.json` only for a type with at least one component, and
none for a schemaless type; a type with no components has no directory under `components/`.

## state.json

A flat JSON object, and a large one: a full pull tracks tens of thousands of files. Search it
for the key you need; do not read it whole. Every key that does not start with `__` is a
workspace-relative path mapped to the lowercase hex MD5 of that file's bytes. Reserved keys:

| Key | Meaning |
|---|---|
| `__version__` | The CLI version that last wrote the workspace. |
| `__source_env__` | The environment label the pull used. |
| `__source_dns__` | The Vault DNS the pull came from. The guard for "wrong Vault in this directory". |
| `__pull_mode__` | `advanced` for a CLI pull. |

## What the pull covers

Metadata-class components with their MDL parsed and embedded code lifted into sidecar files,
per-type metadata under `.vault/metadata/`, Java SDK source, Custom Pages, and the
dependency graph. A pull also writes over files you changed, and stops before it drops files
for components the Vault does not have. The workspace has no compare and no
way to pull one component type; a pull takes the whole Vault. Admin >
Settings are not components and are never in a pull. A component whose MDL Vault does not
return is listed as ERROR and never written, and the pull exits 1.

## When a file is missing

SKILL.md Step 4 maps questions to files. If the file is not there, the path may be in
`.vdxignore`, or the pull may predate the component. Run the freshness check from SKILL.md
Step 3 before trusting an old workspace, and pull again rather than guessing.
