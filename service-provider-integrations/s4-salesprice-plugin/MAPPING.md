# S/4 Sales Price Plugin — Field Mapping Table

## Scenario 1: Outbound — CCS Request → S/4 Request

| # | CCS Source Field                                                                        | S/4 Target Field | Type | Notes |
|---|-----------------------------------------------------------------------------------------|-----------------|------|-------|
| 1 | `data.salesArea.salesOrganization`                                                      | `SalesOrganization` | Direct map | Mandatory in S/4 |
| 2 | `data.salesArea.distributionChannel`                                                    | `DistributionChannel` | Direct map | Mandatory in S/4 |
| 3 | `data.salesArea.division`                                                               | `Division` | Direct map | Mandatory in S/4 |
| 4 | `data.currency.code`                                                                    | `TransactionCurrency` | Direct map | Optional in S/4; ISO currency code |
| 5 | _(hardcoded `""`)_                                                                      | `SalesDocumentType` | Hardcoded | Mandatory-but-defaultable; S/4 applies system default when empty |
| 6 | _(hardcoded `""`)_                                                                      | `SalesDocumentItemCategory` | Hardcoded | Mandatory-but-defaultable; S/4 applies system default when empty |
| 7 | _(hardcoded `""`)_                                                                      | `BillingDocumentType` | Hardcoded | Mandatory-but-defaultable; S/4 applies system default when empty |
| 8 | _(hardcoded `""`)_                                                                      | `LanguageISOCode` | Hardcoded | Optional; S/4 uses system language when empty |
| 9 | _(hardcoded `"SAP_DFLT"`)_                                                              | `SalesPriceRetrieveUseCase` | Hardcoded | Optional; triggers S/4 defaults for fields 5–7 when empty |
| 10 | _(hardcoded `true`)_                                                                    | `PriceElementsAreRequested` | Hardcoded | Mandatory; always `true` to retrieve full price breakdown |
| 11 | _(data.pricingAt.split('T')[0],otherwise JS: `new Date().toISOString().split('T')[0]`)_ | `PricingDate` | Computed | Mandatory in S/4; CCS request has no pricing date field — **potential gap if past/future date needed** |
| 12 | `data.products[].id`                                                                    | `_Product[].Product` | Direct map (array) | Mandatory in S/4 |
| 13 | _(hardcoded `""`)_                                                                      | `_Product[].ProductUsedByCustomer` | Hardcoded | Optional; not present in CCS request |
| 14 | `data.products[].quantity`                                                              | `_Product[].RequestedQuantity` | **Known business gap** | CCS `ProductResource` has no quantity field; S/4 defaults to qty 1; scale/tiered pricing unsupported |
| 15 | `data.products[].unit.code`                                                             | `_Product[].RequestedQuantityISOUnit` | **Known business gap** | Omitted because `RequestedQuantity` is omitted (only mandatory when quantity is set) |
| 16 | **NOT MAPPED**                                                                          | `_Product[].RequestedQuantitySAPUnit` | **Known business gap** | Omitted because `RequestedQuantity` is omitted |
| 17 | `data.businessPartner.customerNumber`                                                   | `_SoldToParty[].SoldToParty` | Direct map (wrapped in array) | Mandatory in S/4 |

---

## Scenario 2: Inbound — S/4 Response → CCS Response

| # | S/4 Source Field | CCS Target Field                                                      | Type | Notes |
|---|-----------------|-----------------------------------------------------------------------|------|-------|
| 1 | `value[].Product` | `data.operations[0].data.products[].id`                               | Direct map (array iteration) | Used to correlate each priced product back to CCS product id |
| 2 | `value[].NetAmount` | `data.operations[0].data.products[].price.effectiveValue`             | Direct map | Net amount after discounts, excl. tax; primary price value |
| 3 | `value[].NetAmount` | `data.operations[0].data.products[].price.scalePrices[0].price`       | Direct map (duplicated) | Single scale price entry; always minQuantity=1 — **consequence of known business gap on RequestedQuantity** |
| 4 | _(hardcoded `1`)_ | `data.operations[0].data.products[].price.scalePrices[0].minQuantity` | Hardcoded | Fixed to 1 because S/4 was always called with qty 1 |
| 5 | _(hardcoded `"productPrice"`)_ | `data.operations[0].operationId`                                      | Hardcoded | Required CCS operation discriminator |
| 6 | `value[]._SystemMessages[]` | _(not mapped)_                                                        | **Known limitation** | EXPRESSION mapper cannot conditionally emit a second operation; S/4 messages are discarded — see Gap G7 |

### S/4 response fields available but not mapped to CCS

| S/4 Field | Reason not mapped |
|-----------|------------------|
| `value[].NetPriceAmount` | Redundant with `NetAmount` for the qty-1 case; `NetAmount` preferred as it reflects all discounts |
| `value[].NetPriceQuantity` | Quantity context; not surfaced in CCS price model |
| `value[].NetPriceQuantityUnitISOCode` / `SAPCode` | Unit context; not surfaced in CCS price model |
| `value[].TaxAmount` | CCS price model does not include tax in `determinePrice` response |
| `value[].PricingHasError` | Not mapped to a structured CCS field; pricing errors surface via `_SystemMessages` instead |
| `value[].SalesOrganization` / `DistributionChannel` / `Division` | Echo of request context; not needed in CCS response |
| `value[].SoldToParty` | Echo of request context; not needed in CCS response |
| `value[].PricingDate` / `TransactionCurrency` / `LanguageISOCode` | Echo of request context; not needed in CCS response |
| `value[].RequestedQuantity` / `RequestedQuantityISOUnit` / `RequestedQuantitySAPUnit` | Echo of request context; not needed in CCS response |
| `value[]._SalesPriceElements[]` | Full pricing procedure breakdown (gross, discounts, tax lines, etc.); no corresponding structure in CCS `ProductPriceInput` — **potential gap if CCS adds a price breakdown model** |
| `value[].ProductUsedByCustomer` | Customer material number; not used in CCS response |

---

## Gap Summary

| # | Gap                                                                                     | Direction | Impact | Resolution |
|---|-----------------------------------------------------------------------------------------|-----------|--------|------------|
| G1 | ~~`PricingDate` not in CCS request~~                                                        | CCS→S/4 | Plugin always uses today's date; past/future pricing not possible | CCS SPI enhancement: add optional `pricingDate` to `ProductPriceRequestResource` |
| G2 | ~~`RequestedQuantity` not in CCS request~~                                                  | CCS→S/4 | S/4 always prices for qty 1; scale/tiered pricing returns wrong price for other quantities | CCS SPI enhancement: add `quantity` to `ProductResource` |
| G3 | `SalesDocumentType` / `SalesDocumentItemCategory` / `BillingDocumentType` not in CCS request | CCS→S/4 | S/4 uses system defaults; may not match the actual sales document context | CCS SPI enhancement: add optional document type fields, or expose `SalesPriceRetrieveUseCase` |
| G4 | ~~`_SalesPriceElements` not mapped to CCS response~~                                        | S/4→CCS | Full pricing breakdown (conditions, discounts, tax lines) is discarded | CCS SPI enhancement: add a price elements / breakdown structure to `ProductPriceInput` |
| G5 | `TaxAmount` not mapped to CCS response                                                  | S/4→CCS | Tax is excluded from the determined price; downstream tax calculation may double-count | Accepted: tax is handled separately via the `simulateTax` extension point |
| G6 | `PricingHasError` not mapped to CCS response                                            | S/4→CCS | Pricing error flag is lost unless S/4 also populates `_SystemMessages` | Mitigation: S/4 typically populates `_SystemMessages` when `PricingHasError=true`; add explicit check if needed |
| G7 | `_SystemMessages` not forwarded to CCS                                                  | S/4→CCS | S/4 warning/error messages are silently discarded | Inbound mapper uses EXPRESSION type (platform constraint); upgrade to FUNCTION-based inbound mapper once platform supports multiple functions per plugin |
