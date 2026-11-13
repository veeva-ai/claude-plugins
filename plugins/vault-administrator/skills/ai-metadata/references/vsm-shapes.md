# Vsm component shapes

Copy a shape from the user's own pull when it has one, and use these when it does not.

## Layout

```
components/Vsmentity/employee__c/employee__c.mdl
components/Vsmentity/employee__c/Vsmfield/record_name__c/record_name__c.mdl
components/Vsmentity/employee__c/Vsmrelationship/manager__c/manager__c.mdl
components/Vsmentity/employee__c/Vsmmetric/leave_days_remaining__c/leave_days_remaining__c.mdl
components/Vsmpicklist/employment_status__c/employment_status__c.mdl
components/Vsmpicklist/employment_status__c/Vsmvalue/on_leave__c/on_leave__c.mdl
```

A pull rewrites a pushed file in Vault's own form: it sorts the subcomponent references, and it replaces `target_label` with the target's real label. Copy `target_label` from the target's own file.

## Lengths

| Attribute | Max |
| --- | --- |
| Entity `description` | 1,000 |
| Field, relationship, metric, picklist and value `description` | 500 |
| Metric `instructions` | 2,000 |
| `label` | 60 |
| `target` | 150 |
| `target_label` | 200 |

`description` is required on all six types, and so are a metric's `instructions` and `target_fields`. Read the user's own `.vault/metadata/` files when they differ from this table; they are the Vault's answer.

Long descriptions in the examples below are cut at `...`.

## Entity

```
RECREATE Vsmentity employee__c (
   label('Employee'),
   active(true),
   description('A person the company employs, as HR tracks them ... It is not Person (person__sys), which tracks any individual whether or not they work here ...'),
   synonyms('staff',
      'staff member',
      'worker'),
   target('Object.employee__c'),
   target_label('Employee'),
   Vsmfield(@Vsmfield/employment_status__c.mdl),
   Vsmfield(@Vsmfield/record_name__c.mdl),
   Vsmmetric(@Vsmmetric/leave_days_remaining__c.mdl),
   Vsmrelationship(@Vsmrelationship/manager__c.mdl)
);
```

## Field on a standard target

The subcomponent name is `__c`; the target is the standard field. A `name__v` subcomponent name fails with GEN-FADDS-ONA-1019.

```
Vsmfield record_name__c (
   label('Name'),
   active(true),
   description('The employee''s full name, first and last. Use it to find one employee by name.'),
   example_values(),
   synonyms('full name',
         'employee name'),
   target('name__v'),
   target_label('Full Name'),
   vsm_picklist(),
   vsm_picklist_local()
);
```

## Field on a picklist that has its own Vsmpicklist

```
Vsmfield employment_status__c (
   label('Employment Status'),
   active(true),
   description('Whether the person works here today: Active, On Leave or Terminated. Use it for headcount and for who has left. The record Status does not say this.'),
   example_values(),
   synonyms('HR status',
         'employment state'),
   target('employment_status__c'),
   target_label('Employment Status'),
   vsm_picklist('Vsmpicklist.employment_status__c'),
   vsm_picklist_local()
);
```

## Relationship

```
Vsmrelationship manager__c (
   label('Manager'),
   active(true),
   cardinality('MANY_TO_ONE'),
   description('The employee''s line manager, another Employee record. Use it for who someone reports to (manager__cr.name__v) ...'),
   join_object(),
   synonyms('line manager',
         'supervisor',
         'reports to'),
   target('manager__cr'),
   target_label('Manager'),
   target_object('employee__c')
);
```

Delivered relationships spell `join_object` two ways (`'language_locale__sys'` and `'Object.legalhold_item__v'`); which one a custom many-to-many takes is not confirmed.

## Metric

```
Vsmmetric leave_days_remaining__c (
   label('Leave Days Remaining'),
   active(true),
   description('Days of leave allowance an employee has left in the current calendar year after approved PTO requests. Use it for how many days off someone has left. Draft and rejected requests do not count.'),
   instructions('Take leave_allowance_days__c on the employee. Subtract the sum of days_requested__c on that employee''s PTO requests (pto_requests__cr) where state__v is approved_state__c and start_date__c falls in the current calendar year.'),
   synonyms('days left',
         'remaining PTO',
         'leave balance'),
   target_fields('id',
         'leave_allowance_days__c')
);
```

## Picklist and value

```
RECREATE Vsmpicklist employment_status__c (
   label('Employment Status'),
   active(true),
   description('Whether a person still works for the company ... Short time off is a PTO request, not On Leave.'),
   synonyms('HR status',
      'employment state'),
   target('Picklist.employment_status__c'),
   target_label('Employment Status'),
   Vsmvalue(@Vsmvalue/active__c.mdl),
   Vsmvalue(@Vsmvalue/on_leave__c.mdl),
   Vsmvalue(@Vsmvalue/terminated__c.mdl)
);
```

```
Vsmvalue on_leave__c (
   label('On Leave'),
   active(true),
   description('Employed but away on an extended leave of absence, such as parental or medical leave, and still counted as an employee ...'),
   synonyms('leave of absence',
         'LOA'),
   target('on_leave__c'),
   target_label('On Leave')
);
```

An object lifecycle picklist targets `Objectlifecycle.<name>` and its values the bare state names. A document lifecycle can carry picklist metadata (RN AI, 26R2.3), but its target form is not yet confirmed, so it is not shown.

## Apostrophes

A drafted string writes an apostrophe as two single quotes (`employee''s`). Delivered files carry `\''`.
