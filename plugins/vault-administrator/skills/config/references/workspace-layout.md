# Config workspace layout

The directory `vault config pull` writes. Any skill that needs to know how a Vault is
configured reads from here before it calls the API.

```
<workspace>/
  components/
    <ComponentType>/
      <name>/
        <name>.mdl               the component's RECREATE statement, its subcomponents inline
                                 or each in a folder of its own (Field/, Picklistentry/, ...):
                                 the two layouts in ${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md
        <attr>.xml|txt           text lifted out of the MDL, beside the file that names it
                                 (rule, step_detail, page_markup); .txt in some workspaces only
  javasdk/                       Java SDK source by package path, one file per class
  custom_pages/                  Custom Page distributions and code
  .vault/
    dependencies.json            in some workspaces only; read relationships.json instead
    relationships.json           { "<Type>.<name>": { "label", "relationships": [ { "blocking_type",
                                 "target_type", "target_name", ... } ] } }
    metadata/<ComponentType>.json
                                 type schema: attributes, subcomponents, datatypes
  .vdxignore                     globs the CLI leaves alone on pull and push
```

Component type names are the Metadata Definition Language (MDL) names (`Object`, `Picklist`, `Doclifecycle`, `Workflow`,
`Pagelayout`, ...), the same strings the component-types metadata endpoint returns in
`name`. Objects are under `components/Object/<object_name>/`. `<object_name>.mdl` holds the
object-level attributes, and the fields are either inline in it or split out under `Field/`, one
folder each (the two layouts in `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`).
A list attribute keeps the form the pull wrote it in.
The CLI writes `.vault/metadata/<ComponentType>.json` only for a type with at least one component, and
none for a schemaless type; a type with no components has no directory under `components/`.

## Which Vault it came from

Nothing the pull writes names the source Vault or the pull time. The DNS lives in the first
commit's message (SKILL.md Step 2). The modified time of `.vault/relationships.json` is the last
pull that changed it, which can be earlier than the last pull. No git history, or a first commit that names no DNS, means ask the
user which Vault it was pulled from.

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
