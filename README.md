### Overview

This describes a product that lets anonymous users create, send, and submit forms. It focuses on requirements, data models, and business logic. It ignores delivery timelines, pricing, infrastructure, monitoring, e.t.c.

### Product Requirements

1. Users should be able to anonymously create forms that support:
    - Dropdowns
    - File uploads up to 2 megabytes.
    - Text fields
    - Email fields
    - Yes or No radio buttons

2. Users can create rules on a field that must evaluate to true before it is displayed. This rules are based on the values of other fields(except file uploads). 

3. Users who create forms can have the links to their forms mailed to respondents. They can also copy the link and share it manually. 

4. Anyone with the link to a form should be able to fill and submit it.

### Data Models

**Form**

- id
- title
- description
- reference       (For use in web links)
- creationDate
- fields: Field[] 

**Field**

- id:
- type: Type [TEXT, EMAIL, DROPDOWN, BOOLEAN, FILE] 
- question
- isRequired: boolean
- isConditional: boolean
- conditions: Condition[]

**Condition**

- id:
- operator: Operator [AND, OR]
- rules: Rule[]

**Rule**

- id
- target: Field 
- expectedValue: string

**FormResponse**

- id
- form: Form
- submissionDate:
- fieldResponses: FieldResponse[]

**FieldResponse**

- id
- formResponse: FormResponse
- field: Field
- value: string // For files, this is a reference from an object store 

### Workflows

**Create Form**

- Generate a unique reference that is used in the link for accessing the form. Set it as the value of `form.reference`. This useful to avoid exposing internal ID of the form.

- When `field.isConditional` is true
    - Check that `field.isRequired` is false.
    - Check that `field.conditions` has at least one element.
    - Check that `rule.target.type` is never `FILE`.

*Conditional Fields:*

- A `Rule` represents the value that must be present on another field(the target) before the field containing the rule can be displayed. 

- A `Condition` has a collection of `Rule`s  that are combined by it's `Operator`(AND or OR) to produce a boolean. A field is only displayed if all it's `Condition`s evaluates to true. 

- When a `Condition` has only one rule, then the field is only displayed if the current value of the target field is equal to `rule.expectedValue`.

For example: A field for `Employer Address` can have the following condition(s):

```json
[
    {
      "operator": "AND",
      "rules": [
        {"target":"isEmployed", "expectedValue": true}
        {"target": "country", "expectedValue": "Argentina"}
      ]
    },
    {
      "operator": "OR",
      "rules": [
        {"target":"title", "expectedValue": "Software Engineer"}
        {"target": "city", "expectedValue": "Buenos Aires"}
      ]
    }
]
```

This means the employer address field is only displayed if the following evaluates to true: ((`isEmployed = true` AND `country = Argentina`) AND (`title = Software Engineer` OR `city = Buenos Aires`)).

**Send Form**

- Get `form.reference` and combine with it base URL of current environment to create a link.
- This link can be sent in a mail or displayed.

**Accept Response**

- Where `field.type` is `FILE`, upload the bytes to object storage and set the ID received as `fieldResponse.value`. Where this upload fails, the form submission is not successful and has to be retried.

- Where `field.isRequired` is true, ensure that `fieldResponse.value` is not empty or null.

### Possible Improvements

- Form sections: This would let form creators separate forms into logical sections. Only one section would be displayed on a single page.

- Responses Dashboard: This would visualize the number and nature of responses to a form. Form creators can see the number answers to questions, form submission times e.t.c.

- Form Settings: This lets form creators decide how respondents can interact with a form. For example, form creators can set when forms expire.
