# Template Schemas [Version: 2023-09]

This document contains the formal specifications for the Open Job Description Template schemas. There are two schemas
defined in this document:

1. The Job Template describes the shape of a Job, its runtime environment, and all the processes that will run as part
of it. Jobs are created from Job Templates by providing a value for each of the Job Parameters defined in the Template,
and instantiating the template with those values.
2. The Environment Template describes an Environment that is defined outside a Job. This can be used in render management
systems to allow users to define Environments that are automatically applied to Job Templates that are submitted for rendering.

Reading the broad overviews in [How Jobs Are Constructed](How-Jobs-Are-Constructed) and [How Jobs Are Run](How-Jobs-Are-Run)
might provide an easier introduction and broader context prior to reading the schema in entirety.

Both Templates are expressed as UTF-8 documents in either
[ECMA-404 JavaScript Object Notation (JSON)](https://www.json.org/json-en.html) or
[YAML Ain't Markup Language (YAML) 1.2](https://yaml.org/) interchange format.

Throughout this document, a "character" is a single Unicode code point; all length limits expressed in characters
(e.g. "Maximum length: 128 characters") are counts of Unicode code points, not bytes.

Notations used in this document to annotate aspects of the schema definition:

* `@fmtstring` - The value of the annotated property is a Format String. See [Format Strings](#73-format-strings).
  * `@fmtstring[host]` - The value is evaluated at runtime on the worker host on which the job is running. The value is otherwise
     evaluated when the job template is submitted and the render manager is constructing a job.
* `@optional` - The annotated property is optional.
* `@extension EXTENSION_NAME` - The annotated property is available only when extension EXTENSION_NAME is requested
  in the `extensions` list of the template. Implementations can choose which extensions to support.
* `@incompatible PROPERTY_NAME` - The annotated property cannot be used in
combination with PROPERTY_NAME.

## 1. Root elements

### 1.1. Job Template

```yaml
specificationVersion: "jobtemplate-2023-09"
$schema: <string> # @optional
extensions: [ <ExtensionName>, ... ] # @optional
name: <JobName> # @fmtstring
description: <Description> # @optional
parameterDefinitions:  [ <JobParameterDefinition>, ... ] # @optional
jobOutputDefinitions: [ <JobOutputDefinition>, ... ] # @optional @extension CONTROL_FLOW
jobEnvironments: [ <Environment>, ... ] # @optional
steps: [ <StepTemplate> | <StepLoop>, ... ] # <StepLoop> @extension CONTROL_FLOW
```

Where:

1. *specificationVersion* — A literal that identifies the document as adhering to this schema.
2. *$schema* — Ignored. This property is allowed for compatibility with JSON-editing IDEs.
3. *extensions* — If provided, is a non-empty list of extensions to the schema. Introduced in [RFC 0002](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0002-model-extensions.md).
      * Extensions available for specification version 2023-09: [TASK_CHUNKING](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0001-task-chunking.md),
         [REDACTED_ENV_VARS](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0003-redacted-env-vars.md),
         [FEATURE_BUNDLE_1](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0004-enhanced-limits-and-capabilities.md),
         [EXPR](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0005-expression-language.md),
         [WRAP_ACTIONS](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0008-environment-wrap-actions.md),
         [CONTROL_FLOW](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0011-control-flow.md) (draft;
         specified in RFCs 0011, [0012](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0012-step-output-variables.md), and [0013](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0013-step-loops.md))
      * A template that lists `CONTROL_FLOW` must also list `EXPR`. Schedulers and validators must reject templates
        that list `CONTROL_FLOW` without `EXPR`.
4. *name* — The name to give to a Job that is created from the template. See: [&lt;JobName&gt;](#111-jobname).
5. *description* — A description to apply to all Jobs that are created from the template. It has no functional purpose,
   but may appear in UI elements. See: [&lt;Description&gt;](#72-description).
6. *parameterDefinitions* — A list of the Job Parameters that are available within the Job Template. Values
   for Job Parameters are defined when submitting the Job Template to create a Job to a render management system.
   The values of Job Parameters can be referenced in Format Strings.
   See: [&lt;JobParameterDefinition&gt;](#2-jobparameterdefinition).
      * Minimum number of elements: If provided, then this list must contain at least one element.
      * Maximum number of elements: The list must not contain more than 50
      elements or 200 elements with the `FEATURE_BUNDLE_1` extension.
7. *jobOutputDefinitions* — `@extension CONTROL_FLOW`. A list of the Job's outputs: typed values computed from the
   Job Parameters and the Steps' output variables, which the render management system reports to the Job's
   submitter and other callers. See: [&lt;JobOutputDefinition&gt;](#217-joboutputdefinition-extension-control_flow).
      * Minimum number of elements: If provided, then this list must contain at least one element.
      * Maximum number of elements: The list must not contain more than 50 elements.
      * No two elements in this list may have the same value for the `name` property.
8. *jobEnvironments* — An ordered list of the environments that are required to run Tasks in the Jobs created by this Job
   Template. These are entered in the order provided at the start of every Session for Tasks in the Job, and exited in the
   reverse order at the end of those Sessions. See: [&lt;Environment&gt;](#4-environment). Constraints:
      1. No two Environments in this list may have the same value for the `name` property.
      2. The Environments defined in this list must not have the same `name` as an Environment
         defined in any Step within the same Job Template.
9. *steps* — A list of the Steps in the Job. See: [&lt;StepTemplate&gt;](#3-steptemplate).
      * With the `CONTROL_FLOW` extension, an element that has a `loop` property is a loop: see
        [&lt;StepLoop&gt;](#38-steploop-extension-control_flow). The names of all Steps of the Job Template, the Steps
        in this list, the loops, and the Steps in every loop's body, must differ from each other, and the
        *dependencies* of an element of this list name only elements of this list.

#### 1.1.1. `<JobName>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Allowed characters: Any unicode character except those in the Cc unicode character category.
2. Minimum length: 1 character.
3. Maximum length: 128 characters, after the format string has been resolved.
512 characters, after the format string has been resolved if using the
`FEATURE_BUNDLE_1` extension.

#### 1.1.2. `<ExtensionName>`

A literal string matching the following character regex: `[A-Z_0-9]{3,128}`.

### 1.2. Environment Template

```yaml
specificationVersion: environment-2023-09
parameterDefinitions: [ <JobParameterDefinition>, ... ] # @optional
environment: <Environment>
```

1. *specificationVersion* — A literal that identifies the document as adhering to this schema.
2. *parameterDefinitions* — A list of the Job Parameters that are available within the Environment Template. Values
   for Job Parameters are defined when submitting a Job Template to create a Job to a render management system.
   Both the Job Template and Environment Template(s) for a submission are allowed to contain definitions for the same
   job parameter name, but the submission must be rejected if any of the definitions of the same job parameter name
   have unresolvable conflicts
   (See: [Merging Environment Template Parameter Definitions](#121-merging-environment-template-parameter-definitions)).
   The values of Job Parameters can be referenced in Format Strings.
   See: [&lt;JobParameterDefinition&gt;](#2-jobparameterdefinition).
      * Minimum number of elements: If provided, then this list must contain at least one element.
      * Maximum number of elements: The list must not contain more than 50 elements.
3. *environment* — The definition of the Environment that the Environment Template defines.
   See [&lt;Environment&gt;](#4-environment).

#### 1.2.1. Merging Environment Template Parameter Definitions

When submitting a job using a Job Template and one or more Environment Templates, then two or more of the given Templates
may contain job parameter definitions with the same job parameter name. This section describes how we determine whether
those definitions are in conflict, and how the constraints that they define are merged to derive the constraints on the
job parameters' values.

Job Parameter Definitions for the same parameter name are in conflict if:

1. They have differing values for the `type` property of the job parameter definition; or
2. The parameter type is "PATH" and either the `objectType` or `dataFlow` properties are defined with different values.

Properties in a Job Parameter Definition such as the `minLength` and `allowedValues` define constraints on
the value for the job parameter. These constraints do not need to be identical in every Job Parameter Definition for the same
parameter name, but each constraint must become more constrained as a subsequent definition is merged with the previous
ones in processing order -- the Environment Templates are processed in an order defined by the render management system
or submission, and the Job Template is always processed last. The exception to this rule is the `default` property of a
Job Parameter Definition; the default value for a job parameter is always the last `default` value defined in the template
processing order. The merged job parameter definition that results from this process must be internally consistent to be considered
valid -- for example, the value of the `default` property must adhere to the constraints in the definition.

## 2. `<JobParameterDefinition>`

```bnf
<JobParameterDefinition> ::= <JobStringParameterDefinition> | <JobPathParameterDefinition> |
                             <JobIntParameterDefinition> | <JobFloatParameterDefinition> |
                             <JobBoolParameterDefinition> |              # @extension EXPR
                             <JobRangeExprParameterDefinition> |         # @extension EXPR
                             <JobListStringParameterDefinition> |        # @extension EXPR
                             <JobListPathParameterDefinition> |          # @extension EXPR
                             <JobListIntParameterDefinition> |           # @extension EXPR
                             <JobListFloatParameterDefinition> |         # @extension EXPR
                             <JobListBoolParameterDefinition> |          # @extension EXPR
                             <JobListListIntParameterDefinition>         # @extension EXPR
```

When the `EXPR` extension is enabled, job parameter and task parameter type names become
case-insensitive. For example, `INT`, `Int`, and `int` are all equivalent, as are
`LIST[STRING]`, `List[String]`, and `list[string]`. See
[RFC 0007](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0007-extend-parameter-types.md)
for the design rationale.

The `userInterface` property on parameter definitions provides hints to user interface implementations. Its fields are suggestions for how to present the parameter, not validation constraints. If a field does not apply to the chosen UI element, it should be ignored. Properties outside of `userInterface`, such as `allowedValues`, `minValue`, and `maxValue`, are constraints that must be enforced during validation.

### 2.1. `<JobStringParameterDefinition>`

Defines a job parameter that allows input of a single string value to a Job Template.

The value of a Job Parameter of this type referenced in format strings as both:

1. `Param.<name>` and
2. `RawParam.<name>`

A `<JobStringParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "STRING"
description: <Description> # @optional
default: <JobParameterStringValue> # @optional
allowedValues: [ <JobParameterStringValue>, ... ] # @optional
minLength: <integer>,# @optional
maxLength: <integer> # @optional
userInterface: # @optional
   control: enum("LINE_EDIT", "MULTILINE_EDIT", "DROPDOWN_LIST", "CHECK_BOX", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
   See: [&lt;JobParameterStringValue&gt;](#25-jobparameterstringvalue).
4. *allowedValues* — An array of the values that the parameter is allowed to be. It is an error to provide a value that is
   not in this list, if the list is defined. See: [&lt;JobParameterStringValue&gt;](#25-jobparameterstringvalue).
5. *minLength* — The minimum allowable length of the parameter string value.
6. *maxLength* — The maximum allowable length of the parameter string value.
7. *userInterface* — User interface hints for this parameter. See [§2](#2-jobparameterdefinition) for how implementations should treat these fields.
    1. *control* — The user interface control to use when editing this parameter.
       The default, if not provided, is "LINE_EDIT" when *allowedValues* is not provided, "DROPDOWN_LIST" when it is.
        1. "LINE_EDIT" — This is a freeform string line edit control. Cannot be used when *allowedValues* is provided.
        2. "MULTILINE_EDIT" — This is a freeform string multi-line edit control. It uses a fixed width font, intended for
           editing script code. The vertical size of this control is set to grow to fit the available space. Cannot be
           used when *allowedValues* is provided.
        3. "DROPDOWN_LIST" — This is a dropdown list, for selecting from a fixed set of values. It requires that
           *allowedValues* is also provided.
        4. "CHECK_BOX" — This is a checkbox control. It requires that *allowedValues* is also provided, and contains two
           values, case-insensitive, one representing true and another representing false.
            * Valid pairs are ["true", "false"], ["yes", "no"], ["on", "off"], and ["1", "0"].
        5. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

With the EXPR extension, the value of a Job Parameter of this type is referenced in format strings as `Param.<name>`, returning a `string` type value.

### 2.2. `<JobPathParameterDefinition>`

Defines a job parameter that allows input to a Job Template of a single string value that represents a file or directory
path on a filesystem, or a URI. This parameter type differs from a
[`<JobStringParameterDefinition>`](#21-jobstringparameterdefinition) in that the value of the parameter automatically has
defined and matching path mapping rules applied to it.

The value of a Job Parameter of this type referenced in format strings as both:

1. `Param.<name>` — the value of the parameter with applicable path mapping applied to it; and
2. `RawParam.<name>` — the value of the parameter after the join described under *default* below, with no path
   mapping applied to it.

**`@extension EXPR` — URI values**: When the EXPR extension is enabled, PATH parameter values
may be URIs (values with a `scheme://` prefix, e.g. `s3://bucket/key`). URI values are not
subject to relative path resolution — they are not joined with the job template directory or
the current working directory. If no path mapping rule matches a URI value, it passes through
to `Param.<name>` unchanged. See the
[Expression Language path type description](2026-02-Expression-Language#121-types) for how
path operations handle URIs.

A `<JobPathParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "PATH"
description: <Description> # @optional
default: <JobParameterStringValue> # @optional
allowedValues: [ <JobParameterStringValue>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
objectType: enum("FILE", "DIRECTORY") # @optional
dataFlow: enum("NONE", "IN", "OUT", "INOUT") # @optional
userInterface: # @optional
   control: enum("CHOOSE_INPUT_FILE", "CHOOSE_OUTPUT_FILE", "CHOOSE_DIRECTORY", "DROPDOWN_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelString> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   fileFilters: [ <JobPathParameterFileFilter>, ... ] # @optional
   fileFilterDefault: <JobPathParameterFileFilter> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
   See: [&lt;JobParameterStringValue&gt;](#25-jobparameterstringvalue). When a Job is created from the template,
   a non-empty default that is not a URI is resolved as follows. Template validation does not apply these rules.
    1. The default must be a relative path. An absolute path is an error.
    2. The default is joined with the directory containing the Job Template, and the result is lexically
       normalized: `.` components are removed and `..` components are applied, without resolving symbolic links.
    3. The normalized result must be the Job Template's directory or a path within it. Otherwise it is an error;
       for example, `../x` and `a/../../x` are errors.

   A value supplied for the parameter when submitting a Job is not subject to these rules. It may be absolute or
   relative; a relative value is joined with the current working directory of the submission and lexically
   normalized in the same way. URI values are never joined; see *URI values* above.
4. *allowedValues* — An array of the values that the parameter is allowed to be. It is an error to provide a value that is
   not in this list, if the list is defined. See: [&lt;JobParameterStringValue&gt;](#25-jobparameterstringvalue).
   Template validation checks a *default* against this list as written. Job creation checks the value after the
   join described under *default* (with the Job Template's directory for a default, or with the current working
   directory for a submitted relative value), because the join is performed by the submitting client and the
   Job may be created elsewhere from the joined values. A relative *default* can therefore satisfy both checks
   only if the list contains both the default as written and its joined path. In practice, use *allowedValues*
   with absolute paths or URIs, not with a relative *default*.
5. *minLength* — The minimum allowable length of the parameter string value. Checked twice, as for *allowedValues*:
   at template validation against the *default* as written, and at job creation against the joined value.
6. *maxLength* — The maximum allowable length of the parameter string value. Checked twice, as for *allowedValues*:
   at template validation against the *default* as written, and at job creation against the joined value.
7. *objectType* — The type of object the path represents; either a FILE or a DIRECTORY. Default is DIRECTORY.
8. *dataFlow* — Whether the object the path represented serves as input, output or both for the Job. Default is NONE.
9. *userInterface — User interface properties for this parameter*
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, depends on
       *objectType*, *dataFlow*, and *allowedValues*. If *objectType* is FILE, then if *dataFlow* is "OUT", it is
       "CHOOSE_OUTPUT_FILE", otherwise it is "CHOOSE_INPUT_FILE". If *objectType* is not "FILE", then it is
       "CHOOSE_DIRECTORY". If *allowedValues* is provided, then the default is instead "DROPDOWN_LIST".
        1. "CHOOSE_INPUT_FILE" — This is a combination of a line edit and a button that uses the system's input file
           dialog. Cannot be used when *allowedValues* is provided.
        2. "CHOOSE_OUTPUT_FILE" — This is a combination of a line edit and a button that uses the system's output/save
           file dialog. Cannot be used when *allowedValues* is provided.
        3. "CHOOSE_DIRECTORY" — This is a combination of a line edit and a button that uses the system's directory dialog.
           Cannot be used when *allowedValues* is provided.
        4. "DROPDOWN_LIST" — This is a dropdown list, for selecting from a fixed set of values. It requires that
           *allowedValues* is also provided.
        5. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. fileFilters — Can be provided when the *uiControl* is "CHOOSE_INPUT_FILE" or "CHOOSE_OUTPUT_FILE". Defines the file
       filters that are shown in the file choice dialog. Maximum of 20 filters.
    5. fileFilterDefault — Can be provided when the *uiControl* is "CHOOSE_INPUT_FILE" or "CHOOSE_OUTPUT_FILE". The
       default file filter that's shown in the file choice dialog.

With the EXPR extension, the value of a Job Parameter of this type is referenced in format strings as `Param.<name>`, returning a `path` type value.

### 2.3 `<JobIntParameterDefinition>`

Defines a job parameter that allows input to a Job Template of a single integer value.

The value of a Job Parameter of this type referenced in format strings as both:

1. `Param.<name>` and
2. `RawParam.<name>`

A `<JobIntParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "INT"
description: <Description> # @optional
default: <integer> | <intstring> # @optional
allowedValues: [ <integer> | <intstring>,... ] # @optional
minValue: <integer> | <intstring> # @optional
maxValue: <integer> | <intstring> # @optional
userInterface:  # @optional
   control: enum("SPIN_BOX", "DROPDOWN_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   singleStepDelta: <positiveint> # @optional

```

Where `<intstring>` is a string whose value is the string representation of an integer value in base-10, with optional
white space around it (see [Numeric strings](#75-numeric-strings)), and:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *allowedValues* — An array of the values that the parameter is allowed to be. It is an error to provide a value that is
   not in this list, if the list is defined.
5. *minValue* — Minimum allowable value for the parameter. It is an error to provide a value for the parameter that is
   less than this.
6. *maxValue* — Maximum allowable value for the parameter. It is an error to provide a value for the parameter that is
   greater than this.
7. *userInterface — User interface properties for this parameter*
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is
      "SPIN_BOX" when *allowedValues* is not provided, "DROPDOWN_LIST" when it is.
        1. "SPIN_BOX" — This is an integer editing control. Cannot be used when *allowedValues* is provided.
        2. "DROPDOWN_LIST" — This is a dropdown list, for selecting from a fixed set of values. It requires that
           *allowedValues* is provided.
        3. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. *singleStepDelta* — How much the value changes for a single step modification, such as selecting an up or down
       arrow in the user interface control.

With the EXPR extension, the value of a Job Parameter of this type is referenced in format strings as `Param.<name>`, returning an `int` type value.

### 2.4. `<JobFloatParameterDefinition>`

Defines a job parameter that allows input to a Job Template of a single floating point or integer value.

The value of a Job Parameter of this type referenced in format strings as both:

1. `Param.<name>` and
2. `RawParam.<name>`

A `<JobFloatParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "FLOAT"
description: <Description> # @optional
default: <float> | <floatstring> # @optional
allowedValues: [ <float> | <floatstring>,... ] # @optional
minValue: <float> | <floatstring> # @optional
maxValue: <float> | <floatstring> # @optional
userInterface: # @optional
   control: enum(SPIN_BOX, DROPDOWN_LIST, HIDDEN) # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   decimals: <integer> # @optional
   singleStepDelta: <positivefloat> # @optional
```

Where `<floatstring>` is a string whose value is the string representation of a floating point or integer value in
base-10, with optional white space around it (see [Numeric strings](#75-numeric-strings)), and:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *allowedValues* — An array of the values that the parameter is allowed to be. It is an error to provide a value that is
   not in this list, if the list is defined.
5. *minValue* — Minimum allowable value for the parameter. It is an error to provide a value for the parameter that is
   less than this.
6. *maxValue* — Maximum allowable value for the parameter. It is an error to provide a value for the parameter that is
   greater than this.
7. *userInterface — User interface properties for this parameter*
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is
       "SPIN_BOX" when *allowedValues* is not provided, "DROPDOWN_LIST" when it is.
        1. "SPIN_BOX" — This is a floating point editing control. Cannot be used when *allowedValues* is provided.
        2. "DROPDOWN_LIST" — This is a dropdown list, for selecting from a fixed set of values. It requires that
           *allowedValues* is provided.
        3. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. *decimals* — This is the number of places editable after the decimal point. If *decimals* is not provided then an
       adaptive decimal mode will be used.
    5. *singleStepDelta* — How much the value changes for a single step modification, such as selecting an up or down
       arrow in the user interface control. If *decimals* is provided, this is an absolute value, otherwise it is the
       fraction of the current value to use as an adaptive step.

With the EXPR extension, the value of a Job Parameter of this type is referenced in format strings as `Param.<name>`, returning a `float` type value.

### 2.5. `<JobParameterStringValue>`

*This is a supplemental value type referenced by the parameter definitions in sections 2.1–2.4 and 2.9+.*

A string value subject to the following constraints:

1. Allowable characters: Any.
2. Minimum length: There is no minimum length.
3. Maximum length: 1024 characters.

### 2.6. `<UserInterfaceLabelStringValue>`

*This is a supplemental value type referenced by the parameter definitions in sections 2.1–2.4 and 2.9+.*

A string value subject to the following constraints:

1. Allowable characters: Any unicode character except those in the Cc unicode character category.
2. Minimum length: 1 character.
3. Maximum length: 64 characters.

### 2.7. `<JobPathParameterFileFilter>`

*This is a supplemental value type referenced by the parameter definitions in sections 2.1–2.4 and 2.9+.*

Represents one named file type for an input or output file choice dialog. For example:
`{"label": "Image Files", "patterns": ["*.png", "*.jpg", "*.exr"]}` or `{"label": "All Files", "patterns": ["*"]}`.

A `<JobPathParameterFileFilter>` is the object:

```yaml
label: <UserInterfaceLabelStringValue>
patterns: [ <FileDialogFilterPatternStringValue>, ... ]
```

### 2.8. `<FileDialogFilterPatternStringValue>`

*This is a supplemental value type referenced by the parameter definitions in sections 2.1–2.4 and 2.9+.*

A string value subject to the following constraints:

1. Allowable values: "*", "*.*", and "*.[:file-extension-chars:]+". The characters that :file-extension-chars: can take on
   are any unicode character except
    1. The Cc unicode character category.
    2. Path separators "\" and "/".
    3. Wildcard characters "*", "?", "[", "]".
    4. Characters commonly disallowed in paths: "#", "%", "&", "{", "}", "<", ">", "$", "!", "'", "\"", ":", "@", "`", "|",
       and "=".
2. Minimum length: 1 character.
3. Maximum length: 20 characters.

### 2.9. `<JobBoolParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a boolean value.

```yaml
name: <Identifier>
type: "BOOL"
description: <Description> # @optional
default: <bool> # @optional
userInterface: # @optional
   control: enum("CHECK_BOX", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Accepted values are:
- JSON/YAML boolean literals: `true`, `false`
- Integer or float `1` or `1.0` (true), `0` or `0.0` (false)
- Case-insensitive strings representing true: `"true"`, `"yes"`, `"on"`, `"1"`
- Case-insensitive strings representing false: `"false"`, `"no"`, `"off"`, `"0"`

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "CHECK_BOX".
        1. "CHECK_BOX" — A checkbox control for boolean input.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

The value of a Job Parameter of this type is referenced in format strings as `Param.<name>`, returning a `bool` type value.

Note: Unlike other parameter types, `BOOL` does not support `allowedValues` because restricting
to only `true` or only `false` does not provide meaningful value.

### 2.10. `<JobRangeExprParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a range expression string conforming to the
[`<IntRangeExpr>`](#34111-intrangeexpr) grammar.

```yaml
name: <Identifier>
type: "RANGE_EXPR"
description: <Description> # @optional
default: <string> # @optional, must be valid <IntRangeExpr>
minLength: <integer> # @optional
maxLength: <integer> # @optional
userInterface: # @optional
   control: enum("LINE_EDIT", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
   Must be a valid `<IntRangeExpr>`.
4. *minLength* — Minimum string length of the range expression. Must be >= 1 if provided.
5. *maxLength* — Maximum string length of the range expression. Default is 1024.
6. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "LINE_EDIT".
        1. "LINE_EDIT" — A single-line text input for entering range expressions.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — The user interface label to use when displaying the parameter's edit control. If not provided, the
       implementation should default to using the parameter *name*.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — Parameters with the same *groupLabel* value should be placed together in a grouping control with the
       value of *groupLabel* as its label.
       See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `range_expr` type value.
2. `RawParam.<name>` — Returns a `range_expr` type value (identical to `Param.<name>`).
3. `list(Param.<name>)` — Returns a `list[int]` with the expanded values.

### 2.11. `<JobListStringParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a list of string values.

```yaml
name: <Identifier>
type: "LIST[STRING]"
description: <Description> # @optional
default: [ <string>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
item: # @optional
  allowedValues: [ <string>, ... ] # @optional
  minLength: <integer> # @optional
  maxLength: <integer> # @optional
userInterface: # @optional
   control: enum("LINE_EDIT_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *minLength*/*maxLength* — Constrain the number of items in the list.
5. *item* — Constraints for each item in the list.
    1. *allowedValues* — An array of the values that each item is allowed to be.
    2. *minLength*/*maxLength* — Constrain the string length of each item.
6. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "LINE_EDIT_LIST".
        1. "LINE_EDIT_LIST" — A list of line edit controls.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[string]` type value.
2. `Param.<name>[i]` — Returns the i-th element as `string`.
3. `len(Param.<name>)` — Returns the count of elements.

### 2.12. `<JobListPathParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a list of path values.

```yaml
name: <Identifier>
type: "LIST[PATH]"
description: <Description> # @optional
objectType: enum("FILE", "DIRECTORY") # @optional
dataFlow: enum("IN", "OUT", "INOUT", "NONE") # @optional
default: [ <string>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
item: # @optional
  allowedValues: [ <string>, ... ] # @optional
  minLength: <integer> # @optional
  maxLength: <integer> # @optional
userInterface: # @optional
   control: enum("CHOOSE_INPUT_FILE_LIST", "CHOOSE_OUTPUT_FILE_LIST",
                 "CHOOSE_DIRECTORY_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   fileFilters: [ <JobPathParameterFileFilter>, ... ] # @optional
   fileFilterDefault: <JobPathParameterFileFilter> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *objectType* — The type of object the paths represent; either FILE or DIRECTORY. Default is DIRECTORY.
4. *dataFlow* — Whether the objects the paths represent serve as input, output or both for the Job. Default is NONE.
5. *default* — Default value to use for the parameter if the submission does not include a value for it.
   Each element is resolved by the rules for the *default* of a
   [`<JobPathParameterDefinition>`](#22-jobpathparameterdefinition), applied to that element. It is an error
   if any element violates them. An empty list is a valid default.
6. *minLength*/*maxLength* — Constrain the number of paths in the list.
7. *item* — Constraints for each item in the list. As for a `<JobPathParameterDefinition>`, template validation
   checks each element of *default* as written, and job creation checks each element after it has been joined
   with the Job Template's directory (for a default) or the current working directory (for a submitted relative
   value).
    1. *allowedValues* — An array of the values that each item is allowed to be.
    2. *minLength*/*maxLength* — Constrain the string length of each path.
8. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use. The default depends on *objectType* and *dataFlow*:
       If *objectType* is FILE and *dataFlow* is "OUT", default is "CHOOSE_OUTPUT_FILE_LIST".
       If *objectType* is FILE otherwise, default is "CHOOSE_INPUT_FILE_LIST".
       If *objectType* is DIRECTORY, default is "CHOOSE_DIRECTORY_LIST".
        1. "CHOOSE_INPUT_FILE_LIST" — A list of input file choice controls.
        2. "CHOOSE_OUTPUT_FILE_LIST" — A list of output file choice controls.
        3. "CHOOSE_DIRECTORY_LIST" — A list of directory choice controls.
        4. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. *fileFilters* — File filters for the file choice dialog (only for CHOOSE_INPUT_FILE_LIST/CHOOSE_OUTPUT_FILE_LIST).
    5. *fileFilterDefault* — Default file filter for the file choice dialog.

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[path]` type value with path mapping applied.
2. `RawParam.<name>` — Returns a `list[string]` type value holding each element after it has been joined
   as described under *default*, without path mapping.
3. `Param.<name>[i]` — Returns the i-th element as `path`.
4. `len(Param.<name>)` — Returns the count of elements.

Each element of a `LIST[PATH]` value is processed as the value of a `PATH` parameter is: a relative element of a
submitted value is joined with the current working directory of the submission, a relative element of the
*default* is joined with the Job Template's directory, and path mapping rules are applied to each element
independently.

### 2.13. `<JobListIntParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a list of integer values.

```yaml
name: <Identifier>
type: "LIST[INT]"
description: <Description> # @optional
default: [ <integer> | <intstring>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
item: # @optional
  allowedValues: [ <integer> | <intstring>, ... ] # @optional
  minValue: <integer> | <intstring> # @optional
  maxValue: <integer> | <intstring> # @optional
userInterface: # @optional
   control: enum("SPIN_BOX_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   singleStepDelta: <positiveint> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *minLength*/*maxLength* — Constrain the number of items in the list.
5. *item* — Constraints for each item in the list.
    1. *allowedValues* — An array of the values that each item is allowed to be.
    2. *minValue*/*maxValue* — Constrain the value of each integer item.
6. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "SPIN_BOX_LIST".
        1. "SPIN_BOX_LIST" — A list of integer editing controls.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. *singleStepDelta* — How much the value changes for a single step modification.

Each item of a value given for this parameter when a Job is created, each item of *default* and
*item.allowedValues*, and *item.minValue* and *item.maxValue* may be an integer or an `<intstring>` (see
[Numeric strings](#75-numeric-strings)). For example, `[1, " 2 "]` is the list `[1, 2]`.

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[int]` type value.
2. `Param.<name>[i]` — Returns the i-th element as `int`.
3. `len(Param.<name>)` — Returns the count of elements.

### 2.14. `<JobListFloatParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a list of floating-point values.

```yaml
name: <Identifier>
type: "LIST[FLOAT]"
description: <Description> # @optional
default: [ <float> | <floatstring>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
item: # @optional
  allowedValues: [ <float> | <floatstring>, ... ] # @optional
  minValue: <float> | <floatstring> # @optional
  maxValue: <float> | <floatstring> # @optional
userInterface: # @optional
   control: enum("SPIN_BOX_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
   decimals: <integer> # @optional
   singleStepDelta: <positivefloat> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *minLength*/*maxLength* — Constrain the number of items in the list.
5. *item* — Constraints for each item in the list.
    1. *allowedValues* — An array of the values that each item is allowed to be.
    2. *minValue*/*maxValue* — Constrain the value of each float item.
6. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "SPIN_BOX_LIST".
        1. "SPIN_BOX_LIST" — A list of floating point editing controls.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    4. *decimals* — The number of places editable after the decimal point.
    5. *singleStepDelta* — How much the value changes for a single step modification.

Each item of a value given for this parameter when a Job is created, each item of *default* and
*item.allowedValues*, and *item.minValue* and *item.maxValue* may be a number or a `<floatstring>` (see
[Numeric strings](#75-numeric-strings)). An item of a value or of *default* given as a `<floatstring>` keeps the text
it was written with, without the white space around it, as in
[Float Value Pass-Through](2026-02-Expression-Language#134-float-value-pass-through). For example, given
`[" 1.00 ", "2.50"]`, `{{Param.<name>[0]}}` resolves to `1.00`, and with `default: [1.5, "2.50"]`,
`{{Param.<name>[1]}}` resolves to `2.50`.

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[float]` type value.
2. `Param.<name>[i]` — Returns the i-th element as `float`.
3. `len(Param.<name>)` — Returns the count of elements.

### 2.15. `<JobListBoolParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a list of boolean values.

```yaml
name: <Identifier>
type: "LIST[BOOL]"
description: <Description> # @optional
default: [ <bool>, ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
userInterface: # @optional
   control: enum("CHECK_BOX_LIST", "HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Each list item accepts the same values as [`<JobBoolParameterDefinition>`](#29-jobboolparameterdefinition):
- JSON/YAML boolean literals: `true`, `false`
- Integer or float `1` or `1.0` (true), `0` or `0.0` (false)
- Case-insensitive strings representing true: `"true"`, `"yes"`, `"on"`, `"1"`
- Case-insensitive strings representing false: `"false"`, `"no"`, `"off"`, `"0"`

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *minLength*/*maxLength* — Constrain the number of items in the list.
5. *userInterface* — User interface properties for this parameter.
    1. *control* — The user interface control to use when editing this parameter. The default, if not provided, is "CHECK_BOX_LIST".
        1. "CHECK_BOX_LIST" — A list of checkbox controls.
        2. "HIDDEN" — This hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[bool]` type value.
2. `Param.<name>[i]` — Returns the i-th element as `bool`.
3. `len(Param.<name>)` — Returns the count of elements.

### 2.16. `<JobListListIntParameterDefinition>` `@extension EXPR`

Defines a job parameter that accepts a nested list of integer values. This enables use cases
like representing graph adjacency lists for task-task dependencies.

```yaml
name: <Identifier>
type: "LIST[LIST[INT]]"
description: <Description> # @optional
default: [ [ <integer> | <intstring>, ... ], ... ] # @optional
minLength: <integer> # @optional
maxLength: <integer> # @optional
item: # @optional
  minLength: <integer> # @optional
  maxLength: <integer> # @optional
  item: # @optional
    allowedValues: [ <integer> | <intstring>, ... ] # @optional
    minValue: <integer> | <intstring> # @optional
    maxValue: <integer> | <intstring> # @optional
userInterface: # @optional
   control: enum("HIDDEN") # @optional
   label: <UserInterfaceLabelStringValue> # @optional
   groupLabel: <UserInterfaceLabelStringValue> # @optional
```

Where:

1. *name* — The name by which the parameter is referenced. See: [&lt;Identifier&gt;](#71-identifier).
2. *description* — A description to apply to the parameter. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *default* — Default value to use for the parameter if the submission does not include a value for it.
4. *minLength*/*maxLength* — Constrain the number of inner lists.
5. *item* — Constraints for each inner list.
    1. *minLength*/*maxLength* — Constrain the size of each inner list.
    2. *item* — Constraints for each integer in the inner list.
        1. *allowedValues* — An array of the values that each integer is allowed to be.
        2. *minValue*/*maxValue* — Constrain the value of each integer.
6. *userInterface* — User interface properties for this parameter.
    1. *control* — The only supported control is "HIDDEN", which hides the parameter from the user interface.
    2. *label* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).
    3. *groupLabel* — See: [&lt;UserInterfaceLabelStringValue&gt;](#26-userinterfacelabelstringvalue).

Each integer in an inner list of a value given for this parameter when a Job is created or of *default*, each item of
*item.item.allowedValues*, and *item.item.minValue* and *item.item.maxValue* may be an integer or an `<intstring>` (see
[Numeric strings](#75-numeric-strings)). For example, `[[1, " 2 "], ["3"]]` is `[[1, 2], [3]]`.

The value of a Job Parameter of this type is referenced in format strings as:

1. `Param.<name>` — Returns a `list[list[int]]` type value.
2. `Param.<name>[i]` — Returns the i-th element as `list[int]`.
3. `Param.<name>[i][j]` — Returns the j-th element of the i-th list as `int`.
4. `len(Param.<name>)` — Returns the count of outer list elements.
5. `len(Param.<name>[i])` — Returns the count of inner list elements.

### 2.17. `<JobOutputDefinition>` `@extension CONTROL_FLOW`

Defines a Job output, the counterpart of a Job Parameter: a value the Job reports back to its submitter and other
callers. Introduced in [RFC 0011](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0011-control-flow.md) (draft).

```yaml
name: <Identifier>
type: <OutputVariableType>
value: <FormatString> # @fmtstring
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# @optional constraint properties, as on <OutputVariableDefinition>.
```

Where:

1. *name* — The name by which callers read the output. Job outputs are not available in any Format String. Job
   output names and Job Parameter names are separate namespaces.
2. *type* — See [&lt;OutputVariableDefinition&gt;](#37-outputvariabledefinition-extension-control_flow): the same types,
   expression types, and constraint properties. `LIST[LIST[INT]]` is allowed.
3. *value* — The output's value, whose target type is the expression type of *type*, made optional: a `null` result
   means the output has no value, which is not an error. It may reference `Param.*` (except `PATH` and `LIST[PATH]`
   parameters), `RawParam.*`, `Job.Name`, and the `Steps.*` values of any Step of the Job except a Step of a loop's body (see
   [3.8.4](#384-scopes-in-and-around-a-loop)), with values whose type is or
   contains `path` read through `Steps.<StepName>.RawOutput.*` and `Steps.<StepName>.TaskRawParam.*`. It may not reference `Step.Name`, `Task.*`,
   `Session.*`, or names bound by `let`.
4. *description*, *unit* — For display only, as on `<OutputVariableDefinition>`.
5. Constraint properties — Checked against the value when it is evaluated. A `null` value is not checked.

The render management system handles Job outputs as follows:

1. **When an output is evaluated.** Each Job output is evaluated on its own, as soon as every `Steps.*` value its
   *value* references is concrete (see [Value References](#731-value-references)). An output that references no
   `Steps.*` value, or only Task parameters of Steps whose Tasks are known at Job creation, is evaluated at Job
   creation. An output that reads `Output` or `RawOutput` of a Step is evaluated when that Step succeeds or is
   skipped. It does not need the referenced Steps to be dependencies of anything. A Job does not succeed until every
   Job output whose inputs are concrete has been evaluated.
2. **Commit and display.** An evaluated output is committed at once and kept for the life of the Job. The render
   management system should make Job outputs visible to users and return them to callers, using *description* and
   *unit*, and should show an output that is not yet evaluated as pending and name what it waits for.
3. **Failure.** If evaluating an output fails, including a `fail(...)` call, a value that doesn't convert to *type*, a
   violated constraint property, or a size over the limit of item 7, the Job fails: it is marked failed with a message
   that names the output, the expression, and the problem, no further Tasks are dispatched, and Tasks that are running
   should be canceled. An output evaluated at Job creation that fails makes Job creation fail.
4. **Missing producers.** A Step's Task parameters are concrete once its Tasks are known, which can be before the Step
   can run, and an output evaluated then stays committed even if that Step later fails; the Job then fails as it
   otherwise would. If a Step that an output not yet evaluated references fails, or can never run because one of its
   dependencies failed, the output is not evaluated while that remains so (if a requeue later makes the Step succeed,
   it is evaluated then), and it should be shown as unavailable, naming the Step, rather than as pending. If the Step
   is skipped, its values are `null` and the output is evaluated with them. A Step with zero Tasks gives empty lists, so
   `min(Steps.P.Output.X)` fails the Job when the parameter space is empty; write
   `min(Steps.P.Output.X) if len(Steps.P.Output.X) > 0 else null` where emptiness is expected.
5. **Re-runs.** Job outputs do not count as consumers for the re-run rules of
   [How Jobs Are Run](How-Jobs-Are-Run#step-states-and-step-input-binding). When a re-run of a Task commits new values,
   every Job output that references its Step is evaluated again and its value replaced. If that fails, the re-run's
   values stay committed, the output keeps its earlier value, marked stale, and the Job fails as in item 3, even if it
   had already succeeded.
6. **Paths.** A `PATH` or `LIST[PATH]` Job output built from `PATH` values is in the submission path context. A value
   built from a `STRING` is whatever that string holds.
7. **Limits.** An implementation must accept Job output values totaling at least 1 MiB (1,048,576 bytes) per Job,
   measured as their JSON serialization (see below), and may accept more. The total counts each Job output's current
   value, and an output with no value counts nothing.
8. **Secrets.** Job outputs are stored and displayed, so they are not redacted and must not be used to pass secrets.

**Measuring a value's size.** Where this specification measures a value by its *JSON serialization* (Job outputs in
item 7, the stored form of a `PATH` output value in [3.7.1](#371-reporting-output-values), and loop values in
[3.8.5](#385-running-a-loop)), its size is the number of bytes of its serialization under the JSON Canonicalization
Scheme ([RFC 8785](https://www.rfc-editor.org/rfc/rfc8785)): no whitespace, a `string` or `path` as a UTF-8 JSON string
with that scheme's escapes, and a `float` in its ECMAScript number form (`2`, `1.5e-7`, `1e+300`). An `int` is written
in plain decimal instead, and a `range_expr` as a JSON string of its canonical form, the text the Expression Language's
`string()` gives it (`1-5`). An implementation may store values in any form, but measures them this way. A Task's own
output values are otherwise measured by the bytes that reported them (see [3.7.1](#371-reporting-output-values)), so a
value built from a Task output can measure differently here: a `FLOAT` reported as `1.50` counts as `1.5`.

## 3. `<StepTemplate>`

A `<StepTemplate>` defines a single Step in the Job; the action(s) that it takes, its dependencies, the parameter space
that maps to the Step's Tasks, and its runtime environment and requirements.

A `<StepTemplate>` is the object:

```yaml
name: <StepName>
description: <Description> # @optional
enableIf: <bool> | <FormatString> # @optional @fmtstring @extension CONTROL_FLOW
let: <LetBindings> # @optional @extension EXPR
dependencies: [ <StepDependency>, ... ] # @optional
stepEnvironments: [ <Environment>, ... ] # @optional
hostRequirements: <HostRequirements> # @optional
parameterSpace: <StepParameterSpaceDefinition> # @optional
outputVariables: [ <OutputVariableDefinition>, ... ] # @optional @extension CONTROL_FLOW
script: <StepScript>
```

Where:

1. *name* — The name to identify the Step within the Job. Each Step within a Job Template must have name that differs from
   the names of all other Steps in the same Job Template. See: [&lt;StepName&gt;](#31-stepname). With the `CONTROL_FLOW`
   extension this includes loops and the Steps in their bodies (see [&lt;StepLoop&gt;](#38-steploop-extension-control_flow)).
2. *description* — A description to apply to the step. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *enableIf* — `@extension CONTROL_FLOW`. Whether the Step runs; the expression's target type is `bool`, and a `null`
   result is an error rather than an omitted field. If it evaluates to `false`, the Step is *skipped*: none of its Tasks runs, its Step Environments are not entered, and every
   `Steps.<name>.*` value of the Step (see [Value References](#731-value-references)) is `null`. A skipped Step
   satisfies the dependencies of the Steps that depend on it once its own dependencies are satisfied. If omitted, the
   Step is always enabled. Constraints:
    1. It may reference `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`, `Job.Name`, `Step.Name`, and the
       `Steps.*` values of the Step's direct dependencies. In a Step of a loop's body, it may also reference `Loop.*`,
       `RawLoop.*`, and the `Steps.*` values that [3.8.4](#384-scopes-in-and-around-a-loop) allows. It may not
       reference names bound by the Step's own *let*, `Task.*`, or `Session.*`.
    2. If every value it references is concrete at Job creation (it references no `Steps.*` value, or only values
       concrete then, such as those of a Step skipped at Job creation), it is evaluated at Job creation. A Step
       disabled at Job creation is still part of the Job and is shown as skipped, so every Job created from a template
       has the same Steps, whatever its parameter values. Otherwise it
       is evaluated at step input binding (see [Template Processing Stages](#74-template-processing-stages)) once
       those values are concrete. Either way it is decided before the Step's *let* and every other materialization expression of the
       Step is evaluated, and if it is `false` none of them is: the Step's parameter space is the empty set (its `Steps.*` values are
       `null`, not `[]`).
4. *let* — An ordered list of expression bindings evaluated once per step. Bound names are available in
   *stepEnvironments*, *hostRequirements*, *parameterSpace*, and *script* fields. See: [&lt;LetBindings&gt;](#36-letbindings).
5. *dependencies* — A list of the dependencies of this Step. These dependencies must be resolved before the Tasks of the
   Step may be scheduled. See: [&lt;StepDependency&gt;](#32-stepdependency)
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: There is no maximum defined, though implementations may choose to constrain the number of
    dependencies.
    * With the `CONTROL_FLOW` extension, listing a Step here is also what makes its `Steps.*` values available to this
      Step. In a Step of a loop's body, this list names only Steps of the same body (see
      [&lt;StepLoop&gt;](#38-steploop-extension-control_flow)).
6. *stepEnvironments* — An ordered list of the environments that are required to run Tasks in this Step.
  These are entered in the order provided at the start of every Session for Tasks in the Step, and exited in the reverse
  order at the end of those Sessions.
  See: [&lt;Environment&gt;](#4-environment).
    * Constraints:
        1. No two Environments in this list may have the same value for the `name` property.
        2. The Environments defined in this list must not have the same `name` as a Job Environment defined in the same
           Job Template.
    * Note: The scope of a Step Environment's `name` is the Step that defines it. Different Steps may each define a
      Step Environment with the same `name`; a Session only ever contains the Step Environments of a single Step, so
      these names never collide.
7. *hostRequirements* — Describes the requirements on Worker host's capabilities that must be satisfied for the Task(s) of
   the Step to be scheduled to the host. See: [&lt;HostRequirements&gt;](#33-hostrequirements).
8. *parameterSpace* — Defines the parameterization of the Step's action; the available parameters, the values that they
   take on, and how those parameters' values are combined to produce the Tasks of the Step. Absent this property the Step
   is run a single time.
9. *outputVariables* — `@extension CONTROL_FLOW`. The output variables that every Task of the Step reports with
   `openjd_output_var:` directives. See: [&lt;OutputVariableDefinition&gt;](#37-outputvariabledefinition-extension-control_flow).
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: 32.
    * No two elements in this list may have the same value for the `name` property.
10. *script* — The action that is taken by this Step's Tasks when they are run on a Worker host.

With the `CONTROL_FLOW` extension, the Step's format strings that are resolved at Job creation (those not annotated
`@fmtstring[host]`, in *enableIf*, *let*, *hostRequirements*, *parameterSpace*, *stepEnvironments*, and *script*), and
with the `ENV_PARAMS` extension the *enableIf* of its *stepEnvironments* and the fields of its Environment selection
(see [&lt;Environment&gt;](#4-environment)), are the Step's *materialization expressions*. They may reference the
`Steps.*` values of the Step's direct dependencies. They are divided into *materialization groups*. *enableIf* is
decided first; after it is `true` (or if it is absent), each other group can be evaluated as soon as the `Steps.*`
values it reads (directly or through the *let* bindings it uses) are concrete, and should be evaluated then. Every group
whose values are concrete at Job creation is evaluated then, only in a Step whose *enableIf* is `true` or absent at Job
creation. The groups are listed below. When more than one group can be evaluated at the same moment, they are evaluated
in this order:

1. *enableIf*, which is decided before any other group is evaluated, followed, once it is `true` and with the
   `ENV_PARAMS` extension, by the Step's Environment selection (see [&lt;Environment&gt;](#4-environment));
2. *parameterSpace* (the `range` of each Task parameter in declaration order, then its other format strings, such as
   the `chunks` fields of `TASK_CHUNKING`); once it is evaluated, the Step's Tasks are known;
3. *hostRequirements*, and with `ENV_PARAMS` the forming and checking of the Step's combined host requirements; once
   it is evaluated, the Step's host requirements are known;
4. with a future task-task dependencies extension (`TASK_DEPENDENCIES`) only, the subspace ranges of its
   *dependencies* and the checks of the Task-level dependency graph built from them, evaluated once the Step's Tasks,
   and the Tasks of every Step those dependencies name, are known;
5. every other materialization expression: for each of its *stepEnvironments*, in order, its *enableIf*
   (`ENV_PARAMS`) and then, unless that is `false`, its format strings; and the format strings of its *script*, such as
   an action's `timeout` and `cancelation` fields.

A *let* binding is evaluated with the first group that uses it, and the last group uses every *let* binding. An
evaluation error in any group fails the Step as soon as that group is evaluated (or Job creation, for a group
evaluated then), with a message that names the field and the expression, and no Task of the Step is dispatched until every group has been evaluated. See
[Template Processing Stages](#74-template-processing-stages) and
[How Jobs Are Run](How-Jobs-Are-Run#step-states-and-step-input-binding).

### 3.1. `<StepName>`

A string subject to the following constraints:

1. Allowed characters: Any unicode character except those in the Cc unicode character category.
2. Minimum length: 1 character.
3. Maximum length: 64 characters. 512 characters if using the
`FEATURE_BUNDLE_1` extension.

### 3.2. `<StepDependency>`

This entity is used by the template author to specify a Step in the same Job that a Step depends upon.

A `<StepDependency>` is the object:

```yaml
dependsOn: "<StepName>"
```

Where:

1. *dependsOn* — Provides the name of another step in the same job upon which this step depends. The Task(s) of this Step
   may be scheduled only when the depended-upon Step has fully completed successfully.

With the `CONTROL_FLOW` extension, a dependency is also satisfied by a depended-upon Step that completed with zero
Tasks, or that was skipped by its *enableIf*, once that Step's own dependencies are satisfied. A Step that failed, or
that can never run because one of its own dependencies failed, never satisfies a dependency.

### 3.3. `<HostRequirements>`

This entity is used by the template author to describe the requirements on Worker host and/or render manager's available
capabilities that must be satisfied for the Task(s) of the Step to be scheduled to the host.

Each requirement corresponds to an attribute of a host or render manager that must be satisfied to allow the Step to
be scheduled to the host. Some examples of concrete attributes include processor architecture (x86_64, arm64, etc), the
number of CPU cores, the amount of system memory, or available floating licenses for an application. We also allow for
user-defined whose meaning is defined by the customer; a "SoftwareConfig" requirement whose values could be "Option1" or "Option2",
for example.

There are two types of requirements: attribute and amount.

A `<HostRequirements>` is the object:

```yaml
amounts: [ <AmountRequirement>, ... ] # @optional
attributes: [ <AttributeRequirement>, ... ] # @optional
```

Where:

1. *amounts* — Defines a set of quantifiable requirements on the host that must be met and reserved for a Session running
   Tasks from this Step to be scheduled to the host. See: [&lt;AmountRequirement&gt;](#331-amountrequirement).
2. *attributes* — Defines a set of attributes that a host must have for a Session running Tasks from this Step to be
   scheduled to the host. See: [&lt;AttributeRequirement&gt;](#332-attributerequirement).

With the constraints:

1. If this object is provided in a Job Template, then at least one of the *amounts* or *attributes* properties must be
   defined.
2. The sum of the lengths of the *amounts* and *attributes* arrays must not exceed 50 elements.
3. No two elements in the *amounts* array may have the same value for the *name* property, after the *name* format
   strings have been resolved.
4. No two elements in the *attributes* array may have the same value for the *name* property, after the *name* format
   strings have been resolved.

#### 3.3.1. `<AmountRequirement>`

Amount requirements are the mechanism for defining a quantity of something that the Worker host or render manager needs to have
for a Step to run. They represent quantifiable things that need to be reserved to do the work — vCPUs, memory, licenses, etc. They
are always non-negative floating point valued, and a Step can require a certain amount of that capability to be able to run —
"at least 4 CPU cores" for example. Further, a quantity of each amount required are logically allocated to a Session while that
session is running on a host. A Step requiring, say, "at least 4 CPU cores", might result in a Session with 4 CPU cores allocated
to it being created on a host. Those cores are reserved for that Session while that Session is running on the host; effectively
making the number of available cores on the host 4 less for scheduling purposes during the duration of the Session. Logically
allocating amounts to Sessions is the key mechanism by which system resources can be optimally utilized through bin packing
multiple running Sessions onto hosts at the same time.

An `<AmountRequirement>` is the object:

```yaml
name: <AmountCapabilityName> # @fmtstring
min: <nonnegativefloat> # @optional
min: <nonnegativefloat> | <nonnegativefloatstring>  # @optional @fmtstring @extension FEATURE_BUNDLE_1
max: <positivefloat> # @optional
max: <positivefloat> | <positivefloatstring> # @optional @fmtstring @extension FEATURE_BUNDLE_1
```

Where:`<nonnegativefloatstring>` is a string whose value is the string
representation of a non-zero and positive floating point or integer value in
base-10, `<positivefloatstring>` is a string whose value is the string
representation of a positive floating point or integer value in base-10, each with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *name* — Is a dotted name that identifies the specific host capability that is being required.
2. *min* — If provided, then the host must have at least the given amount of the named capability available and reserved
   for a Session running Tasks from this Step to be scheduled to the host. If not provided, then the default is 0 unless
   the specific host capability defines a minimum. Can only be a
   `<nonnegativefloatstring>` when using the extension `FEATURE_BUNDLE_1`.
3. *max* — If provided, then the host must have at most the given amount of the named capability available and reserved
   for a Session running Tasks from this Step to be scheduled to the host. Can
   only be a `<positivefloatstring>` when using the extension `FEATURE_BUNDLE_1`.

Subject to the constraint that at least one of *min* or *max* must be provided.

##### 3.3.1.1. `<AmountCapabilityName>`

A [Format String](#73-format-strings) subject to the following constraints, after the format string has been resolved.

1. Minimum length: 1 character.
2. Maximum length: 100 characters.
3. Its format matches the pattern `[<Identifier>:]amount.<Identifier>[.<Identifier>]*` where:
   1. `[<Identifier>:]` denotes an optional vendor-specific prefix.
   2. This specification has reserved specific values of the first `<Identifier>` after "amount." for use in this and
      future revisions. The reserved values are: "worker", "job", "step", and "task"
4. The value is not case-sensitive - All comparisons between strings of this type must be case-insensitive comparisons.

See [&lt;Identifier&gt;](#71-identifier) for information on the `<Identifier>` substring.

This specification defines the following amount capabilities, but an implementation is allowed, and encouraged, to allow
users to define their own custom capabilities.

|**Capability Name**|**Minimum Value**|**Description**|
|---|---|---|
|amount.worker.vcpu        |1|A number of vCPUs/CPU-cores available on the host.|
|amount.worker.memory      |0|An amount of system memory available on the host. Units: MiB. |
|amount.worker.gpu         |0|A number of GPUs available on the host.|
|amount.worker.gpu.memory  |0|The lower bound of total memory provided by each GPU on the host. For example, if a host has one GPU with 4096 and one GPU with 2048, this value would be 2048. Units: MiB.|
|amount.worker.disk.scratch|0|A static amount of disk storage installed on the host for use as scratch space. Units: GiB.|

#### 3.3.2. `<AttributeRequirement>`

Attribute requirements are the mechanism for defining an abstract or concrete attribute/property the host must have for
a Step to run. They are always defined on a host as a set of strings. A Step can assert that it requires that a host have
specific value(s) of the attribute for it to be scheduled to the host.

An `<AttributeRequirement>` is the object:

```yaml
name: <AttributeCapabilityName> # @fmtstring
anyOf: [ <AttributeCapabilityValue>, ... ] # @optional @fmtstring
allOf: [ <AttributeCapabilityValue>, ... ] # @optional @fmtstring
```

Where:

1. *name* — Is a dotted name that identifies the specific host capability that is being required.
2. *anyOf* — For the Step to be scheduled to it, the host's value for the named capability must have **at least one** of
   the given values. This comparison is case-insensitive.
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: The list must not contain more than 50 elements.
3. *allOf* — For the Step to be scheduled to it, the host's value for the named capability must have **all** the given
   values. This comparison is case-insensitive.
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: The list must not contain more than 50 elements.

Subject to the constraint that at least one of *anyOf* or *allOf* must be provided.

##### 3.3.2.1. `<AttributeCapabilityName>`

A [Format String](#73-format-strings) subject to the following constraints, after the format string has been resolved.

1. Minimum length: 1 character.
2. Maximum length: 100 characters.
3. Its format matches the pattern `[<Identifier>:]attr.<Identifier>[.<Identifier>]*` where:
   1. `[<Identifier>:]` denotes an optional vendor-specific prefix.
   2. This specification has reserved specific values of the first `<Identifier>` after "amount." for use in this and
      future revisions. The reserved values are: "worker", "job", "step", and "task"
4. The value is not case-sensitive - All comparisons between strings of this type must be case-insensitive comparisons.

See [&lt;Identifier&gt;](#71-identifier) for information on the `<Identifier>` substring.

This specification defines the following attribute capabilities, but an implementation is allowed, and encouraged, to
allow users to define their own custom capabilities.

|**Capability Name**|**Values**|**Description**|
|---|---|---|
|attr.worker.os.family|linux, windows, macos|The family of operating system on the host.|
|attr.worker.cpu.arch |x86_64, arm64|The architecture of the CPUs on the host.|

##### 3.3.2.2. `<AttributeCapabilityValue>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Minimum length: 1 character.
2. After the format string has been resolved:
    1. Max length: 100 characters
    2. Unicode alphanumeric characters in the latin character set, plus the underscore and hyphen characters.
    3. Must start with either a letter character or the underscore character.

### 3.4. `<StepParameterSpaceDefinition>`

The step parameter space defines the task parameters that are substituted into the `<StepScript>`
to produce all the tasks to run. A `<StepParameterSpaceDefinition>` is the object:

```yaml
taskParameterDefinitions: [ <TaskParameterDefinition>, ... ]
combination: <CombinationExpr> # @optional
```

Where:

1. *taskParameterDefinitions* — This is a list that defines the task parameters over which the Step's script is
   parameterized. Each task parameter defines its name, type, and the range of values that it takes.
   When the `TASK_CHUNKING` is used, only one `CHUNK[INT]` parameter may be included.
    1. Minimum number of elements: If provided, then this array must have at least one element in the list.
    2. Maximum number of elements: The list must contain no more than 16 elements in the list.
2. *combination* — An expression that defines how the task parameters are combined to produce the parameter space that
   maps a Step to a set of Tasks. Each Task has a single value for each task parameter that specifies its coordinates in
   the defined space. If not provided, then the default is all possible combinations of task parameter values.

With the `CONTROL_FLOW` extension:

1. A task parameter `range` given as a Format String, and the other format strings of the *parameterSpace* (such as
   the `chunks` fields of `TASK_CHUNKING`), may reference the `Steps.*` values of the Step's direct dependencies. The
   *combination* may not. A `PATH` Task parameter whose `range` comes from `Steps.*.RawOutput.*` or
   `Steps.*.TaskRawParam.*` holds values in the submission path context, and its `Task.Param.*` is mapped forward on
   the worker host, as a range from `RawParam.*` is. Expressions over output values are evaluated under the Expression
   Language's existing memory and operation limits.
2. A task parameter `range` given as a Format String may evaluate to an empty list, for any Task parameter type.
   Combined by product, it makes the parameter space empty; associated with a non-empty range, it is an error when
   that range is evaluated, and associated empty ranges give an empty space. A Step with an empty parameter space has
   zero Tasks and completes successfully once its dependencies are satisfied, and its `Steps.*.Output` and
   `Steps.*.TaskParam` lists are `[]`. A range expression string such as `"1-{{N}}"` cannot be empty (with `N = 0` it
   is `1-0`); write `"{{range(1, N + 1)}}"` for zero Tasks when `N = 0`. A `range` written as a list literal must have
   at least one item as written; after its items are evaluated (where the Expression Language flattens a list-valued
   item), it follows the same rules as a `range` given as a Format String, and so may be empty.
3. The maximum of 1024 elements on a task parameter range list applies to the items of a list literal, counted as
   written. The number of values a `range` given as a Format String evaluates to is not constrained by this
   specification, as for [&lt;IntRangeExpr&gt;](#34111-intrangeexpr). An implementation may impose a maximum number
   of Tasks per Step, and a maximum total number of Tasks per Job. A materialized parameter space that exceeds the Step
   limit, or that would take the Job's total over the Job limit, is an error when the Step's *parameterSpace* is
   evaluated, before any of its Tasks runs, with a message that names the limit: it fails Job creation if that happens
   then, and the Step otherwise. The Job's total is the sum of the Task counts of every Step, and of every instance of
   a Step of a loop's body (see [3.8.5](#385-running-a-loop)), whose Tasks are known, whatever its state. Retries and
   re-runs add no Tasks to it, and with `TASK_CHUNKING` Tasks are counted, not chunks. When two Steps' Tasks become
   known after Job creation, which of them exceeds the Job limit can depend on the order in which they are evaluated.

#### 3.4.1. `<TaskParameterDefinition>`

Definition of a single task parameter; its name, type, and the range of values that it takes.

```bnf
<TaskParameterDefinition> ::= <IntTaskParameterDefinition> |
                              <FloatTaskParameterDefinition> |
                              <StringTaskParameterDefinition> |
                              <PathTaskParameterDefinition> |
                              <ChunkIntTaskParameterDefinition> # @extension TASK_CHUNKING
```

##### 3.4.1.1. `<IntTaskParameterDefinition>`

An integer valued task parameter. An `<IntTaskParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "INT"
range: <IntRangeList> | <IntRangeExpr>
```

With:

```bnf
<IntRangeList> ::= [ <integer> | <intstring> | <TaskParameterStringValue>, ... ]
```

Where `<intstring>` is a string whose value is the string representation of an integer value in base-10,
`<TaskParameterStringValue>` (See [&lt;TaskParameterStringValue&gt;](#342-taskparameterstringvalue)) must resolve to the
string representation of an integer value in base-10, each with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *name* — The name of the parameter.
2. *type* — The literal "INT", defining this parameter as integer valued.
3. *range* — The list of values that the parameter takes on to define Tasks of the Step.
4. `<IntRangeList>` is subject to the constraints:
   * Minimum number of elements: If provided, then this list must contain at least one element.
   * Maximum number of elements: The list must not contain more than 1024 elements.

The value of a task parameter of this type can be referenced in format strings that will be evaluated when running a Task
using the following names:

1. `Task.Param.<name>` and
2. `Task.RawParam.<name>`

###### 3.4.1.1.1. `<IntRangeExpr>`

This is a [Format String](#73-format-strings) with an expression for providing an array of integer values using a succinct
string representation. The motivating use-case for this form is providing a succinct way to describe a frame range, but
this has been generalized to apply to more than simple positive integer frame numbers.

```bnf
<IntRangeExpr> ::= <Element> | <Element>,<IntRangeExpr>
<Element>      ::= <WS>*<Int><WS>* | <WS>*<Range><WS>* | <WS>*<SkipRange><WS>*
<Range>        ::= <Int><WS>*-<WS>*<Int>
<SkipRange>    ::= <Range>:<Skip>
<Int>          ::= Any integer value (positive, negative, or zero)
<Skip>         ::= base-10 non-zero number
<WS>           ::= whitespace character: tabs or spaces
```

With:

1. `<Int>` — The specific number given
2. `<Range>` = `x-y` —  The set of values $`\{x\} \cup\{ x+m: m\in\mathbb{Z}^+, x+m\leq y\}`$
3. `<SkipRange>` = `x-y:n`— The set of values
   $`\{x\}\cup\{x+mn: m\in\mathbb{Z}^+, x+mn\leq y\textrm{ if }n>0, x+mn\geq y\textrm{ if }n<0\}`$

Subject to the constraint that no two elements of the expression overlap. The *span* of an element is the interval
from the smallest to the largest value in its set, not its written endpoints, so the span of "1-10:4" is 1 to 9 and
the span of "19-23:5" is the single value 19. Two elements overlap when their spans share at least one integer, even
when their sets share no value: "1-9:2,2-10:2" is an error, since the spans 1 to 9 and 2 to 10 intersect.

The number of values that an `<IntRangeExpr>` expands to is not constrained by this specification. In particular, the maximum of 1024 elements that §3.4.1.1 places on an `<IntRangeList>` does not apply to this form; expressing a range of more than 1024 values, such as a long frame range, is the motivating use-case for it. An implementation that needs to bound the number of Tasks a Step may define is expected to do so with its own limit on the Task count, not by constraining this expression.

The elements of the range expression are combined to form a list of values in increasing order.

The value of a range expression is this list; the way it is written, including the order of its elements, how its
values are split into elements, and white space, is not part of the value.

For example:

|`<IntRangeExpr>`|**List of values**|
|---|---|
|"1 - 5"|`[1, 2, 3, 4, 5]`|
|"1 - -1"|`[1]`|
|"-1 - 1"|`[-1,0,1]`|
|"1-5:2"|`[1,3,5]`|
|"10-15:2,1-5"|`[1,2,3,4,5,10,12,14]`|
|"1-10:4"|`[1,5,9]`|
|"1-10:4,10-15"|`[1,5,9,10,11,12,13,14,15]`|
|"1-10:4,9-15"|Error: ranges overlap|
|"1-9:2,2-10:2"|Error: ranges overlap|

**Canonical form.** Each range expression value has one *canonical form*, a string determined by its list of values
alone. Wherever this specification writes a range expression value as a string, it is written in its canonical form,
unless the specification gives that value another form (as it does for a `CONTIGUOUS` chunk; see
[&lt;ChunkIntTaskParameterDefinition&gt;](#3415-chunkinttaskparameterdefinition-extension-task_chunking)).

For a list of values $`v_1 < v_2 < \dots < v_n`$, the canonical form is built by splitting the list, from the left,
into runs:

1. A run starts at the first value not yet in a run, $`v_i`$. If $`v_i`$ is the last value, the run is $`v_i`$ alone.
2. Otherwise let $`s = v_{i+1} - v_i`$, and extend the run to the last value $`v_j`$ such that every two consecutive
   values from $`v_i`$ to $`v_j`$ differ by $`s`$.
3. If the run has three or more values ($`j \geq i + 2`$), it is $`v_i, \dots, v_j`$, and the next run starts at
   $`v_{j+1}`$. Otherwise the run is $`v_i`$ alone, and the next run starts at $`v_{i+1}`$.

Each run is written as `x` when it is the single value `x`; as `x-y` when its values are `x` to `y` with step 1; and
as `x-y:s` when they are `x` to `y` with step `s` > 1. Integers are written in base 10, with a leading `-` when
negative, and with no `+` sign and no leading zeros. The runs are joined by `,`, with no white space.

The canonical form is a valid `<IntRangeExpr>` whose list of values is the original list. Parsing it and writing the
result in canonical form again gives the same string, and two range expressions have the same canonical form exactly
when they have the same list of values.

|List of values|**Canonical form**|
|---|---|
|`[5]`|"5"|
|`[1,2]`|"1,2"|
|`[1,2,3]`|"1-3"|
|`[1,3,5,7,9]`|"1-9:2"|
|`[3,7]`|"3,7"|
|`[1,3,4,5]`|"1,3-5"|
|`[1,2,4]`|"1,2,4"|
|`[1,2,3,5,7,9]`|"1-3,5-9:2"|
|`[2,4,6,8,9,10]`|"2-8:2,9,10"|
|`[13,14,15,16,17,18,19]`|"13-19"|
|`[-3,-2,-1]`|"-3--1"|
|`[-2,-1,3,4,5,7,8,9,12,100]`|"-2,-1,3-5,7-9,12,100"|

So "1-10:4" is written "1-9:4", "5-1:-1" is written "1-5", "1,3,5,7,9" is written "1-9:2", and "19-23:5,13-18" and
"13-18,19" are both written "13-19".

A range expression can denote very many values with few elements, and an implementation should compute the canonical
form without listing the values. One way (informative): represent the value as a sequence of arithmetic progressions
in increasing order, each with a start, a positive step, and a count. Each `<Int>`, `<Range>`, or `<SkipRange>`
element of the grammar is one progression. Walk the runs as above. Extending a run along a progression whose step is
$`s`$ jumps to the end of that progression in one move, and crossing to the next progression compares its first value
with the current one. Each progression is visited a bounded number of times, so the work is proportional to the number
of progressions and never to the number of values. The canonical form has at most two runs per progression, so an
implementation that keeps a range expression as the runs of its canonical form, and builds new values from those runs,
does this work in time proportional to the number of runs. Reading an arbitrary `<IntRangeExpr>` is proportional to
its number of elements, which can exceed its number of runs: "1,2,3,4,5,6" has six elements and one run. The step of a
run of three or more values fits in a 64-bit signed integer; the difference between two consecutive values may not, so
an implementation computes $`s`$ in wider arithmetic or checks for overflow first.

##### 3.4.1.2. `<FloatTaskParameterDefinition>`

A floating point valued task parameter. An `<FloatTaskParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "FLOAT"
range: <FloatRangeList>
```

With:

```bnf
<FloatRangeList> ::= [ <float> | <floatstring> | <TaskParameterStringValue>, ... ]
```

Where `<floatstring>` is a string whose value is the string representation of a floating point value in base-10,
`<TaskParameterStringValue>` (See [&lt;TaskParameterStringValue&gt;](#342-taskparameterstringvalue)) must resolve to the
string representation of a floating point value in base-10, each with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *name* — The name of the parameter.
2. *type* — The literal "FLOAT", defining this parameter as floating point valued.
3. *range* — The list of values that the parameter takes on to define Tasks of the Step.
4. `<FloatRangeList>` is subject to the constraints:
   * Minimum number of elements: If provided, then this list must contain at least one element.
   * Maximum number of elements: The list must not contain more than 1024 elements.

The value of a task parameter of this type can be referenced in format strings that will be evaluated when running a Task
using the following names:

1. `Task.Param.<name>` and
2. `Task.RawParam.<name>`

##### 3.4.1.3. `<StringTaskParameterDefinition>`

A string valued task parameter. A `<StringTaskParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "STRING"
range: [ <TaskParameterStringValue>, ... ] # @fmtstring
```

1. *name* — The name of the parameter.
2. *type* - The literal "STRING", defining this parameter as string valued.
3. *range* — The list of values that the parameter takes on to define tasks of the step.
   See: [&lt;TaskParameterStringValue&gt;](#342-taskparameterstringvalue).
    1. Minimum list length: One element
    2. Maximum list length: 1024 elements

The value of a task parameter of this type can be referenced in format strings that will be evaluated when running a Task
using the following names:

1. `Task.Param.<name>` and
2. `Task.RawParam.<name>`

##### 3.4.1.4. `<PathTaskParameterDefinition>`

A string valued task parameter that represents an absolute path to a file or directory, and will automatically have any
defined path mapping applied to it. A `<PathTaskParameterDefinition>` is the object:

```yaml
name: <Identifier>
type: "PATH"
range: [ <TaskParameterStringValue>, ... ] # @fmtstring
```

1. *name* — The name of the parameter.
2. *type* — The literal "PATH", defining this parameter as string valued path parameter.
3. *range* — The list of values that the parameter takes on to define tasks of the step.
   See: [&lt;TaskParameterStringValue&gt;](#342-taskparameterstringvalue).
    1. Minimum list length: One element
    2. Maximum list length: 1024 elements

The value of a task parameter of this type can be referenced in format strings that will be evaluated when running a Task
using the following names:

1. `Task.Param.<name>` — the value of the parameter with relevant path mapping rules applied to it; and
2. `Task.RawParam.<name>` — the value of the parameter as it was defined, with no path mapping rules applied.

##### 3.4.1.5. `<ChunkIntTaskParameterDefinition>` `@extension TASK_CHUNKING`

An integer valued task parameter, processed as chunks instead of as individual elements.
At most one `CHUNK[INT]` parameter can be specified in a step parameter space. When forming
a chunk to run, all the non-chunked dimensions take on a single value, and the chunked dimension
takes a set of values that satisfies the range constraint. A `<ChunkIntTaskParameterDefinition>` is the object:

```yaml
  name: <Identifier>
  type: "CHUNK[INT]"
  range: <IntRangeList> | <IntRangeExpr>
  chunks:
    defaultTaskCount: <integer> | <intstring> # @fmtstring
    targetRuntimeSeconds: <integer> | <intstring> # @optional @fmtstring
    rangeConstraint: "CONTIGUOUS" | "NONCONTIGUOUS"
```

See section `<IntTaskParameterDefinition>` for the definitions of `<IntRangeExpr>` and `<intstring>`.

1. *name* — The name of the parameter.
2. *type* — The literal "CHUNK[INT]", defining this parameter as integer valued and processed
     as chunks.
3. *range* — The list of values that the parameter takes on to define Tasks of the Step.
4. *chunks* — Specifies how to form sets of values into chunks.
    1. *defaultTaskCount* — How many tasks to combine into a single chunk by default.
        1. Minimum value: 1
    2. *targetRuntimeSeconds* — If provided and its value is greater than 0, the number of seconds
        to aim for when forming chunks. A scheduler can ignore this, or dynamically adjust
        the chunk task count to be closer to this value once some chunks have completed.
        When the value is 0, a scheduler should ignore this configuration and use `defaultTaskCount`
        for each chunk.
        1. Minimum value: 0
    3. *rangeConstraint* — If CONTIGUOUS, a chunk must always be a contiguous range
        of integers, written as two integers separated by "-", `<N>-<M>`, like a single "5-5" or interval "1-10",
        even when it has one or two values. This is the only range expression value this specification does not
        write in canonical form.
        If NONCONTIGUOUS, a chunk can be an arbitrary set of integers, written in the canonical form of
        [§3.4.1.1.1](#34111-intrangeexpr) like "1,3,7,9" or "1-3,5-9:2". Frame ranges are inclusive, so "10-20"
        contains 11 frames.
5. `<IntRangeList>` is subject to the constraints:
   * Minimum number of elements: If provided, then this list must contain at least one element.
   * Maximum number of elements: The list must not contain more than 1024 elements.

The value of a task parameter of this type can be referenced in format strings that will be evaluated when running a Task
using the following names:

1. `Task.Param.<name>` and
2. `Task.RawParam.<name>`

#### 3.4.2. `<TaskParameterStringValue>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Allowed characters: Any
2. Minimum length: 1 character.
3. Maximum length: 1024 characters after the format string has been resolved.

With the EXPR extension, an element of a task parameter's `range` list that is exactly `"{{<expr>}}"` is a list
item, which can resolve to no value or to several. See
[Task Parameter Range Field Extensions](2026-02-Expression-Language#1312-task-parameter-range-field-extensions).

#### 3.4.3. `<CombinationExpr>`

The combination expression defines how to combine the task parameters to define the parameter space of the Step. If not
provided then the expression defaults to the product of all defined task parameters. The string takes the form:

```bnf
<CombinationExpr> ::= <CombinationExpr> '*' <Element> | <Element>
<Element>         ::= <Associative> | <Identifier>
<Associative>     ::= '(' <ExprList> ')'
<ExprList>        ::= <ExprList>,<CombinationExpr> | <CombinationExpr>
```

Subject to the following constraints:

1. Allowed characters: Any allowed in an `<Identifier>` plus the space, `*`, `(`, and `)` characters.
2. Minimum length: 1 character.
3. Maximum length: 1280 characters.
4. Each `<Identifier>` in the expression must be the name of a defined task parameter, and each task parameter must occur
   exactly once in the entire expression.
5. Every comma-separated expression within an associative operator must have the exact same number of values defined
   in their range.
6. If a task parameter is chunked, for example it has type `CHUNK[INT]`, it must not be
   combined with any other task parameter using the associative operator. For example if
   `A` is chunked, then `A * B` is permitted but `(A, B)` is an error.

For example, given the four Task Parameters named "A", "B", "C", and "D" with values:

```python
A=[1,2,3]
B=[10,11,12]
C=[20,21]
D=["a","b","c"]
```

The following table lists some expressions and the resulting parameter space.

|**Expression**|**Parameter Space**|
|---|---|
|A * B     |(A=1,B=10), (A=1,B=11), (A=1,B=12), (A=2,B=10), (A=2,B=11), ...|
|(A,B)     |(A=1,B=10), (A=2,B=11), (A=3,B=12)|
|(A,B) * C |(A=1,B=10,C=20), (A=1,B=10,C=21), (A=2,B=11,C=20), (A=2,B=11,C=21), ...|
|(A,B,D)   |(A=1,B=10,D=a), (A=2,B-11,D=b), (A=3,B=10,D=c)|
|(A,C)     |Error: length mismatch (A and C have differing numbers of values)|
|(A,B,A)   |Error: each parameter may only appear once in the expression.|

If `A` has type `CHUNK[INT]`, the parameter space is the same as if it had type `INT`,
but each chunk provided to the `onRun` action specifies an integer range expression for
`A` instead of a single value. The scheduler might select the following chunks when
`rangeConstraint` is set to `CONTIGUOUS`:

(A="1-1", B=10), (A="2-2", B=10), (A="3-3", B=10),
(A="1-2", B=11), (A="3-3", B=11), (A="1-3", B=12)

### 3.5. `<StepScript>`

The Script of a Step defines the properties of the action that the Step runs on a Worker host.

A `<StepScript>` is the object:

```yaml
let: <LetBindings> # @optional @extension EXPR
actions: <StepActions> # @incompatible python | bash | cmd | powershell | node
embeddedFiles: [ <EmbeddedFile>, ... ] # @optional @incompatible python | bash | cmd | powershell | node
python | bash | cmd | powershell | node: <SimpleAction> # @optional @fmtstring[host] @incompatible command embeddedFiles @extension FEATURE_BUNDLE_1
```

Where:

1. *let* — An ordered list of expression bindings evaluated once per task. Bound names are available in *actions*
   and *embeddedFiles* fields, and can reference `Task.Param.*` values. See: [&lt;LetBindings&gt;](#36-letbindings).
2. *actions* — The Actions that are run by Tasks of the Step.
3. *embeddedFiles* — Files embedded into the Step that are materialized to a Session's working directory as the Step's
   Task is running within the Session. See: [&lt;EmbeddedFile&gt;](#6-embeddedfile).
   1. Minimum number of items: If defined, then there must be at least one element in this list.
   2. Maximum number of items: There is no limit on the number of elements in this list.
4. *bash | cmd | node | powershell | python* - Syntactic sugar for scripts,
   removes some commonly needed boilerplate for the given script interpreter.

The format string scopes available to format strings within a `<StepScript>` are:

1. `Param.*` and `RawParam.*` — Values of Job Parameters.
2. `Session.*` — Values such as the Session's working directory.
3. `Task.*` — Values of embedded file locations defined within the `<StepScript>`, and Task Parameters.
4. Names bound by `let` in the enclosing `<StepTemplate>` and `<StepScript>`. Available with the `EXPR` extension.
5. `Steps.*` — Output variables and Task parameter values of the Step's direct dependencies. Available with the
   `CONTROL_FLOW` extension.
   See [Value References](#731-value-references).
6. `Loop.*` and `RawLoop.*` — In a Step of a loop's body, the loop variables and `Loop.Iteration` for the iteration
   that the Step instance belongs to. Available with the `CONTROL_FLOW` extension. In a Step of a loop's body, the
   `Steps.*` values of item 5 are those that [3.8.4](#384-scopes-in-and-around-a-loop) allows.

#### 3.5.1. `<StepActions>`

The Actions of a Step are the set of commands that are run by the Worker Agent to accomplish the goal(s) of the Step.

```yaml
onRun: <Action>
```

Where:

1. *onRun* — The action that is run when a Task for the Step is run on a host. See: [&lt;Action&gt;](#5-action).

### 3.6. `<LetBindings>`

Available when using the `EXPR` extension.

A `<LetBindings>` is an ordered array of `<LetBinding>` strings:

```yaml
let:
  - <LetBinding>
  - <LetBinding>
  ...
```

Bindings are evaluated in declaration order. Later bindings can reference names from earlier bindings in the same
`let` block. A binding that shadows a previous binding in the same `let` block or any enclosing scope is an error.
For example, a `let` binding in a `<StepScript>` cannot shadow a binding from the enclosing `<StepTemplate>`'s `let`
block, and a `let` binding in a `<StepTemplate>`'s *stepEnvironments* cannot shadow a binding from that step's `let` block.

Constraints:
1. Minimum number of items: If defined, then there must be at least one element in this list.
2. Maximum number of items: 50.

#### 3.6.1. `<LetBinding>`

A `<LetBinding>` is a string containing a Python assignment expression:

```bnf
<LetBinding>     ::= <UserIdentifier><WS>*"="<WS>*<Expression>
<UserIdentifier> ::= [a-z_][A-Za-z0-9_]*
<Expression>     ::= (any valid expression as defined by the EXPR extension)
<WS>             ::= whitespace character: tabs or spaces
```

Where:

1. The `<UserIdentifier>` must start with a lowercase letter or underscore. This ensures user-defined names never
   conflict with spec-defined symbols (`Param`, `Task`, `Session`, `Env`, `RawParam`), which always start with an
   uppercase letter.
2. Minimum length of `<UserIdentifier>`: 1 character.
3. Maximum length of `<UserIdentifier>`: 512 characters.
4. The `<Expression>` is evaluated according to the rules defined by the `EXPR` extension. See:
   [Expression Language Specification](2026-02-Expression-Language).
5. The type of the binding is the natural result type of the expression.

Examples:

```yaml
let:
  - frame_count = Param.EndFrame - Param.StartFrame + 1
  - output_dir = Param.OutputRoot / Param.ProjectName
  - files = [Param.InputDir / f for f in Param.FileNames]
```

#### 3.6.2. Let Binding Scope Summary

The following table summarizes where `let` bindings can appear, what symbols they can reference,
and where the bound names are available:

| Location | Can Reference | Bound Names Available In |
|----------|---------------|--------------------------|
| `<StepTemplate>.let` | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name`, earlier bindings in same `let` | *stepEnvironments*, *hostRequirements*, *parameterSpace*, *script* (including nested `<StepScript>.let` or `<SimpleAction>.let`) |
| `<StepScript>.let` | `Param.*`, `RawParam.*`, `Task.Param.*`, `Task.RawParam.*`, `Session.*`, `Task.File.*`, `Job.Name`, `Step.Name`, step-level bindings, earlier bindings in same `let` | *actions*, *embeddedFiles* |
| `<SimpleAction>.let` | `Param.*`, `RawParam.*`, `Task.Param.*`, `Task.RawParam.*`, `Session.*`, `Job.Name`, `Step.Name`, step-level bindings, earlier bindings in same `let` | *script*, *args* |
| `<EnvironmentScript>.let` | `Param.*`, `RawParam.*`, `Session.*`, `Env.File.*`, `Job.Name`, earlier bindings in same `let` | *actions*, *embeddedFiles* |

Note: `Task.Param.*` and `Task.RawParam.*` are only available in `<StepScript>` and `<SimpleAction>` contexts because task parameter
values are not known until task execution time.

Note: With the `CONTROL_FLOW` extension, `<StepTemplate>.let`, `<StepScript>.let`, `<SimpleAction>.let`, and the
`<EnvironmentScript>.let` of a Step Environment may also reference the `Steps.*` values of the Step's direct
dependencies, and in a Step of a loop's body, the `Loop.*` and `RawLoop.*` values and the `Steps.*` values that
[&lt;StepLoop&gt;](#38-steploop-extension-control_flow) makes available there (see
[3.8.4](#384-scopes-in-and-around-a-loop)). The `<EnvironmentScript>.let` of a Job Environment may not. A `<StepTemplate>.let` binding, whether or not it references
`Steps.*`, is evaluated with the first materialization group of the Step that uses it, and not at all if the Step is
skipped, at Job creation or at step input binding (see
[&lt;StepTemplate&gt;](#3-steptemplate)). A group evaluates the bindings it uses in declaration order, so a binding
is never evaluated before one it reads, but a later binding may be evaluated before an earlier one that it doesn't
read.

Note: `Task.File.*` symbols in `<StepScript>.let` and `Env.File.*` symbols in `<EnvironmentScript>.let` refer to
embedded files defined in the same script. The file path is determined before evaluation, so `let` bindings can
reference the path where the file will be written, even though the file content (which may also contain format
strings) is evaluated separately.

Note: `Job.Name` is concrete at job creation, so it is available in every `let` scope. `Step.Name` is scoped to the
Step Template (see [7.3.1](#731-value-references)), so it is available in the Step-side `let` scopes but not in
`<EnvironmentScript>.let`, which may also appear in a Job-level environment or a standalone environment template where
no enclosing Step exists.

### 3.7. `<OutputVariableDefinition>` `@extension CONTROL_FLOW`

Defines a value that every Task of a Step reports, for the Steps that depend on it to read and for the render
management system to store and display. A metric, such as an error score or a frame rate, is an output variable that
no Step reads. Introduced in [RFC 0012](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0012-step-output-variables.md) (draft).

An `<OutputVariableDefinition>` is the object:

```yaml
name: <Identifier>
type: <OutputVariableType>
collect: <OutputCollectMode> # @optional
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# @optional constraint properties: see the table below.
```

```bnf
<OutputVariableType> ::= "INT" | "FLOAT" | "STRING" | "PATH" | "BOOL" | "RANGE_EXPR" |
                         "LIST[INT]" | "LIST[FLOAT]" | "LIST[STRING]" | "LIST[PATH]" | "LIST[BOOL]" |
                         "LIST[LIST[INT]]"
<OutputCollectMode>  ::= "ONCE" | "LAST" | "APPEND"
```

Where:

1. *name* — The name by which the value is reported in an `openjd_output_var:` directive and referenced as
   `Steps.<StepName>.Output.<name>`. See: [&lt;Identifier&gt;](#71-identifier).
2. *type* — The type of the value each Task reports. Type names are case-insensitive. Each type maps to the expression
   type of the job parameter type of the same name, and accepts that job parameter definition's constraint properties
   with the same meaning:

   | `<OutputVariableType>` | Expression type of one Task's value | Constraint properties, as on |
   |---|---|---|
   | `INT` | `int` | [§2.3](#23-jobintparameterdefinition): `allowedValues`, `minValue`, `maxValue` |
   | `FLOAT` | `float` | [§2.4](#24-jobfloatparameterdefinition): `allowedValues`, `minValue`, `maxValue` |
   | `STRING` | `string` | [§2.1](#21-jobstringparameterdefinition): `allowedValues`, `minLength`, `maxLength` |
   | `PATH` | `path` | [§2.2](#22-jobpathparameterdefinition): `allowedValues`, `minLength`, `maxLength`, `objectType` |
   | `BOOL` | `bool` | [§2.9](#29-jobboolparameterdefinition-extension-expr): none |
   | `RANGE_EXPR` | `range_expr` | [§2.10](#210-jobrangeexprparameterdefinition-extension-expr): `minLength`, `maxLength` |
   | `LIST[INT]` | `list[int]` | [§2.13](#213-joblistintparameterdefinition-extension-expr): `minLength`, `maxLength`, `item` |
   | `LIST[FLOAT]` | `list[float]` | [§2.14](#214-joblistfloatparameterdefinition-extension-expr): `minLength`, `maxLength`, `item` |
   | `LIST[STRING]` | `list[string]` | [§2.11](#211-jobliststringparameterdefinition-extension-expr): `minLength`, `maxLength`, `item` |
   | `LIST[PATH]` | `list[path]` | [§2.12](#212-joblistpathparameterdefinition-extension-expr): `minLength`, `maxLength`, `item`, `objectType` |
   | `LIST[BOOL]` | `list[bool]` | [§2.15](#215-joblistboolparameterdefinition-extension-expr): `minLength`, `maxLength` |
   | `LIST[LIST[INT]]` | `list[list[int]]` | [§2.16](#216-joblistlistintparameterdefinition-extension-expr): `minLength`, `maxLength`, `item` |

3. *collect* — How the directives a Task prints for this output become the Task's value. Default `ONCE`.
    1. `ONCE` — Exactly one directive. A second directive, or none, fails the Task.
    2. `LAST` — One or more directives. The last one is the value. None fails the Task.
    3. `APPEND` — Zero or more directives. The *type* must be a `LIST[...]` type, each directive supplies one element,
       and the value is the list of elements in the order the directives were read. With none, the value is `[]`.
       A runtime that reads stdout and stderr separately does not define their relative order, so a Task should
       print all the directives for an `APPEND` or `LAST` output on one stream.
4. *description* — A description of the value. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
5. *unit* — The unit of the values, such as `mm`, `%`, or `frames/s`, for display. It has no functional purpose, and
   values are not converted. Allowed characters: any Unicode character except those in the Cc category. Length: 1 to
   32 characters.
6. Constraint properties — Checked against each value as the Task reports it, so under `collect: LAST` every reported
   value must satisfy them. A value that violates one fails the Task. For `collect: APPEND`, `item` constraints and
   `maxLength` are checked as each element is reported, and `minLength` when the Task's action exits. Only the
   properties the definition declares are checked: a default implied by a job parameter definition, such as the
   1024-character `maxLength` of a `RANGE_EXPR` parameter, does not apply. The `default` and `userInterface`
   properties of job parameter definitions are not available.

Constraints:

1. `LIST[LIST[INT]]` may only be declared by a Step without a *parameterSpace*.
2. A constraint property must be valid for the declared type, with the same rules as on the job parameter definition
   of that type.
3. `collect: APPEND` requires a `LIST[...]` *type*.

A `PATH` or `LIST[PATH]` value is stored in the submission path context (see
[3.7.1](#371-reporting-output-values)), and its constraint properties are checked against that stored form.
`objectType` describes the value and is not checked; `dataFlow` is not available.

#### 3.7.1. Reporting output values

A Task reports a value by printing one of these lines to stdout or stderr from its `onRun` action (see
[How Jobs Are Run](How-Jobs-Are-Run#stdoutstderr-messages)):

```
openjd_output_var: <Name>=<JSON>
openjd_output_var: <Name>=@<FilePath>
openjd_output_var: <Name>[<Int>]=<JSON>        # either form, in a Step with a CHUNK[INT] parameter
openjd_output_var: <Name>[<Int>]=@<FilePath>
```

```bnf
<Directive>    ::= <WS>* <Identifier> <TaskSelector>? "=" <WS>* ( <JSON> | "@" <FilePath> ) <WS>*
<TaskSelector> ::= "[" <Int> "]"
<WS>           ::= " " | "\t"
```

Where:

* The line terminator (`\n` or `\r\n`) is not part of the directive, and the line must be valid UTF-8.
* `<Identifier>` is the output's name, matched case-sensitively against the Step's declared outputs. There is no
  whitespace between it, the `<TaskSelector>`, and the `=`.
* `<Int>` is a decimal integer, an optional `-` then digits, with no whitespace inside the brackets.
* `<JSON>` is one JSON value on the rest of the line. A string that contains a line break is written with the escape
  `\n`. `NaN`, `Infinity`, and `-Infinity` are not JSON.
* `<FilePath>` is the rest of the line after `@`, with trailing spaces and tabs removed. It must not be empty.

The runtime recognizes a directive only in a Session running a Job whose Job Template lists `CONTROL_FLOW`; in any other
Session the line is ordinary output. For example:

```bash
echo "openjd_output_var: FrameCount=$count"
echo "openjd_output_var: Keys=@keys.json"
echo "openjd_output_var: RenderTimeS[$frame]=$seconds"   # in a Step with a CHUNK[INT] parameter
```

In a Step with a `CHUNK[INT]` Task parameter, one action runs a chunk of Tasks, and every directive must have a
`<TaskSelector>`: the decimal value of the `CHUNK[INT]` parameter for the Task it reports for, which must be a value in
the chunk being run, not merely of the parameter's whole range. In a Step that declares *outputVariables*, the
`CHUNK[INT]` parameter's values must be distinct, so that a value names exactly one Task. When its `range` is a list
literal, its integer elements are checked at template validation (a range expression cannot repeat a value); format
string elements and computed ranges are checked when the parameter space is materialized, where a duplicate fails the
Step (or fails Job creation, if that is when it is materialized). In any other Step a `<TaskSelector>` is not allowed. Collect modes, constraint properties, and the
size limit apply to each Task of the chunk separately, and when the action exits successfully each Task must have its
required values; if any Task doesn't, or the action fails, the whole chunk fails and no Task of it commits any
output; a Task of the chunk that had already succeeded keeps its earlier state and values. The selector is matched by integer value and is required even in a chunk of one Task.

A relative `<FilePath>` is resolved against the Session's Working Directory; an absolute one is used as written. Either
way, the file the runtime opens must be a regular file inside the Session's Working Directory, after following symbolic links, and the runtime must read it
with no more access than the user that runs the action. A runtime should open the path without blocking and check
that it is a regular file before reading, because opening a FIFO for reading blocks. The file holds exactly
one UTF-8 JSON value, which may span lines and be preceded by a byte order mark. The runtime does not modify or delete
the file. The runtime reads the file when it reads the directive, so the action must finish writing the file before it
prints the directive.

The runtime processes each directive as it reads it. Each JSON value is given its natural expression type (a number
without a fraction or exponent that fits in a signed 64-bit integer is an `int`; any other number a `float`; a string
a `string`; `true`/`false` a `bool`; an array a list, whose elements are typed by the same rule and which is inferred
as a list literal with the target type, as in
[List Literal Type Inference](2026-02-Expression-Language#126-list-literal-type-inference)) and is converted to the declared type, or for `collect: APPEND`
to its element type, with the [implicit type coercion](2026-02-Expression-Language#123-implicit-type-coercion) rules of
the Expression Language. `null` and objects are not accepted anywhere in a value, and a number too large to represent
as a `float`, or nesting deeper than the target type, does not convert.

A directive is invalid, and cancels and fails the action it is attributed to as soon as it is read, whatever the exit
code, using the action's normal cancelation, with a message that names the output, the problem, and the offending text,
if it: does not match the grammar or is not valid JSON; names an undeclared output; has a value that
is `null`, an object, or does not convert; violates a constraint property; repeats a `collect: ONCE` output (for the
same Task, in a chunk); lacks a required `<TaskSelector>`, has one where none is allowed, or names a value not in the
chunk; names a file that is not a regular file inside the Session's Working Directory, is unreadable, or is not UTF-8,
contains NUL, or is not exactly one JSON value; is on a line that is not valid UTF-8; is on a line longer than the
runtime's line limit, which must be at least 64 KiB (65,536 bytes of UTF-8; an over-long directive is never truncated
and parsed, and its discarded remainder is never recognized as a directive); has a `PATH` value that is relative or
has no submission-context form (see below); or takes the size of the values the Task holds over the runtime's limit.
That limit must be at least 1 MiB (1,048,576 bytes) per Task attempt, per Task of a chunk. The size counted is that
of the values the Task currently holds, as the bytes that reported them (the inline `<JSON>` text without the
whitespace around it, or the whole content of an `@` file): the `ONCE` value, the current `LAST` value (a value it
replaces stops counting), and every `APPEND` element. For a `PATH` or `LIST[PATH]` value, or a `PATH` element under
`APPEND`, the larger of its reported bytes and the [JSON serialization](#217-joboutputdefinition-extension-control_flow)
of its stored form counts. When reading an `@` file, the runtime reads at most as many bytes as remain of that limit
(not counting the current value of a `LAST` output that the directive replaces), plus one to detect a file that
exceeds it. The render management system should encourage authors to declare `maxLength` on list outputs that feed a
parameter space, and on `APPEND` outputs. The Task also fails if, when the action
exits, a `ONCE` or `LAST` output has no value or an `APPEND` output has fewer elements than its *minLength*. In a Job
that lists `CONTROL_FLOW`, a directive printed by any action of the Job Template other than the `onRun` of a Task of a
Step that declares *outputVariables* cancels and fails that action. Under `WRAP_ACTIONS`, a line from a wrap hook counts
as output of the action it runs in place of, so `onWrapTaskRun` reports the Task's outputs whichever template defines
it. For an action of an Environment Template, a directive is ordinary output. If the attempt fails for any reason, it
has no outputs. Like the other messages, a directive line appears in the action's log by default. A runtime option to
hide recognized messages may hide a valid directive, but an invalid one always stays in the log with the error.

A `PATH` value, or each element of a `LIST[PATH]` value, is stored in the **submission path context**, the form in which
`PATH` Job Parameter values are given, by applying the Session's path mapping rules in reverse. A relative path is
invalid. A URI is stored unchanged, and so is any absolute value when the Session has no path mapping rules (before any filtering by source format).
Otherwise
the stored value must map forward, with the producing Session's rules, back to the reported path (compared after lexical
normalization, a root keeping its separator, under the rules of the worker host's path format: Windows paths
case-insensitively with `\` or `/` separators, POSIX paths case-sensitively); if the runtime finds no such form, the
value is invalid. A runtime may find the stored form however it likes, and one with more knowledge of the farm's
storage may do better than the rules alone, but a runtime that has only the rules should use this reference inverse. It
normalizes the path and each rule's `destination_path` lexically in the same way, compares them under the rules of the
worker host's path format, and matches each rule's `destination_path` against the path on a component boundary. Each
matching rule gives a candidate: the components of the path after `destination_path`, appended unchanged and joined
with the separator of the rule's `source_path_format` (`/` for `URI`), then joined to `source_path` with one such
separator unless `source_path` already ends with one; if no components remain, the candidate is `source_path`. The
candidates are tried longest `destination_path` first and then in the order the rules are listed, and the first that
maps forward back to the reported path is stored; if none does, the value is invalid. A runtime that knows the
submission path format should use only rules whose `source_path_format` is that format or `URI`.

The values are committed when the Task succeeds, and are retained by the render management system for the life of the
Job. The render management system should make each Task's committed values visible to users, for example beside the
Task's parameters, using *unit* and *description*. They may be forwarded as soon as they are reported, and displayed before then if marked as not yet committed, but
only committed values are visible to consumers through `Steps.*`. They are not redacted and must not be used to pass
secrets.

### 3.8. `<StepLoop>` `@extension CONTROL_FLOW`

A loop: an element of the Job Template's *steps* whose *body*, a list of Steps, runs once per *iteration*, one
iteration after another, until a condition says to stop. State passes from one iteration to the next through typed
*loop variables*. Introduced in [RFC 0013](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0013-step-loops.md) (draft).

A `<StepLoop>` is the object:

```yaml
name: <StepName>
description: <Description> # @optional
enableIf: <bool> | <FormatString> # @optional @fmtstring
dependencies: [ <StepDependency>, ... ] # @optional
loop: <LoopDefinition>
outputVariables: [ <LoopOutputVariableDefinition>, ... ] # @optional
```

Where:

1. *name* — The loop's name. It must differ from the names of all other Steps of the Job Template, including the
   Steps in every loop's body. See: [&lt;StepName&gt;](#31-stepname).
2. *description* — As on [&lt;StepTemplate&gt;](#3-steptemplate).
3. *enableIf* — Whether the loop runs, with the meaning and constraints of a `<StepTemplate>`'s *enableIf*. If it is
   `false`, the loop is skipped: no iteration runs, none of its other fields is evaluated, and its `Steps.<name>.*`
   values are `null`.
4. *dependencies* — The elements of the Job Template's *steps* that the loop depends on. Its first iteration starts
   only when every one is satisfied, and these are the Steps whose `Steps.*` values the loop's expressions and the
   Steps of its body may read. See: [&lt;StepDependency&gt;](#32-stepdependency).
5. *loop* — See: [&lt;LoopDefinition&gt;](#381-loopdefinition).
6. *outputVariables* — The loop's outputs, read by the Steps that depend on the loop and by Job outputs as
   `Steps.<name>.Output.<VarName>`. See: [&lt;LoopOutputVariableDefinition&gt;](#383-loopoutputvariabledefinition).
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: 32.
    * No two elements in this list may have the same value for the `name` property.

A `<StepLoop>` has no other properties. It has no Tasks of its own. Elsewhere in this specification, where a Step is
named as a dependency, has a state, or is skipped, a loop is a Step.

#### 3.8.1. `<LoopDefinition>`

```yaml
variables: [ <LoopVariableDefinition>, ... ] # @optional
while: <bool> | <FormatString> # @optional @fmtstring
until: <bool> | <FormatString> # @optional @fmtstring
maxIterations: <int> | <FormatString> # @fmtstring
steps: [ <StepTemplate>, ... ]
```

Where:

1. *variables* — The loop variables. See: [&lt;LoopVariableDefinition&gt;](#382-loopvariabledefinition).
    * Minimum number of elements: If provided, then this list must contain at least one element.
    * Maximum number of elements: 32.
    * No two elements in this list may have the same value for the `name` property.
2. *while* — Checked before each iteration, including the first: the iteration runs only if it is `true`. If it is
   `false` before the first iteration, the loop runs zero iterations and, once its dependencies are satisfied,
   succeeds if every `APPEND` output's `minLength` holds for `[]`. Its target type is `bool`, and a `null` result is
   an error. It may reference `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`,
   `Job.Name`, `Loop.*` and `RawLoop.*`, and the `Steps.*` values of the loop's *dependencies*, but not the Steps of
   the loop's body.
3. *until* — Checked after each iteration: if it is `true`, it ends the loop, which succeeds if every `APPEND`
   output's `minLength` holds. A loop with *until* runs at
   least once. Its target type is `bool`, and a `null` result is an error. It may reference everything *while* may,
   and the `Steps.*` values of every Step of the loop's body, for the iteration that just finished.
4. *maxIterations* — The most iterations the loop may run, an integer from 1 to 1,000. A format string here may
   reference only `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`, and `Job.Name`. A loop that
   would start another iteration after running *maxIterations* iterations fails.
5. *steps* — The loop's body: one or more `<StepTemplate>`s (see [&lt;StepTemplate&gt;](#3-steptemplate)). A
   `<StepLoop>` may not appear here: loops do not nest. The *dependencies* of a Step of the body name only Steps of the
   same body, and must not form a cycle.

Exactly one of *while* and *until* must be provided.

#### 3.8.2. `<LoopVariableDefinition>`

```yaml
name: <Identifier>
type: <OutputVariableType>
initial: <FormatString> # @fmtstring
next: <FormatString> # @fmtstring
description: <Description> # @optional
# @optional constraint properties, as on <OutputVariableDefinition>.
```

Where:

1. *name* — The name by which the variable is read, as `Loop.<name>` and `RawLoop.<name>`. It may not be
   `Iteration`. See: [&lt;Identifier&gt;](#71-identifier).
2. *type* — The variable's type. See: [&lt;OutputVariableDefinition&gt;](#37-outputvariabledefinition-extension-control_flow):
   the same types, expression types, and constraint properties. `LIST[LIST[INT]]` is allowed.
3. *initial* — The value for the first iteration. Its target type is the expression type of *type*, and a `null`
   result is an error. It may reference `Param.*` (except `PATH` and `LIST[PATH]` parameters), `RawParam.*`,
   `Job.Name`, and the `Steps.*` values of the loop's *dependencies*. It may not reference `Loop.*` or `RawLoop.*`.
4. *next* — The value for the next iteration, evaluated at each iteration boundary at which *until*, if present, did
   not end the loop, before *while* and the *maxIterations* bound; so a loop with *while* evaluates it at its final
   boundary too. Its target type and `null` rule are those of *initial*. It may reference everything *until* may.
   Every *next* reads the values of the iteration that just finished, not another variable's new value.
5. *description* — For display only.
6. Constraint properties — Checked against every value the variable takes. A violation fails the loop.

A loop variable's value is converted to *type* as a Job output's *value* is (see
[2.17](#217-joboutputdefinition-extension-control_flow)), except that the type is not made optional, so a `null`
result is an error.

A `PATH` or `LIST[PATH]` variable's value is in the submission path context, because its expressions read `PATH`
values through `RawParam.*`, `Steps.<StepName>.RawOutput.*`, `Steps.<StepName>.TaskRawParam.*`, and `RawLoop.*`.

#### 3.8.3. `<LoopOutputVariableDefinition>`

```yaml
name: <Identifier>
type: <OutputVariableType>
collect: <LoopOutputCollectMode>
value: <FormatString> # @fmtstring
description: <Description> # @optional
unit: <OutputVariableUnit> # @optional
# @optional constraint properties, as on <OutputVariableDefinition>.
```

```bnf
<LoopOutputCollectMode> ::= "LAST" | "APPEND"
```

Where:

1. *name* — The name by which the output is read, as `Steps.<LoopName>.Output.<name>`.
2. *type* — As on [&lt;OutputVariableDefinition&gt;](#37-outputvariabledefinition-extension-control_flow).
   `LIST[LIST[INT]]` is allowed.
3. *collect* — Required. `LAST`: the output is the value of the last iteration. `APPEND`: the *type* must be a
   `LIST[...]` type, each iteration supplies one element, and the output is the list of them in iteration order.
4. *value* — The iteration's value, evaluated after each iteration. Its target type is the expression type of *type*,
   or under `APPEND` of its element type, and a `null` result is an error. It may reference everything *until* may.
5. *description*, *unit* — For display only, as on `<OutputVariableDefinition>`.
6. Constraint properties — Under `LAST`, checked against each iteration's value; under `APPEND`, `item` constraints
   and `maxLength` as each element is added, and `minLength` when *until* or *while* ends the loop, before it
   succeeds, including after zero iterations, but not when the loop is skipped. A violation fails the loop.

The type of `Steps.<LoopName>.Output.<name>`, where `T` is the expression type of *type*:

| Loop | `collect: LAST` | `collect: APPEND` |
|---|---|---|
| has *until*, no *enableIf* | `T` | `T` |
| has *while*, no *enableIf* | `T?`, `null` after zero iterations | `T`, `[]` after zero iterations |
| has *enableIf* | `T?`, also `null` when skipped | `T?`, `null` when skipped |

`Steps.<LoopName>.TaskParam.*` and `Steps.<LoopName>.TaskRawParam.*` are not available: a loop has no Task
parameters.

A loop's outputs are committed, and its `Steps.<LoopName>.Output.*` and `RawOutput.*` values become concrete, when the
loop succeeds or is skipped. A loop that fails commits no outputs. The render management system should show each
iteration's values as they are evaluated, and may show a running loop's outputs so far if it marks them as not yet
committed. Loop outputs and loop variables are stored and displayed, so they are not redacted and must not be used to
pass secrets.

#### 3.8.4. Scopes in and around a loop

* A Step of a loop's body may reference, in the fields where `Steps.*` is available (not *combination*), `Loop.*` and
  `RawLoop.*` (see [Value References](#731-value-references)), with a `Loop.*` value whose type is or contains `path`
  only in format strings annotated `@fmtstring[host]` (`RawLoop.*` elsewhere), the `Steps.*` values of the Steps of the same body that it lists in its
  own *dependencies*, which are those Steps in the same iteration, and the `Steps.*` values of the Steps that the loop
  lists in its *dependencies*. It may not reference any other Step, including the Steps of the body in an earlier
  iteration.
* No element of the Job Template's *steps*, and no Job output, may reference a Step of a loop's body. They reference
  the loop's outputs instead.
* `Loop.*` and `RawLoop.*` are available only in the Steps of the loop's body and in the loop's *while*, *until*,
  *next*, and output *value*s.
* With a future task-task dependencies extension (`TASK_DEPENDENCIES`), a loop's own *dependencies* must be whole-Step
  dependencies, an element of *steps* may not have a subspace dependency on a loop, which has no Tasks, and a Step of
  a loop's body may have subspace dependencies only on Steps of the same body, selecting Tasks of their instances in
  the same iteration, with subspace ranges that may reference `Loop.*` and `RawLoop.*`.

#### 3.8.5. Running a loop

The loop's *enableIf* is decided first, as a Step's is. Then, in this order, its *maxIterations*, each variable's
*initial* in declaration order, and, with *while*, the first *while* are evaluated, as materialization expressions of
the loop, each value converted and checked against its variable's type and constraints. When the loop's dependencies are
satisfied it runs its iterations in order, starting with iteration 0. Each iteration creates a new instance of every
Step of the body, with its own Tasks, states, and output values, and the body's Steps run as the Steps of a Job do, in
the order of their *dependencies*. `Loop.*` values are fixed for the whole iteration. The next iteration starts only
when every Step of the current one has completed successfully or been skipped. After each iteration, in this order:
every loop output's *value* is evaluated; *until*, if present, is checked, and if it is `true` the loop ends; every
variable's *next* is evaluated; *while*, if present, is checked with the new values, and if it is `false` the loop ends;
and if the loop has run *maxIterations* iterations, it fails, with a message that names *maxIterations* and its value.
An error in any of these steps fails the loop at once, with a message that names the field, the expression, the
iteration, and the problem, and no later step is evaluated. A loop whose first *while* is `false` ends with zero
iterations once its dependencies are satisfied. When a loop ends, each `APPEND` output's `minLength` is checked, and
then the loop succeeds; this check is not a materialization expression and is never part of Job creation.

The limits of a loop are:

1. *maxIterations* is at most 1,000, and an implementation must support every value from 1 to 1,000.
2. An implementation's maximum number of Tasks per Step (see [3.4](#34-stepparameterspacedefinition)) applies to each
   instance of a Step of the body. An implementation may also impose a maximum on the total number of Tasks of all
   instances of a loop, or on the number of Step instances a loop creates. An instance whose parameter space would take
   the loop over such a limit fails when its *parameterSpace* is evaluated, and an iteration that would take it over
   the instance limit fails the loop when it would start; the message names the limit. The maximum total number of
   Tasks per Job counts the Tasks of every Step instance of every iteration, and an instance that would take the Job
   over it fails in the same way. A loop has no Tasks of its own, so it adds nothing to a Task count.
3. A loop and each Step of its body count once each as a Step of the Job Template, as an element of *steps* does,
   wherever this specification or an implementation counts the Steps of a Job Template. Step instances are not Steps
   of the template, and the number of iterations does not change the count.
4. An implementation must accept loop variable and loop output values totaling at least 1 MiB (1,048,576 bytes) per
   loop at any moment, and may accept more. The size counted is that of the values the loop currently holds, each as
   its [JSON serialization](#217-joboutputdefinition-extension-control_flow): the current value of each loop variable
   (a value that *next* replaces stops counting), the current value of each `LAST` output (nothing before its first
   iteration's value), and each element of each `APPEND` output, counted on its own, so the brackets and separators of
   the list do not count. The total is checked after the *initial* values are evaluated (where exceeding it fails Job
   creation if they are evaluated then), and after each iteration once the outputs and once the *next* values are
   evaluated.

A failed Step instance in the iteration in progress, or an error in any of the loop's expressions, fails the loop at
once, and no further iteration starts. Other Step instances of the same iteration that do not depend on the failed
one may still run, as independent Steps of a Job may after a Step fails. Only a failure of the Tasks of a Step instance
of the iteration in progress can be recovered, as any failed Step can: a loop whose failure is recoverable is `FAILED`
until every failed body Step instance of the iteration in progress has succeeded after a retry or requeue of its Tasks;
it then returns to `RUNNING`, and the iteration continues to its boundary. Until then the Steps that depend on it are
blocked, and a Job output that reads it is not evaluated. Every other failure of a loop is final: an error in a
materialization expression of a Step instance or of the loop (its *enableIf*, *maxIterations*, an *initial*, or the
first *while*), exceeding a limit of the list above, including an instance whose Tasks would take the loop or the Job
over a Task limit, an error in *while*, *until*, *next*, or an output *value*, a violated constraint, including
`minLength` when the loop ends, and reaching *maxIterations*. Once any final failure occurs, the loop cannot return to
`RUNNING`, even if a failed Task of the iteration in progress is later retried or requeued and succeeds.

Any `Steps.*` output that a loop's *while*, *until*, *next*, *initial*, or output *value* references counts as consumed
(see [How Jobs Are Run](How-Jobs-Are-Run#step-states-and-step-input-binding)) once that expression has been evaluated,
so a re-run of a Task of a completed iteration that succeeds with a different value for it is reported as a failed
attempt under the consumed-output rule, and the Task keeps its earlier values. Loop variables and
loop outputs are never evaluated again, and an iteration is never run again. A re-run of a body Task that has already
succeeded leaves the Task and its Step instance `SUCCEEDED`
(see [RFC 0012 §Retries and re-runs](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0012-step-output-variables.md#retries-and-re-runs)),
so it never fails the loop or changes its state, variables, or outputs, and the iteration boundary does not wait for
it. Only a failed attempt of a Task that has not yet succeeded counts as a failure of a Step instance. The loop's
outputs are available to the Steps that depend on it when it succeeds.

A Step instance is a Step for Sessions: a Session that runs Tasks of two instances, of the same Step of the body in two
iterations or of two Steps of the body, exits the first instance's Step Environments and enters the second's. A Session
may run instances of different iterations, keeping its Job Environments entered, but need not. With the `ENV_PARAMS`
extension, the instances of one Step of the body enter the same Job Environments and attached Environments with the
same parameter values in every iteration, since their Environment selection reads no `Loop.*` or `Steps.*` value (see
[4](#4-environment)), and their Step Environments may differ.

The render management system should show each iteration with its index, the loop variable values it ran with, its
Step instances with their states and Tasks, and the loop outputs evaluated at its boundary, and for the loop as a whole
the iteration in progress, the number of iterations completed, and *maxIterations*. A loop's Task count is not known
until the loop ends; it is reported as not yet known, with the Tasks of the iterations so far. See
[How Jobs Are Run](How-Jobs-Are-Run#step-loops).

## 4. `<Environment>`

The Environment is a mechanism provided in this specification to enable users to amortize expensive, or time-consuming,
setup and tear-down operations of the worker host's environment over a sequence of Tasks run back-to-back on the worker.
Each Environment defines actions to run on a host when entering or exiting the Environment.

An `<Environment>` is the object:

```yaml
name: <EnvironmentName>
description: <Description> # @optional
script: <EnvironmentScript> # @optional
variables: <EnvironmentVariables> # @optional
```

Where:

1. *name* — An identifier given to the environment that is unique within the Environment's defined scope: the Job
   Template for a Job Environment, and the Step Template for a Step Environment.
2. *description* — A description to apply to the environment. It has no functional purpose, but may appear in UI elements.
   See: [&lt;Description&gt;](#72-description).
3. *script* — The action that is taken by this Environment when it is run on a Worker host.
4. *variables* — A set of environment variable name/value pairs, with the values being
   [Format Strings](#73-format-strings) that are resolved when entering the environment. The specified variables must be
   set prior to running either `onEnter` or `onExit` for the environment, and for all actions that are run with the
   environment active.
5. At least one of "script" or "variables" must be provided.

The `ENV_PARAMS` extension gives an Environment, and a Step's *environment* dependency entry, an *enableIf*, decided
for each Step when the Job is created, except as follows in a Job Template that also lists `CONTROL_FLOW`:

1. The *enableIf* of a Step Environment may also reference the `Steps.*` values of the enclosing Step's direct
   dependencies, directly or through the Step's *let* bindings. In a Step of a loop's body, it may also reference
   `Loop.*`, `RawLoop.*`, and the `Steps.*` values that [3.8.4](#384-scopes-in-and-around-a-loop) allows. It is
   evaluated with the last of the Step's materialization groups, which holds its *stepEnvironments* and *script* fields
   (see [&lt;StepTemplate&gt;](#3-steptemplate)), before the Environment's format strings, which are not evaluated if it
   is `false`. A failure, including a `null` result, fails the Step, or Job creation for a group evaluated then. No
   other Environment field of `ENV_PARAMS` may reference `Steps.*`, `Loop.*`, or `RawLoop.*`.
2. An *environment* entry's *parameters* values and *enableIf* may not reference `Steps.*`, `Loop.*`, or `RawLoop.*`,
   directly or through a *let* binding that does.
3. What `ENV_PARAMS` does for one Step when the Job is created is the Step's *Environment selection*: evaluating its
   entries' *enableIf*, checking its enabled entries' keys and values and rejecting an enabled entry with
   `required: true` that resolves to no Environment, deciding which of the Job's Environments and attached
   Environments its Sessions enter and with which parameter values, applying `exclusiveGroup`, checking constraint 7,
   the `ALWAYS` rules, and the values, including a non-literal `default` that the Step takes, and resolving the
   Environments' *hostRequirements* with the Step's values. The *enableIf* of its Step Environments (rule 1), the
   Step's own host requirements (rule 4), and the other checks over its whole set of Environments (rule 5) are not
   part of it. It is made once the Step's own *enableIf* is `true`, before any other materialization group of the Step
   is evaluated, which is when the Job is created for a Step whose *enableIf* is absent or decided `true` then.

   The work that does not depend on which Steps enter an Environment stays at Job creation, whatever any Step's
   *enableIf*: attaching the Environment Templates, resolving every *environment* entry that names no Job Environment
   against them and rejecting a name that matches two of them, checking literal `default`s, and rejecting an attached
   `ALWAYS` Environment with the name of a Job Environment and a parameter without a `default`. So do the checks of
   every entry as written, which template validation makes. A skipped Step has no Environment selection, so no
   Environment is entered, no entry or Environment *enableIf* is evaluated, and no other check of its entries or
   Environments is made for it.

   A check that `ENV_PARAMS` makes for one Step at template validation, other than those of an entry as written, is
   made then only if the Step is known to be enabled: its *enableIf* is absent or the literal `true`, and for a Step of
   a loop's body the loop's *enableIf* is too and the loop has no *while*. Otherwise it is made in the Step's
   Environment selection, or never if the Step is skipped.
4. A Step's combined host requirements are formed, and every check of them is made, when its *hostRequirements* are
   evaluated, and its host requirements are known when they are.
5. A check over the whole set of Environments a Step's Sessions enter, other than constraint 7 and the value checks,
   such as the `WRAP_ACTIONS` single-layer rule ([&lt;EnvironmentActions&gt;](#43-environmentactions)), is made, from
   Job creation on, as soon as its outcome no longer depends on an *enableIf* whose value is not yet known, the Step's
   own included: at the latest when the last of the Step's materialization groups is evaluated. A literal `true` or
   `false` is known.

An error in the Step's Environment selection, its combined host requirements, or a check of rule 5 fails the Step at
once, or fails Job creation if it is found then.

The format string scopes available to format strings within an Environment are:

1. `Param.*` and `RawParam.*` — Values of Job Parameters.
2. `Session.*` — Values such as the Session's working directory.
3. `Env.*` — Scope of the environment entity itself. Values such as the embedded files defined within the Environment
   entity.
4. Names bound by `let` in the enclosing `<EnvironmentScript>`. Available with the `EXPR` extension.
5. `Steps.*` — In a Step Environment, the `Steps.*` values of the enclosing Step's direct dependencies. Available
   with the `CONTROL_FLOW` extension. See [Value References](#731-value-references).
6. `Loop.*` and `RawLoop.*` — In a Step Environment of a Step of a loop's body, the loop variables and
   `Loop.Iteration` for the iteration, and there the `Steps.*` values of item 5 are those that
   [3.8.4](#384-scopes-in-and-around-a-loop) allows. Available with the `CONTROL_FLOW` extension.
7. `WrappedAction.*` — Available within `onWrapEnvEnter`, `onWrapTaskRun`, and `onWrapEnvExit` only. Carries
   the wrapped `<Action>`'s fields. Available with the `WRAP_ACTIONS` extension.
   See [Wrap hook template variables](#431-wrap-hook-template-variables) for the available fields.
8. `WrappedEnv.Name` — Available within `onWrapEnvEnter` and `onWrapEnvExit` only. The name of the inner
   environment whose action is being wrapped. Available with the `WRAP_ACTIONS` extension.
9. `WrappedStep.Name` — Available within `onWrapTaskRun` only. The name of the step whose task is being
   wrapped. Available with the `WRAP_ACTIONS` extension.

Templates must not reference `WrappedAction.*` outside the three wrap hooks, `WrappedEnv.*` outside
`onWrapEnvEnter` and `onWrapEnvExit`, or `WrappedStep.*` outside `onWrapTaskRun`. Schedulers must reject
templates that violate this scope rule.

Implementations of this specfication must watch STDOUT when running the `onEnter` action for any line matching:

1. The regular expression `^openjd_env: (.*)$`. The captured value must be of the form `<varname>=<value>` where
   `varname` is the name of an environment variable, and `value` is the value to assign to it. The defined value of the
   given variable will be set for all actions that are run with the environment active.
2. The regular expression `^openjd_redacted_env: (.*)$`. The captured value must be interpreted identically to
   openjd_env, and `<value>` shall be redacted with a fixed length string for this and any future occurrences from the
   emitted STDOUT message. Requires the REDACTED_ENV_VARS extension, however supporting applications must honor the
   redaction even when the extension is not specified.
3. The regular expression `^openjd_unset_env: (.*)$`. The captured value must be of the form `<varname>` where
   `varname` is the name of an environment variable. The given environment variable will be unset as long as this
   environment is active. If an environment both sets and unsets a particular environment variable, then the unset takes
   precedence.

### 4.1. `<EnvironmentName>`

A string value subject to the following constraints:

1. Allowable characters: Any unicode character except those in the Cc unicode character category.
2. Minimum length: 1 characters.
3. Maximum length: 64 characters. 512 characters if using the
`FEATURE_BUNDLE_1` extension.

### 4.2. `<EnvironmentScript>`

An `<EnvironmentScript>` is the object:

```yaml
let: <LetBindings> # @optional @extension EXPR
actions: <EnvironmentActions>
embeddedFiles: [ <EmbeddedFile>, ... ] # @optional
```

1. *let* — An ordered list of expression bindings evaluated once when the environment is entered. Bound names are
   available in *actions* and *embeddedFiles* fields. See: [&lt;LetBindings&gt;](#36-letbindings).
2. *actions* — The actions to run at different stages of the Environment's lifecycle.
3. *embeddedFiles* — Files embedded into the Environment that are materialized to a Session's working directory as the
   Environment is running within the Session. See: [&lt;EmbeddedFile&gt;](#6-embeddedfile).
   1. Minimum number of items: If defined, then there must be at least one element in this list.
   2. Maximum number of items: There is no limit on the number of elements in this list.

### 4.3. `<EnvironmentActions>`

An `<EnvironmentActions>` is the object:

```yaml
onEnter: <Action> # @optional
onWrapEnvEnter: <Action>    # @optional @extension WRAP_ACTIONS
onWrapTaskRun: <Action>     # @optional @extension WRAP_ACTIONS
onWrapEnvExit: <Action>     # @optional @extension WRAP_ACTIONS
onExit: <Action> # @optional
```

1. *onEnter* — The action run when the environment is being entered on a host.
2. *onWrapEnvEnter* — When provided, runs instead of the `onEnter` action of every *inner* environment that
   enters the session while this environment is active. Available only when using the `WRAP_ACTIONS`
   extension.
3. *onWrapTaskRun* — When provided, runs instead of the task's `onRun` action for every task that runs
   while this environment is active. Available only when using the `WRAP_ACTIONS` extension.
4. *onWrapEnvExit* — When provided, runs instead of the `onExit` action of every *inner* environment that
   exits while this environment is active. Available only when using the `WRAP_ACTIONS` extension.
5. *onExit* — The action run when the environment is being exited on a host.

   > **NOTE:** When *onExit* action does not define a *timeout* the action will default to 300
   seconds, or five minutes. Job schedulers may provide the ability to cancel jobs/steps/tasks. A
   reasonable default expectation should be that OpenJobDescription sessions are able to end and
   cleanup within a bound amount of time.

Subject to the constraint that at least one of *onEnter* or *onExit* must be provided. (With the
`WRAP_ACTIONS` extension, a complete set of wrap hooks also satisfies this requirement.)

> **`WRAP_ACTIONS` extension constraints.**
>
> 1. *All-or-nothing rule.* An environment that defines any of `onWrapEnvEnter`, `onWrapTaskRun`, or
>    `onWrapEnvExit` must define all three. Schedulers must reject templates that define only a subset of
>    the wrap hooks before the session begins.
> 2. *Single-layer rule.* A session must contain at most one environment that defines wrap hooks.
>    Schedulers must reject sessions whose stack contains two or more such environments before
>    entering any environment.
> 3. *EXPR prerequisite.* A template that lists `WRAP_ACTIONS` in `extensions:` must also list `EXPR`.
>    Schedulers must reject templates that list `WRAP_ACTIONS` without also listing `EXPR`.
> 4. The wrapping environment's own `onEnter` and `onExit` are never wrapped; they always run
>    normally.
> 5. *Nothing-to-replace rule.* A wrap hook runs only in place of an action the inner entity
>    defines. When an inner environment defines no `onExit` — or defines no `script` at all (a
>    `variables:`-only environment) — there is nothing to replace, and the corresponding wrap
>    hook must not run for that environment. Every `<StepScript>` defines `onRun`, so
>    `onWrapTaskRun` runs for every task.

#### 4.3.1. Wrap hook template variables

Each wrap hook receives the wrapped `<Action>`'s fields as read-only template variables supplied by the
runtime. The variables have identical names and semantics across all three hooks, so helper scripts can
be reused unchanged.

**Available in `onWrapEnvEnter`, `onWrapTaskRun`, and `onWrapEnvExit`:**

| Variable                          | Type           | Description |
|-----------------------------------|----------------|-------------|
| `WrappedAction.Command`           | `string`       | The `command` from the wrapped action. |
| `WrappedAction.Args`              | `list[string]` | The `args` from the wrapped action. |
| `WrappedAction.Environment`       | `list[string]` | Environment variables defined earlier in the session — `openjd_env` exports and environments' declarative `variables:` maps — as `["KEY=value", ...]`. |
| `WrappedAction.Timeout`           | `int?`         | The timeout in seconds specified on the wrapped action. `null` when the wrapped action specifies no timeout. See the *timeout* defaults table in [&lt;Action&gt;](#5-action). |
| `WrappedAction.Cancelation.Mode`  | `string?`      | The cancelation method of the wrapped action (`TERMINATE` or `NOTIFY_THEN_TERMINATE`). `null` when the wrapped action defines no [&lt;CancelationMethod&gt;](#53-cancelationmethod). |
| `WrappedAction.Cancelation.NotifyPeriodInSeconds` | `int?` | The effective `notifyPeriodInSeconds` of the wrapped action when its cancelation mode is `NOTIFY_THEN_TERMINATE`, with the [&lt;CancelationMethodNotifyThenTerminate&gt;](#532-cancelationmethodnotifythenterminate) defaults applied when the wrapped action omits the field. `null` when the mode is `TERMINATE` or the wrapped action defines no `<Cancelation>`. |

**Additionally available in `onWrapEnvEnter` and `onWrapEnvExit`:**

| Variable          | Type     | Description |
|-------------------|----------|-------------|
| `WrappedEnv.Name` | `string` | The `name` of the inner environment whose action is being wrapped. |

**Additionally available in `onWrapTaskRun`:**

| Variable           | Type     | Description |
|--------------------|----------|-------------|
| `WrappedStep.Name` | `string` | The `name` of the step whose task is being wrapped. |

`WrappedAction.Environment` carries only session-defined variables: variables exported with `openjd_env` —
including by earlier actions that themselves ran via a wrap hook — and entries of entered environments'
declarative `variables:` maps. Host-inherited variables (`HOME`,
`PATH`, `OPENJD_*`, etc.) and host filesystem paths referenced in `WrappedAction.Command` or
`WrappedAction.Args` (for example, `{{Env.File.X}}` resolves to a host path) are not surfaced; the wrapping
environment is responsible for forwarding or path-mapping them into the wrapped execution context.

The `WrappedAction.*` namespace is the single, standardized surface for every field of the wrapped
`<Action>`. When a future specification adds a field to `<Action>`, it must surface under the same
namespace using the field's PascalCased name (e.g., `WrappedAction.LogMessageTimeout`). New fields gated
by an `@extension` activate under this namespace exactly when that extension is active in the template.
Wrap scripts may reference any field they recognize and must tolerate fields they do not.

The wrap hooks and these template variables were introduced in
[RFC 0008](https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/rfcs/0008-environment-wrap-actions.md).

### 4.4. `<EnvironmentVariables>`

An `<EnvironmentVariables>` is a map from `<EnvironmentVariableNameString>`s to `<EnvironmentVariableValueString>`s:

```yaml
<EnvironmentVariableNameString>: <EnvironmentVariableValueString>, # @fmtstring[host]
...
```

#### 4.4.1. `<EnvironmentVariableNameString>`

A string value subject to the following constraints:

1. Allowable characters: Alphanumeric characters in the latin character set, and the underscore ('_') character.
   * Note: This is as recommended in
     [Chapter 8 of IEEE Std 1003.1](https://pubs.opengroup.org/onlinepubs/000095399/basedefs/xbd_chap08.html).
2. Minimum length: 1 character.
3. Maximum length: 256 characters.
4. The first character cannot be a digit.

#### 4.4.2. `<EnvironmentVariableValueString>`

A string value subject to the following constraints:

1. Allowable characters: Any
2. Minimum length: 0 characters
3. Maximum length: 2048 characters

## 5. `<Action>`

An Action is a specific command with arguments that is run on a host. An `<Action>` is the object:

```yaml
command: <CommandString> # @fmtstring[host]
args: [ <ArgString>, ... ] # @optional @fmtstring[host]
timeout: <posinteger> # @optional
timeout: <posinteger> | <posintstring> # @optional @fmtstring @extension FEATURE_BUNDLE_1
cancelation: <CancelationMethod> # @optional
```

Where `<posintstring>` is a string whose value is the string representation of a
positive integer value in base-10, with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *command* — A [Format String](#73-format-strings) containing the name of a runnable command that is run on a Worker
   host.
2. *args* — An array of [Format Strings](#73-format-strings) that will be passed as arguments to the **command** when the
   command is run on the host.
3. *timeout* — The positive number of seconds that the command is given to successfully run to completion. A command that
   does not return before the timeout is canceled and is treated as a failed
   run. Can only be a `<posintstring>` when using the extension `FEATURE_BUNDLE_1`.
   When the value is a single whole-field interpolation expression (`"{{ ... }}"` with no
   surrounding text), its target type is `int?` per the
   [expression evaluation type](2026-02-Expression-Language#expression-evaluation-types) rules:
   a `null` result is treated as if the field were not provided (the defaults below apply), and a
   non-null result MUST be a positive integer.

   The default timeout, if not provided, depends on the action:

   | Action container | Action | Default |
   | --- | --- | --- |
   | `<StepActions>` | `onRun` | *no timeout* |
   | `<EnvironmentActions>` | `onEnter` | *no timeout* |
   | `<EnvironmentActions>` | `onWrapEnvEnter` <sup>2</sup> | *no timeout* |
   | `<EnvironmentActions>` | `onWrapTaskRun` <sup>2</sup> | *no timeout* |
   | `<EnvironmentActions>` | `onWrapEnvExit` <sup>2</sup> | 300 seconds (five minutes) <sup>1</sup> |
   | `<EnvironmentActions>` | `onExit` | 300 seconds (five minutes) <sup>1</sup> |

   <sup>1</sup> Environment exit actions are treated specially. Job schedulers may provide the
   ability to cancel jobs, steps, and tasks. A reasonable default expectation should be that
   OpenJobDescription sessions are able to end and cleanup within a bound duration of time.

   <sup>2</sup> Available with the `WRAP_ACTIONS` extension. The wrap hook's `timeout` field bounds
   the wrap script on the host (infrastructure-side); the `WrappedAction.Timeout` template variable
   carries the wrapped action's timeout (workload-side). When the wrapped action specified no
   timeout, the variable is `null` (it is typed `int?`). Wrap scripts must propagate the wrapped
   timeout to the underlying execution runtime where the runtime exposes a timeout mechanism, and
   should enforce it in-script when the runtime exposes none. Wrap scripts must treat `null` as
   "no timeout" and omit the underlying runtime's timeout flag in that case.

4. *cancelation* — If defined, provides details regarding how this action should be canceled. If not provided, then it is
   treated as though provided with `<CancelationMethodTerminate>`.

The host uses the return code of the **command** run to determine success or failure of the Action. A zero exit code
indicates success, and any non-zero exit code indicates failure. A timeout also indicates failure.

### 5.1. `<CommandString>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Characters allowed: Any unicode character except those in the Cc unicode character category. Note that the specific
   operating system that the command is run on will impose its own additional restrictions on permitted characters.
2. Minimum length: 1 character.
3. Maximum length: There is no maximum string length imposed by this specification. Note that the specific operating
   system that the command is run on will impose its own maximum length.

### 5.2. `<ArgString>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Characters allowed: Any unicode character except those in the Cc unicode character category. Note that the specific
   operating system that the command is run on will impose its own additional restrictions on permitted characters.
2. Minimum length: 0 characters.
3. Maximum length: There is no maximum string length imposed by this specification. Note that the specific operating
   system that the command is run on will impose its own maximum length.

### 5.3. `<CancelationMethod>`

The cancellation method defines the process by which an action is canceled.

```bnf
<CancelationMethod> ::= <CancelationMethodTerminate> | <CancelationMethodNotifyThenTerminate>
```

The *mode* value selects between the two forms. When using the `FEATURE_BUNDLE_1` extension, *mode*
may instead be a [Format String](#73-format-strings):

```yaml
mode: "TERMINATE" | "NOTIFY_THEN_TERMINATE"
mode: "TERMINATE" | "NOTIFY_THEN_TERMINATE" | <fmtstring> # @fmtstring @extension FEATURE_BUNDLE_1
notifyPeriodInSeconds: <posinteger> | <posintstring> # @optional @fmtstring @extension FEATURE_BUNDLE_1
```

When *mode* is a format string, the runtime resolves it when it prepares to run the action:

1. If the resolved value is `"TERMINATE"` or `"NOTIFY_THEN_TERMINATE"`, the object is interpreted as
   the corresponding `<CancelationMethod>` form, and MUST then validate against that form's schema.
   A *notifyPeriodInSeconds* whole-field expression that resolves to `null` is treated as if the
   field were not provided (the schema defaults apply).
2. When the value is a single whole-field interpolation expression (`"{{ ... }}"` with no
   surrounding text), its target type is `string?` per the
   [expression evaluation type](2026-02-Expression-Language#expression-evaluation-types) rules: a
   `null` result causes the entire `<CancelationMethod>` object to be treated as if it were not
   provided — the action is canceled as if by `<CancelationMethodTerminate>`, exactly as when the
   template author declares no *cancelation*.
3. Any other resolved value is an error; the runtime MUST fail the action.

The primary use of a format-string *mode* is RFC 0008 wrap hooks forwarding the wrapped action's
cancelation verbatim (see [wrap hook template variables](#431-wrap-hook-template-variables)):

```yaml
cancelation:
  mode: "{{WrappedAction.Cancelation.Mode}}"
  notifyPeriodInSeconds: "{{WrappedAction.Cancelation.NotifyPeriodInSeconds}}"
```

This forwards every case faithfully: a declared mode is reproduced verbatim; a `null` mode (the
wrapped action declared no `<Cancelation>`) drops the whole *cancelation* object; and a `null`
notify period (mode is `TERMINATE` or undeclared) drops that field so the schema default applies.

#### 5.3.1. `<CancelationMethodTerminate>`

An action defined to cancel by this method cancels the running command by sending it a terminal signal.

A `<CancelationMethodTerminate>` is the object:

```yaml
mode: "TERMINATE"
```

The signal sent to the command is:

1. On Posix systems — Send SIGKILL to the entire process tree when a cancel is requested.
2. On Windows systems — Terminate the entire process tree when a cancel is requested.

#### 5.3.2. `<CancelationMethodNotifyThenTerminate>`

An action defined to cancel by this method cancels the running command by sending it a notification signal that it should
gracefully shut down, waiting for a period of time for the command to exit, and then sending it a terminal signal if it has
not exited by the end of the waiting period.

A `<CancelationMethodNotifyThenTerminate>` is the object:

```yaml
mode: "NOTIFY_THEN_TERMINATE"
notifyPeriodInSeconds: <posinteger> # @optional
notifyPeriodInSeconds: <posinteger> | <posintstring> # @optional @fmtstring @extension FEATURE_BUNDLE_1 # @optional
```

Where `<posintstring>` is a string whose value is the string representation of a
positive integer value in base-10, with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *notifyPeriodInSeconds* — Defines the maximum number of seconds between the two signals. It is possible that the actual
   duration allowed in a particular cancel event will be less than this amount
   if circumstances warrant. Can only be a `<posintstring>` when using the
   extension `FEATURE_BUNDLE_1`.
   1. Maximum value: 600
   2. Defaults:
      * 120 if the Action is the "onRun" action of a `<StepActions>` object.
      * 30 otherwise.
   3. When the value is a single whole-field interpolation expression (`"{{ ... }}"` with no
      surrounding text), its target type is `int?` per the
      [expression evaluation type](2026-02-Expression-Language#expression-evaluation-types) rules:
      a `null` result is treated as if the field were not provided (the defaults above apply), and
      a non-null result MUST be a positive integer within the maximum.

The signals sent to the command are:

1. On Posix systems — Send a SIGTERM, followed by waiting for the notify period in seconds, and then sending SIGKILL to
   the entire process tree if the command is still running.
2. On Windows systems — THIS IS WORK IN PROGRESS. This document will be updated once the Windows implementation has been determined.

Prior to sending the first signal, a file called `cancel_info.json` is written to the Session working directory. The
contents of this file provide an ISO 8601 time in UTC, in the form `<year>-<month>-<day>T<hour>:<minute>:<second>Z`, at
which the notify period will end. This file is in the
[ECMA-404 JavaScript Object Notation (JSON)](https://www.json.org/json-en.html) interchange format with contents:

```json
{
   "NotifyEnd": "<yyyy>-<mm>-<dd>T<hh>:<mm>:<ss>Z"
}
```

Implementation notes:

1. The application that runs the action must watch for the process exit after SIGTERM to detect whether and when the
   cancelation was successful.
2. A process may receive more than one SIGTERM signal with this mode of cancelation. This can occur when the
   application running the action receives a signal of its own that indicates that it must shut down (near)immediately
   while it is waiting for the grace period of a cancel to elapse. These subsequent signals will be accompanied by a
   change to the `cancel_info.json` file in the working directory.

## 6. `<EmbeddedFile>`

A step or environment script can have data attached to it via this mechanism. The embedded data is made available to the
environment/task action(s) as a file within the Session working directory while being run on a host. This file is
written prior to every one of the corresponding actions each time that they are run. The materialized files' permissions
are read-only by only the user under which the task will be run on the worker host.

```bnf
<EmbeddedFile> ::= <EmbeddedFileText>
```

### 6.1. `<EmbeddedFileText>`

Embedding of a plain text file into the template. The *data* provided in the file is written as a plain text file. This
file is written prior to every one of the script's actions each time that they are run.

```yaml
name: <Identifier>
type: "TEXT"
filename: <Filename> # @optional
runnable: <bool> # @optional
data: <DataString> # @fmtstring[host]
endOfLine: enum("AUTO", "LF", "CRLF") # @optional @extension FEATURE_BUNDLE_1
```

* *name* — The name of the embeded file. This value is used in Format String references to this file.
* *filename* — The filename for the written file. This must strictly be the basename of the filename, and not contain any
  directory pathing. (i.e. `foo.txt` not `dir/foo.txt`). Defaults to a random filename if not provided.
* *runnable* — A boolean `True` value indicates that the file written to disk should have its execute-permissions set
  to true. Defaults to `False` if not provided.
* *data* — The string data that will be written to the file exactly as it appears.
* *endOfLine* — The line endings that the embedded file will have when written
to disk. If `AUTO` the embedded file will have the default line endings of the
host operating system. Default is `AUTO`. Requires the `FEATURE_BUNDLE_1` extension.

The fully-qualified path of the file written by the host can be referenced in format strings using the following names:

1. `Task.File.<name>` - If the embedded file is part of a `<StepScript>` object; or
2. `Env.File.<name>`- If the embedded file is part of an `<Environment>` object.

#### 6.1.1. `<Filename>`

A string subject to the following constraints:

* Min length: 1
* Max length: 64. 256 if using extension `FEATURE_BUNDLE_1`.
* Characters allowed: Any characters allowed in filenames on the host operating system.

<a name="data-string"></a>

#### 6.1.2. `<DataString>`

A [Format String](#73-format-strings) subject to the following constraints:

1. Allowed characters: Any
2. Minimum length: 1 character.
3. Maximum length: No length limit is imposed by this specification.

## 7. Strings

### 7.1. `<Identifier>`

A string subject to the following constraints:

1. Allowed characters:
   * Unicode alphanumeric characters in the latin character set.
   * The underscore character ('_').
2. Must start with a letter or underscore character.
3. Minimum length: 1 character
4. Maximum length: 64 characters. 512 characters if using extension `FEATURE_BUNDLE_1`.

### 7.2. `<Description>`

A string value subject to the following constraints:

1. Allowable characters:
   * Any unicode character except those in the Cc unicode character category.
   * Additionally: newline, carriage return, and horizontal tab characters.
2. Minimum length: 1 characters
3. Maximum length: 2048 characters

### 7.3. Format Strings

Some strings within the Job Template are Format Strings which may contain one or
more ***string interpolation expressions***. A string interpolation expression
within a format string is denoted by a double-pair of open and closing curly braces as in:

```
{{ <StringInterpExpr> }}
```

Where:

```bnf
<StringInterpExpr> ::= <ValueReference>
<ValueReference>   ::= <Name>
<Name>             ::= <Name> "." <Identifier> | <Identifier>
```

The `<ValueReference>` in a Format String is the dotted name of a value available
in the system (See: ...). A Format String is resolved by replacing its string
interpolation expressions with the value of the expression. For example, given the
Format String `"The value of Job Parameter 'Name' is: {{ Param.Name }}"` and a value
for the symbol `Param.Name` of "Bob", the resulting resolved string is
`"The value of Job Parameter 'Name' is: Bob"`.

When the `EXPR` extension is enabled, the `<StringInterpExpr>` grammar is extended to support
the full expression language including arithmetic, conditionals, function calls, list operations,
and more. See the [Expression Language](2026-02-Expression-Language#11-extended-format-string-grammar)
specification for the extended grammar, type system, and evaluation semantics.

#### 7.3.1. Value References

|**Value**|**Description**|**Scope**|
|---|---|---|
|`Param.<ParamName>`|Values of the Job parameters are available within the `Param` object. This is the same as `RawParam.<ParamName>` for all parameter types except PATH. For PATH type the value is the input value with applicable path mapping rules applied to it. |All types except PATH are available in every Format String in the Job Template. For PATH type parameters, this is only available within format strings that are within an Environment or StepScript context.|
|`RawParam.<ParamName>`|Values of the Job parameters are available within the `RawParam` object. This is always the exact input value of the job parameter.|Available in every Format String in the Job Template.|
|`Task.Param.<ParamName>`|Values of task parameters are available within the `Task.Param` object. This is the same as `Task.RawParam.<ParamName>` for all parameter types except PATH. For PATH type the value is the input value with applicable path mapping rules applied to it.|Available within the Step Script Actions and Embedded Files.|
|`Task.RawParam.<ParamName>`|Values of task parameters are available within the `Task.Param` object.|Available within the Step Script Actions and Embedded Files.|
|`Task.File.<name>`|The filesystem location to which the Task Embedded File with key `<name>` has been written.| Available within the Step Script Actions and Embedded Files.|
|`Env.File.<name>`|The filesystem location to which the Environment Attachment with key `<name>` has been written.|Available within the Environment Script Actions and Embedded Files.|
|`Session.WorkingDirectory`|The agent is expected to create a local temporary scratch directory for the duration of a Session. This builtin provides the location of that temporary directory. This is the working directory that the Worker Agent uses when running the task.|This is available within all Environment Script Actions & Embedded Files, and all Step Script Actions and Embedded Files.|
|`Job.Name`|The resolved name of the Job. This is a `string` type. Requires the `EXPR` extension.|Available in every Format String in the Job Template, except the `name` field of the Job Template itself.|
|`Step.Name`|The name of the current Step. This is a `string` type. Requires the `EXPR` extension.|Available within the Step Template scope: `stepEnvironments`, `hostRequirements`, `parameterSpace`, and `script`, and with the `CONTROL_FLOW` extension `enableIf`.|
|`Session.HasPathMappingRules`|This value can be used to determine whether path mapping rules are available to the Session. Without the `EXPR` extension, it is string valued, with values "true" or "false". With the `EXPR` extension enabled, it is a `bool` type. "true"/`True` means that the path mapping JSON contains path mapping rules. "false"/`False` means that the contents of the path mapping JSON are the empty object.|This is available within all Environment Script Actions & Embedded Files, and all Step Script Actions and Embedded Files.|
|`Session.PathMappingRulesFile`|This is a string whose value is the location of a JSON file on the worker node's local disk that contains the path mapping rule substitutions for the Session.|This is available within all Environment Script Actions & Embedded Files, and all Step Script Actions and Embedded Files.|
|`Steps.<StepName>.Output.<VarName>`|The value of output variable `<VarName>` of Step `<StepName>`, which must declare it. If the Step has no *parameterSpace* its type is the declared type `T`; otherwise it is `list[T]`, one element per Task in parameter-space iteration order. If the Step has an *enableIf*, the type is optional (`T?` or `list[T]?`) and the value is `null` when the Step was skipped. If `<StepName>` is a loop, `<VarName>` is one of its loop outputs, and its type is given by the table in [3.8.3](#383-loopoutputvariabledefinition). A `PATH` value has the consuming Session's path mapping rules applied. `<StepName>` must be an `<Identifier>`: a Step whose name is not of that form is still valid but cannot be referenced through `Steps.`. Requires the `CONTROL_FLOW` extension.|Available in the Step Template scope (*enableIf*, *let*, *hostRequirements*, *parameterSpace* ranges and its other format strings, *stepEnvironments*, *script*) of a Step that lists `<StepName>` directly in its *dependencies* as a whole-Step dependency (with a future `TASK_DEPENDENCIES` extension, a dependency that selects a subspace of the producer's Tasks is not enough), and in the *value* of a `<JobOutputDefinition>` for any Step except a Step of a loop's body. A value whose type is or contains `path` only in format strings annotated `@fmtstring[host]`. Not available in *combination*, other job-level fields, or Environment Templates. Inside and around a loop, see [3.8.4](#384-scopes-in-and-around-a-loop): a Step of a loop's body reads its body dependencies and the loop's dependencies, a loop's *until*, *next*, and output *value*s read any Step of its body, and a Step of a loop's body is not available outside the loop.|
|`Steps.<StepName>.RawOutput.<VarName>`|The same value as `Steps.<StepName>.Output.<VarName>` with no path mapping applied: a `PATH` value is in the submission path context. Requires the `CONTROL_FLOW` extension.|As for `Steps.<StepName>.Output.*`, including TEMPLATE-scope fields for `PATH` values.|
|`Steps.<StepName>.TaskParam.<ParamName>`|A `list` of the values of Task parameter `<ParamName>`, which Step `<StepName>` must have (a Step without a *parameterSpace* has none), for every Task of Step `<StepName>`, in parameter-space iteration order, aligned element-for-element with the Step's `Output` lists. The element type is the Task parameter's type (`int` for `CHUNK[INT]`). Optional, and `null` when skipped, if the Step has an *enableIf*. Requires the `CONTROL_FLOW` extension.|As for `Steps.<StepName>.Output.*`. For a `PATH` Task parameter, only in format strings annotated `@fmtstring[host]`, with path mapping rules applied.|
|`Steps.<StepName>.TaskRawParam.<ParamName>`|The same values as `Steps.<StepName>.TaskParam.<ParamName>` with no path mapping applied. Requires the `CONTROL_FLOW` extension.|As for `Steps.<StepName>.RawOutput.*`.|
|`Loop.<VarName>`|The value of loop variable `<VarName>` for the current iteration (see [3.8.2](#382-loopvariabledefinition)). Its type is the expression type of the variable's *type*. A `PATH` value has the consuming Session's path mapping rules applied. Requires the `CONTROL_FLOW` extension.|Available in a Step of the loop's body in the fields where `Steps.*` is available (*enableIf*, *let*, *hostRequirements*, *parameterSpace* ranges and its other format strings but not *combination*, *stepEnvironments*, *script*, and, with `TASK_DEPENDENCIES`, the subspace ranges of its *dependencies*), with the values of the iteration the Step instance belongs to, and in the loop's *while* (with the values of the iteration about to start) and *until*, *next*, and output *value*s (with the values of the iteration that just finished). A value whose type is or contains `path` only in format strings annotated `@fmtstring[host]`. Not available in the loop's *enableIf*, *maxIterations*, or *initial*, or anywhere outside the loop.|
|`RawLoop.<VarName>`|The same value as `Loop.<VarName>` with no path mapping applied: a `PATH` value is in the submission path context. Requires the `CONTROL_FLOW` extension.|As for `Loop.<VarName>`, including TEMPLATE-scope fields for `PATH` values.|
|`Loop.Iteration`|The index of the current iteration of the loop, an `int` starting at 0. Requires the `CONTROL_FLOW` extension.|As for `Loop.<VarName>`.|
|`<name>` (let binding)|Names bound by `let` in `<StepTemplate>`, `<StepScript>`, `<SimpleAction>`, or `<EnvironmentScript>`. Names must start with a lowercase letter or underscore (see `<UserIdentifier>`). The type is determined by the expression. Available with the `EXPR` extension.|See [Let Binding Scope Summary](#362-let-binding-scope-summary) for detailed scoping rules.|

With the `CONTROL_FLOW` extension, the `Steps.*` values become concrete as follows. Before then they are `unresolved`,
and an expression that reads one waits (see [Template Processing Stages](#74-template-processing-stages)).

* `Steps.<StepName>.Output.*` and `Steps.<StepName>.RawOutput.*` — when Step `<StepName>` is terminal and has not
  failed: when it has succeeded, with the committed values of all of its Tasks, or when it is skipped, with the value
  `null`. There are no partial values: a list is concrete only when every Task of the Step has succeeded, in
  parameter-space iteration order whatever order the Tasks ran in, and only committed values are ever visible (see
  [3.7.1](#371-reporting-output-values)). A Step with zero Tasks gives `[]`. For a loop, when the loop has succeeded,
  with its collected outputs, or is skipped, with `null` (see [3.8.3](#383-loopoutputvariabledefinition)).
* `Steps.<StepName>.TaskParam.*` and `Steps.<StepName>.TaskRawParam.*` — when Step `<StepName>`'s Tasks are known
  (its *enableIf* is `true` and its *parameterSpace* has been evaluated) or it is skipped (with the value `null`),
  which can be before the Step can run, and is at Job creation for a Step whose *enableIf* and *parameterSpace* read
  only values concrete then.
* A Step of a loop's body has its own values in each iteration, which become concrete by the same rules within that
  iteration. `Loop.*` and `RawLoop.*` are concrete from the start of the iteration.

A Step's *parameter-space iteration order* is the order in which its *combination* enumerates its Tasks (or the
default combination when it has none), flattened with the last parameter varying fastest for a product such as
`A * B`, independent of the order in which its Tasks ran, finished, or were retried. Element `i` of every `Output` list
and every `TaskParam` list of the same Step belongs to the same Task.

### 7.4. Template Processing Stages

A Job Template is processed in stages as more information becomes available. At each stage,
different sets of values are known, which determines which format strings can be fully resolved.

| Stage | When | Known Values | Unknown Values | What Happens |
|-------|------|-------------|----------------|--------------|
| **Template validation** | `openjd check` or equivalent | Literal values, parameter type declarations | `Param.*`, `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*`, and with `CONTROL_FLOW` `Steps.*`, `Loop.*`, and `RawLoop.*` | Syntax validation, structural checks. With the `EXPR` extension: type checking of expressions using declared parameter types. `Job.Name` and `Step.Name` are available as `unresolved[string]`. With the `CONTROL_FLOW` extension: `Steps.*` values are available as `unresolved` values of the types derived from the producing Steps' *outputVariables* and Task parameters, and references to them are checked against the referencing Step's *dependencies*; `Loop.*` and `RawLoop.*` values are available as `unresolved` values of the loop variables' types (see [3.8.4](#384-scopes-in-and-around-a-loop)). |
| **Job creation** | Job submission with parameter values | `Param.*`, `RawParam.*`, and with `CONTROL_FLOW`, for a loop's first *while*, `Loop.*` and `RawLoop.*` for iteration 0 | `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*`, and with `CONTROL_FLOW` the `Steps.*` values that are not yet concrete and every value of a Step of a loop's body | Parameter values are bound. Format strings not annotated `@fmtstring[host]` are resolved (e.g., `name`, `parameterSpace` ranges). PATH and `LIST[PATH]` parameter defaults must be relative and are joined with the job template directory (see [§2.2](#22-jobpathparameterdefinition)); relative PATH and `LIST[PATH]` parameter values are joined with the current working directory. With the `EXPR` extension: `let` bindings in `<StepTemplate>` are evaluated and expressions in TEMPLATE scope are fully evaluated (with the `CONTROL_FLOW` extension, only those of the materialization groups evaluated then, and the `let` bindings those groups use); `Job.Name` and `Step.Name` are concrete. With the `CONTROL_FLOW` extension, in Step dependency order: every Step's *enableIf* whose values are all concrete is evaluated; for each Step whose *enableIf* is then `true` or absent, its Environment selection (`ENV_PARAMS`, see [4](#4-environment)) is made and every materialization group whose `Steps.*` values are all concrete is evaluated, and an error fails Job creation; the other groups wait for step input binding. A loop is treated as a Step here; nothing of a Step of a loop's body is evaluated at Job creation, and the `minLength` check of a loop's `APPEND` outputs is never part of it (see [3.8.5](#385-running-a-loop)). |
| **Step input binding** (`CONTROL_FLOW` only) | Per materialization group of a Step (see [3](#3-steptemplate)), as soon as every `Steps.*` value it reads is concrete, with *enableIf* first | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name`, the referenced `Steps.*`, in a Step instance of a loop's body `Loop.*` and `RawLoop.*`, for a loop's first *while* `Loop.*` and `RawLoop.*` for iteration 0, and with `ENV_PARAMS` `Job.EnvNames` and the values that extension lets its fields read | `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*` | The group's format strings are evaluated. If *enableIf* is `false` the Step is skipped, its parameter space is the empty set (its `Steps.*` values are `null`, not `[]`), and no other group is evaluated; if it is `true`, with the `ENV_PARAMS` extension the Step's Environment selection is made at once (see [4](#4-environment)). When *parameterSpace* is evaluated the Step's Tasks are known, and when *hostRequirements* is, its host requirements are known, in either order. If any evaluation fails, or with `ENV_PARAMS` the Step's Environment selection, combined host requirements, or a check over its set of Environments fails, or the Task count exceeds the implementation's per-Step limit or would take the Job's total over its per-Job limit (see [3.4](#34-stepparameterspacedefinition)), the Step fails at once, before any of its Tasks runs. Until every group has been evaluated the Step is pending inputs, and none of its Tasks is dispatched. |
| **Loop iteration start** (`CONTROL_FLOW` only) | When a loop starts an iteration (see [3.8.5](#385-running-a-loop)) | `Param.*`, `RawParam.*`, `Job.Name`, `Step.Name` of each Step of the body, `Loop.*` and `RawLoop.*` for the iteration, the `Steps.*` values of the loop's dependencies | The `Steps.*` values of the Step instances of the iteration that are not yet concrete, `Task.*`, `Session.*` | A new instance of every Step of the loop's body is created. Each is bound as a Step is at step input binding; a group whose values are all concrete is evaluated now. |
| **Loop iteration boundary** (`CONTROL_FLOW` only) | When every Step instance of an iteration has completed successfully or been skipped | `Param.*`, `RawParam.*`, `Job.Name`, `Loop.*` and `RawLoop.*` for the iteration (for *while*, for the next iteration), the `Steps.*` values of the loop's dependencies and of the iteration's Steps | `Step.Name`, `Task.*`, `Session.*` | The loop outputs' *value*s, then *until*, then every *next*, then *while* with the new values, then the *maxIterations* bound. Any failure fails the loop. |
| **Job output evaluation** (`CONTROL_FLOW` only) | Once per Job output, when every `Steps.*` value its *value* references is concrete, or at Job creation if it references none | `Param.*`, `RawParam.*`, `Job.Name`, the referenced `Steps.*` | `Step.Name`, `Task.*`, `Session.*` | The output's *value* is evaluated, converted to its type, and checked. A failure fails the Job. See [2.17](#217-joboutputdefinition-extension-control_flow). |
| **Task execution** | On the worker host | All values: `Param.*`, `Task.Param.*`, `Session.*`, `Task.File.*`, `Env.File.*`, path mapping rules | *(none)* | Format strings annotated `@fmtstring[host]` are resolved. Path mapping rules are applied to PATH-type parameters. Embedded files are written. With the `EXPR` extension: `let` bindings in `<StepScript>` and `<EnvironmentScript>` are evaluated; all remaining expressions are fully evaluated. With the `CONTROL_FLOW` extension: the `Steps.*` values of the Step's direct dependencies are concrete (and for a Step instance of a loop's body, `Loop.*`, `RawLoop.*`, and the `Steps.*` values of the loop's dependencies), and the `openjd_output_var:` directives that `onRun` prints are processed as they are printed, with `PATH` values mapped back to the submission path context (see [3.7.1](#371-reporting-output-values)). |

With the `CONTROL_FLOW` extension, template validation must also emit a warning (not an error) where the Expression
Language discards a `null` silently: an interpolation with text around it (which becomes the empty string, so
`"--size {{Steps.P.Output.X}}"` becomes `"--size "`), a list item such as an entry of `args` (which is dropped), and
an optional field (which is omitted). The warning applies to every interpolated expression in one of those positions
whose static type includes `nulltype` and that references, directly or through a `let` binding, a `Steps.*` value of
a Step that has an *enableIf*, or a `collect: LAST` output of a loop with *while*. It names the expression and suggests
a conditional expression or an `or` default. An expression whose type has no `nulltype`, such as
`'frames only' if x == null else string(x)`, draws no warning, and neither does *enableIf*, where `null` is an error.

The `@fmtstring` and `@fmtstring[host]` annotations in this specification indicate which stage
a format string is resolved in. A format string annotated `@fmtstring` (without `[host]`) is
resolved at job creation time. A format string annotated `@fmtstring[host]` is resolved at task
execution time on the worker host.

With the `EXPR` extension enabled, expressions that reference values not yet known at a given
stage are type-checked using the declared types of those values, catching type errors as early
as possible. See the [Expression Language](2026-02-Expression-Language) specification for details.

### 7.5. Numeric strings

`<intstring>`, `<posintstring>`, `<floatstring>`, `<nonnegativefloatstring>` and `<positivefloatstring>` are the string
forms of a number. Each may have white space before and after the number, and that white space is ignored:
`" 7 "` is the `<intstring>` 7, and `"2.5\n"` is the `<floatstring>` 2.5. White space inside the number is not allowed,
so `"1 000"` is not an `<intstring>`, and a string that is empty or holds only white space is not a number.

This applies wherever this specification reads a string as one of these forms:

1. a value written in a template, such as a *default*, a number in the *default* or the item constraints of a
   LIST[INT], LIST[FLOAT] or LIST[LIST[INT]] parameter (see [§2.13](#213-joblistintparameterdefinition-extension-expr),
   [§2.14](#214-joblistfloatparameterdefinition-extension-expr) and
   [§2.16](#216-joblistlistintparameterdefinition-extension-expr)), or an element of a *range* list;
2. the string a [Format String](#73-format-strings) resolves to where one of these forms is expected, such as a
   `<TaskParameterStringValue>` element of an `<IntRangeList>` or a `chunks.defaultTaskCount`; and
3. a value given as a string for an INT or FLOAT Job Parameter when a Job is created, and a number given as a string
   inside a value for a LIST[INT], LIST[FLOAT] or LIST[LIST[INT]] Job Parameter
   (see [§2.13](#213-joblistintparameterdefinition-extension-expr),
   [§2.14](#214-joblistfloatparameterdefinition-extension-expr) and
   [§2.16](#216-joblistlistintparameterdefinition-extension-expr)).

The white space is not part of the value. Where the text a number was written with is kept, as the
[Expression Language](2026-02-Expression-Language#134-float-value-pass-through) does for a float value, the white space
is not part of that text.

*White space* is any character with the Unicode `White_Space` property: U+0009 to U+000D, U+0020, U+0085, U+00A0,
U+1680, U+2000 to U+200A, U+2028, U+2029, U+202F, U+205F and U+3000. No other character is white space here. In
particular U+001C to U+001F, U+200B (zero width space) and U+FEFF (zero width no-break space) are not.

## 8. `<SimpleAction>`

This object is only available in the extension `FEATURE_BUNDLE_1`

A SimpleAction is syntactic sugar for a command with arguments that is run on
a host. It removes some commonly needed boilerplate for the given script
interpreter. In contrast to `<Action>`, `<SimpleAction>` removes the need for the
`command` property, implicitly prefixes the `args` property, and removes the
need for the `embeddedFiles` property.

The behaviour of the syntactic sugar depends on the key it was specified with:

* *bash* - Implicitly creates a Bash embedded file,
`command` becomes `bash`, and `args` get prefixed with the implicitly
generated file. The file extension is `.sh`. `bash` (all lowercase) is
expected to be available in the runtime environment.

  ```yaml
  steps:
    - name: BashStep
      bash:
        let:  # optional, requires EXPR extension
          - output_file = Param.OutputDir / Param.Pattern.with_number(Task.Param.Frame)
        args: ["--additional-argument"] # optional
        script: |
          # bash code here
          echo Output: {{repr_sh(output_file)}}

  ### syntax sugar equivalent to:

  steps:
    - name: BashStep
      script:
        let:
          - output_file = Param.OutputDir / Param.Pattern.with_number(Task.Param.Frame)
        actions:
          onRun:
            command: bash
            args: ["{{Task.File.<implicitly generated file>}}", "--additional-argument"]
        embeddedFiles:
          - name: <implicitly generated file>
            filename: "<implicitly generated file>.sh"
            type: TEXT
            data: |
              # bash code here
              echo Output: {{repr_sh(output_file)}}
  ```

* *cmd* - Implicitly creates a Batch embedded file,
`command` becomes `cmd`, and `args` get prefixed with
`["/C", "{{Task.File.<implicitly generated file>}}"]`. The file extension is
`.bat`. `cmd` (all lowercase) is expected to be available in the runtime environment.

  ```yaml
  steps:
    - name: CmdStep
      cmd:
        args: ["--additional-argument"] # optional
        script: |
          # batch code here

  ### syntax sugar equivalent to:

  steps:
    - name: CmdStep
      actions:
          onRun:
              command: cmd
              args: ["/C", "{{Task.File.<implicitly generated file>}}", "--additional-argument"]
      embeddedFiles:
          - name: <implicitly generated file>
            filename: "<implicitly generated file>.bat"
            type: TEXT
            data:
                # batch code here
  ```

* *node* - Implicitly creates a JavaScript embedded file,
`command` becomes `node`, and `args` get prefixed with the implicitly
generated file. The file extension is `.js`. `node` (all lowercase) is
expected to be available in the runtime environment.

  ```yaml
  steps:
    - name: NodeStep
      node:
        args: ["--additional-argument"] # optional
        script: |
            # JavaScript code here

  ### syntax sugar equivalent to:

  steps:
    - name: NodeStep
      actions:
          onRun:
              command: node
              args: ["{{Task.File.<implicitly generated file>}}", "--additional-argument"]
      embeddedFiles:
          - name: <implicitly generated file>
            filename: "<implicitly generated file>.js"
            type: TEXT
            data:
                # JavaScript code here
  ```

* *powershell* - Implicitly creates a Powershell embedded file,
`command` becomes `powershell`, and `args` get prefixed with
`["-File", "{{Task.File.<implicitly generated file>}}"]`.
The file extension is `.ps1`. `powershell` (all lowercase) is
expected to be available in the runtime environment.

  ```yaml
  steps:
    - name: PowershellStep
      powershell:
        args: ["--additional-argument"] # optional
        script: |
            # powershell code here

  ### syntax sugar equivalent to:

  steps:
    - name: PowershellStep
      actions:
          onRun:
              command: powershell
              args: ["-File", "{{Task.File.<implicitly generated file>}}", "--additional-argument"]
      embeddedFiles:
          - name: <implicitly generated file>
            filename: "<implicitly generated file>.ps1"
            type: TEXT
            data:
                # Powershell code here
  ```

* *python* - Implicitly creates a Python embedded file,
`command` becomes `python`, and `args` get prefixed with the implicitly
generated file. The file extension is `.py`. `python` (all lowercase) is
expected to be available in the runtime environment.

  ```yaml
  steps:
    - name: PythonStep
      python:
        args: ["--additional-argument"] # optional
            # python code here

  ### syntax sugar equivalent to:

  steps:
    - name: PythonStep
      actions:
          onRun:
              command: python
              args: ["{{Task.File.<implicitly generated file>}}", "--additional-argument"]
      embeddedFiles:
          - name: <implicitly generated file>
            filename: "<implicitly generated file>.py"
            type: TEXT
            data:
                # python code here
  ```

A `<SimpleAction>` is the object:

```yaml
let: <LetBindings> # @optional @extension EXPR
script: <DataString> # @fmtstring[host]
args: [ <ArgString>, ... ] # @optional @fmtstring[host]
timeout: <posinteger> | <posintstring> # @optional @fmtstring
cancelation: <CancelationMethod> # @optional
```

Where `<posintstring>` is a string whose value is the string representation of a
positive integer value in base-10, with optional white space around it
(see [Numeric strings](#75-numeric-strings)), and:

1. *let* — An ordered list of expression bindings evaluated once per task. Bound names are available in *script*
   and *args* fields, and can reference `Task.Param.*` values. See: [&lt;LetBindings&gt;](#36-letbindings).
   Requires both the `FEATURE_BUNDLE_1` and `EXPR` extensions.
2. *script* — The [Data String](#data-string) that will be written to the script
   exactly as it appears.
3. *args* — An array of [Format Strings](#73-format-strings) that will be passed
   as arguments to the **command** when the command is run on the host.
4. *timeout* — The positive number of seconds that the command is given to
   successfully run to completion. A command that does not return before the
   timeout is canceled and is treated as a failed run.

   The default timeout, if not provided, depends on the action:

   | Action container | Action | Default |
   | --- | --- | --- |
   | `<StepActions>` | `onRun` | *no timeout* |
   | `<EnvironmentActions>` | `onEnter` | *no timeout* |
   | `<EnvironmentActions>` | `onWrapEnvEnter` <sup>2</sup> | *no timeout* |
   | `<EnvironmentActions>` | `onWrapTaskRun` <sup>2</sup> | *no timeout* |
   | `<EnvironmentActions>` | `onWrapEnvExit` <sup>2</sup> | 300 seconds (five minutes) <sup>1</sup> |
   | `<EnvironmentActions>` | `onExit` | 300 seconds (five minutes) <sup>1</sup> |

   <sup>1</sup> Environment exit actions are treated specially. Job schedulers
   may provide the ability to cancel jobs, steps, and tasks. A reasonable
   default expectation should be that OpenJobDescription sessions are able to
   end and cleanup within a bound duration of time.

   <sup>2</sup> Available with the `WRAP_ACTIONS` extension. The wrap hook's `timeout` field bounds
   the wrap script on the host (infrastructure-side); the `WrappedAction.Timeout` template variable
   carries the wrapped action's timeout (workload-side). When the wrapped action specified no
   timeout, the variable is `null` (it is typed `int?`). See the *timeout* description in
   [&lt;Action&gt;](#5-action).

5. *cancelation* — If defined, provides details regarding how this action should
   be canceled. If not provided, then it is treated as though provided with `<CancelationMethodTerminate>`.

The host uses the return code of the interpreter run to determine success or
failure of the SimpleAction. A zero exit code indicates success, and any
non-zero exit code indicates failure. A timeout also indicates failure.

## 9. Additional Information

* The Cc unicode character category is all [C0](https://www.unicode.org/charts/PDF/U0000.pdf)
  and [C1](https://www.unicode.org/charts/PDF/U0080.pdf) characters.
  * This is a category of 65 non-printable characters such as NUL, BEL, Backspace, DEL,
     tab, newline, carriage return, and form feed.

## 10. License

Copyright ©2023 Amazon.com Inc. or Affiliates ("Amazon").

This Agreement sets forth the terms under which Amazon is making the Open Job Description
Specification ("the Specification") available to you.

### 10.1. Copyrights

This Specification is licensed under [CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/deed.en).

### 10.2. Patents

Subject to the terms and conditions of this Agreement, Amazon hereby grants to
you a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable
(except as stated in this section) patent license to make, have made, use, offer
to sell, sell, import, and otherwise transfer implementations of the
Specification that implement and are compliant with all relevant portions of the
Specification ("Compliant Implementations"). Notwithstanding the foregoing, no
patent license is granted to any technologies that may be necessary to make or
use any product or portion thereof that complies with the Specification but are
not themselves expressly set forth in the Specification.

If you institute patent litigation against any entity (including a cross-claim
or counterclaim in a lawsuit) alleging that Compliant Implementations of the
Specification constitute direct or contributory patent infringement, then any
patent licenses granted to You under this Agreement shall terminate as of the
date such litigation is filed.

### 10.3. Additional Information

For more info see the [LICENSE file].

[LICENSE FILE]: https://github.com/OpenJobDescription/openjd-specifications/blob/mainline/LICENSE
