# Dependency trace recipes

Run these from the workspace root, over the pulled files. Replace the names in angle brackets.
They read the pull, so they are as current as the pull.

## Report to report type to primary object

```bash
grep -n "report_type" components/Report/<report>/<report>.mdl
grep -n "primary_object" components/Reporttype/<type>/<type>.mdl
```

The first prints `report_type('Reporttype.<type>')`. The second prints the primary object. An
empty `report_type()` means there is no report type to follow, so stop there. A dotted primary
object on a MultiPass type, such as `english_definitions__c.glossary_definition__sys`, is
`<report>.<object>`: the report comes first.

## Every report on one report type

```bash
grep -ln "report_type('Reporttype.<type>')" components/Report/*/*.mdl
```

Run this before dropping or rebuilding a report type.

## Every report type that references a report

```bash
grep -ln "<report>\." components/Reporttype/*/*
```

A MultiPass type names the report in `primary_object` and in its join keys. Run this before
dropping a report.

## Every report type on one object

```bash
grep -ln "primary_object('<object>" components/Reporttype/*/*.mdl
```

## Relationships on a report type

```bash
grep -n "upRelationship \|downRelationship " components/Reporttype/<type>/configuration.xml
```

Each line gives a `key` for a Standard type. A MultiPass type uses `joinRelationship` instead:

```bash
grep -n "joinRelationship " components/Reporttype/<type>/configuration.xml
```

Each of those lines gives a `key`, a `joinKey` and a `joinType`, and joins reports.

## Dashboard to reports

```bash
grep -o 'report="Report\.[^"]*"' components/Dashboard/<dashboard>/dashboard_markup.xml | sort -u
```

## Which dashboards show one report

```bash
grep -ln 'report="Report.<report>"' components/Dashboard/*/dashboard_markup.xml
```

Run this before dropping or renaming a report.

## Fields a report uses

```bash
grep -o 'property="[^"]*"' components/Report/<report>/definition.xml | sort -u
```

Look each field up in the object's files before saying what it means.
