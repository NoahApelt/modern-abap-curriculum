# Adding Behavior

In the [previous unit](./transactional_app.md) the first functionality was added to the business object.
In particular, it was defined which [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) are
available. In this unit the functionality of the business object is extended by adding additional behaviours.  
[Validations](https://help.sap.com/docs/btp/sap-abap-restful-application-programming-model/validations) are
implemented to check the data entered by the user. For example consider the email address of a user. A
validation can be used to check if a valid email address was entered.
[Determinations](https://help.sap.com/docs/btp/sap-abap-restful-application-programming-model/determinations)
are used to change the business object data based on trigger conditions. Finally
[actions](https://help.sap.com/docs/btp/sap-abap-restful-application-programming-model/actions) are added
to allow the user to trigger an operation on the business object.

## Extending the Data Model

Before adding additional behavior to the business object first the data model of the business object is extended.
There are two reasons for that. First, a status field will be used to show how determinations and actions are
implemented. Second, virtual elements are used extend the business object without changing the underlying data model.

### Adding a Status Field

The first step is to add a status field to the business object. This field will store the processing state of
a rating. First, a rating is in the status _New_. Once the customer provided a rating, i.e. rating value and an
optional review, the status is _Customer Review_. If the review was checked by a product manager the status
is changed to _Completed_.

#### Exercise 1

Extend the data model of the review business object. To do this perform the following steps:

1. Create a domain `ZD_STATUS` in the package `Z_RATING_DB`. The domain should allow the following values:
   | Fixed Values | Description |
   | -----------: | --------------- |
   | 0 | undefined |
   | 10 | New |
   | 20 | Customer Review |
   | 30 | Completed |
1. Create a data element `ZE_STATUS` in the package `Z_RATING_DB`. The data element should use the domain `ZD_STATUS`.
1. Add a `status` field to the `ZRATING` table after the `review` field. The field should by of type `ZE_STATUS`
1. Add the `status` field to the `Z_I_Rating` view after the `Review` field and rename it to `Status`.
1. Add the `Status` field to the `Z_C_Rating_M` view and the `Z_C_Rating_M` metadata extension. The `Status` should be
   displayed in the ratings list on a product's object page.

If you completed exercise 1 the resulting object page for a product should look similar to the following
screenshot:

![Object Page with Status Field](./imgs/adding_behavior/object_page_with_status.png)

Note that the status field contains the value `0` for all existing data in the app. In the next steps functionality
to change this value will be added. Furthermore, the screenshot above shows the values in the `Customer Rating` column
as a rating indicator. How to achieve this was discussed in a [previous unit](./ro_list_report.md#beautifying-the-app). The following
snippet shows the necessary additions to the metadata extension `Z_C_Rating_M`.

```abap
...
  @UI:{
    lineItem: [{
        position: 50,
        type: #AS_DATAPOINT
    }],
    dataPoint: {
      qualifier: 'Rating',
      targetValue: 5,
      visualization: #RATING
    }
  }
  Rating;
...
```

## Adding Determinations

In this step functionality to automatically update the value of the status field is added.
In SAP RAP
[determinations](https://help.sap.com/docs/btp/sap-abap-restful-application-programming-model/determinations)
are used to change the value of fields based on some trigger conditions.
The [ABAP keyword documentation](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/index.htm?file=abenbdl_determinations.htm)
shows that determinations can be execute `on save` and `on modify`. According to the documentations
`on modify` specifies that:

> the determination is executed immediately after data changes take place in the transactional buffer so that the result is available
> during the transaction.

In contrast to that `on save` specifies that:

> the determination is executed during the save sequence at the end of an transaction, when change in
> the transactional buffer are persistent on the database.

In both cases the trigger condition can be one of the CRUD operations `create`, `update` and `delete`. Furthermore, the changes
of a field value can also be used as a trigger condition.

The following snippet shows two determination as part of the behavior definition of `Z_I_Rating`.
The determination `setStatusNew` is executed `on modify` whenever a new `Rating` is created. The idea is to
use this determination to set the initial status value of a `Rating`.
The determination `setStatusCustomerFeedback` is executed `on modify` whenever the value of the field `Rating` changes.
In this case the idea is to set the status field to the value `20` whenever a rating value is provided.

```abap
define behavior for Z_I_Rating alias Rating
persistent table zrating
lock dependent by _Product
authorization dependent by _Product
{
  update;
  delete;
  field ( readonly ) Product;
  association _Product;

  field ( readonly, numbering : managed ) RatingUUID;
  field ( readonly ) LastChangedAt, LastChangedBy, CreatedAt, CreatedBy;

  determination setStatusNew on modify {create; }
  determination setStatusCustomerFeedback on modify { field Rating; }

  mapping for zrating corresponding
    {
      RatingUUID    = rating_uuid;
      LastChangedAt = last_changed_at;
      LastChangedBy = last_changed_by;
      CreatedAt     = created_at;
      CreatedBy     = created_by;
    }
 }
```

Adding the determination to the behavior definition is only the first step.
The implementation of these determinitions is still missing. The little light bulb icon
in the editor shows, that a quick fix is available. By clicking on the light bulb icon
or pressing `<ctrl> + 1` this quick fix can be applied.

Executing the quick fix for `determination setStatusNew...` creates a method `setstatusnew` in the local
class `lhc_rating` inside the class `Z_BP_I_Product`, i.e. the global class implementing the behaviors for
`Z_I_Product`. Before implementing the method I would recommend renaming it to `set_status_new` to
conform with common ABAP naming conventions. Below is the resulting method declaration.

```abap
...
METHODS set_status_new FOR DETERMINE ON MODIFY
      IMPORTING keys FOR Rating~setStatusNew.
...
```

The implementation of the method `set_status_new` is shown in the following listing:

```abap
METHOD set_status_new.
    DATA ratings_for_update TYPE TABLE FOR UPDATE Z_I_Rating.

    READ ENTITIES OF Z_I_Product IN LOCAL MODE
     ENTITY Rating
       FIELDS ( Status )
       WITH CORRESPONDING #( keys )
     RESULT DATA(ratings).

    DELETE ratings WHERE Status IS NOT INITIAL.
    CHECK ratings IS NOT INITIAL.

    LOOP AT ratings ASSIGNING FIELD-SYMBOL(<rating>).
      APPEND VALUE #( %tky = <rating>-%tky
                      status = rating_status-new )
             TO ratings_for_update.
    ENDLOOP.

    MODIFY ENTITIES OF Z_I_Product IN LOCAL MODE
      ENTITY Rating
        UPDATE FIELDS ( Status )
        WITH ratings_for_update
      REPORTED DATA(update_reported).

    reported = CORRESPONDING #( DEEP update_reported ).

  ENDMETHOD.
```

It should obvious, that the implementation of `set_status_new` is not plain ABAP. Instead it contains
[Entity Manipulation Language (EML)](https://help.sap.com/docs/btp/sap-abap-restful-application-programming-model/entity-manipulation-language-eml)
statements. EML is an extension to the ABAP language enabling access to RAP business object. A complete introduction to
EML is beyond the scope of this curriculum. Instead, only the EML elements necessary to implement the requirement of the
example app are explained. The implementation of the `set_status_new` method performs the following operations:

1. The variable `ratings_for_update` is defined as a `TABLE FOR UPDATE` with the structure `Z_I_Rating`.
   A `TABLE FOR UPDATE` is a table type with a special structure for working with ABAP RAP objects. The main features off
   these table types are:
   - Enabled for of data in RAP
   - Containing [special components](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/index.htm?file=abapderived_types_comp.htm) like
     %tky. Details on these special components can be found in the
     [ABAP documentation](https://help.sap.com/doc/abapdocu_cp_index_htm/CLOUD/en-US/index.htm?file=abapderived_types_comp.htm)
     or in the [EML cheat sheet](https://github.com/SAP-samples/abap-cheat-sheets/blob/main/08_EML_ABAP_for_RAP.md).
1. The `READ ENTITIES` EML statement is used to read the value of the `Status` field for all elements in the importing parameter `keys`.
   This parameter contains the primary key of all created entites. Note, the design of ABAP RAP and EML focuses on mass data. Therefore all
   operations are always operations on multiple objects. Working with just one object is a special case. The result of the
   `READ ENTITIES` EML statement is stored in the variable `ratings`.
1. All ratings that already have a `Status` value are deleted from `ratings`. This is done to ensure no values are overwritten with the
   default value. The `CHECK` statement ensure that the processing only continues if the are still entries in the variable `ratings`.
1. In the `LOOP` the a new entry is added to `ratings_for_update`. This entry contains the values of the special component `%tky` and
   the default value `rating_status-new` for the status. Setting the default value uses the following constant defined in the header of
   `lhc_rating`.
   ```abap
   ...
   CONSTANTS:
      BEGIN OF rating_status,
        new               TYPE i VALUE 10,
        customer_feedback TYPE i VALUE 20,
        completed         TYPE i VALUE 30,
      END OF rating_status.
   ...
   ```
1. The `MODIFY ENTITIES` statement updated the transactional buffer for all changed ratings, i.e. the entries in
   `ratings_for_update`. Any error messages occuring during the modification of the entites will be returned in the
   `REPORTED` parameter. In this example these errors are returned form the method.

---

[< Previous Chapter](./transactional_app.md) | [Next Chapter >](./next_steps.md) | [Overview 🏠](../README.md)