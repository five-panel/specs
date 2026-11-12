# Issue #275: Short implementation guide

[Spec](275-give-record-form.md) · [Technical implementation plan](275-give-record-form-implementation-plan.md)

## What changes

`ModelForm.vue` already shows a label above each field. The change connects that label to the dropdown so a screen reader knows which field it is. Choosing a value must not replace the field's name with only `CLP`, `USD`, or `Activo`.

Give the existing label a stable ID and pass it to both ordinary-choice and relation dropdowns using PrimeVue's supported label reference. Currency and unit companions already use the ordinary-choice branch. The existing form layout, required markers, loading, templates, validation, and saving are reused. The only planned new file is a browser regression using the real form and PrimeVue controls.

Labels already live in model definitions (`models.fields`); saved business values already live in `custom_data.data`. The new label references exist only in the rendered page. They add no database fields or new requests.

## Concrete flow example

These are sample values for the test model described in the plan. IDs are examples; tests use IDs returned by their own fixtures. The example starts in the real record form opened from a table's **Nuevo** button, after existing loading has finished. The user has chosen the related product `Café de origen`, entered a name, and filled the amounts. Other optional fields are unset.

The complete local `formData` at this point is:

```json
{
  "product": "11111111-1111-4111-8111-111111111111",
  "name": "Café de origen · 250 g",
  "status": "active",
  "currentPrice": 12500,
  "currentUnitCost": 0
}
```

Both currency controls still display `Selecciona una opción`. Their proposed label connections are:

| Control ID | Visible label and screen-reader name | Label ID |
| --- | --- | --- |
| `record-currentPriceCurrency` | Moneda: precio actual | `record-currentPriceCurrency-label` |
| `record-currentUnitCostCurrency` | Moneda: costo unitario actual | `record-currentUnitCostCurrency-label` |

1. The user focuses price currency. Its name is **Moneda: precio actual**, even while it displays the placeholder.
2. The user chooses **CLP** with the keyboard. PrimeVue updates `formData.currentPriceCurrency` through the existing `v-model`. The dropdown displays CLP and keeps its field name.
3. The user moves to cost currency and chooses **USD**. Its separate `v-model` updates `formData.currentUnitCostCurrency`; its name remains **Moneda: costo unitario actual**.

The complete local draft is now:

```json
{
  "product": "11111111-1111-4111-8111-111111111111",
  "name": "Café de origen · 250 g",
  "status": "active",
  "currentPrice": 12500,
  "currentUnitCost": 0,
  "currentPriceCurrency": "CLP",
  "currentUnitCostCurrency": "USD"
}
```

The visible result is **12500 CLP** for price and **0 USD** for cost, with the same existing field labels. A screen reader can distinguish the two currencies. These selections update local state only; this example ends before Save, so they trigger no record write. The unchanged Save path later sends the business values through `ModelView`, `dataStore`, and GraphQL; naming attributes are never part of that data.

For a model whose visible label is **Currency: current price**, the same connection reads that English text. The fix does not translate model definitions. The browser regression also checks status and relation fields, clearing, template visibility, validation recovery, and saved/reopened records.

The [Doop preview](https://five-pane-dev-server.tail7d89bf.ts.net/c/J88FjcHO5C/proto?frame=wdTVgJirzc) illustrates the naming difference. Its readout and sample-language controls explain the preview; they are not additions to the application. The actual application change remains to be implemented.
