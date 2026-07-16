# S4 Retrieve Sales Price Plugin

## Description

SAP Commerce Cloud, cloud ERP edition exposes an extension point for determining product prices with an external provider. This extension point lets you integrate with SAP S/4HANA to retrieve real-time sales price information.

The S4 Retrieve Sales Price Plugin integrates the `determinePrice` extension point in SAP Commerce Cloud, cloud ERP edition with the `RetrieveSalesPrice` OData action in SAP S/4HANA. The plugin creates the "S4 Retrieve Sales Price" extension configuration and provides the required data mappings for this extension.

## Requirements

- SAP Commerce Cloud, cloud ERP edition
- SAP S/4HANA with the `API_PRODUCT_PRICING_SRV` OData service enabled
- A destination named `s4-salesprice` configured in the CCS destination service pointing to the S/4HANA system

## Download and Installation

1. Build the plugin zip from the `service-provider-integrations/s4-salesprice-plugin` directory.
   The resulting zip must have the following structure:

   ```
   s4-salesprice-plugin-<version>.zip
   ├── plugin-manifest.yaml
   └── config/
       └── ...
   ```

   On Unix/macOS:
   ```bash
   zip -r s4-salesprice-plugin-<version>.zip plugin-manifest.yaml config -x "*.DS_Store"
   ```

2. Install the plugin zip via the SAP Commerce Cloud plugin management UI or API.

## Content Structure

```
s4-salesprice-plugin/
├── plugin-manifest.yaml                          # Plugin metadata and component registry
├── config/
│   ├── synchronous_extensions/
│   │   └── s4_retrieve_sales_price.json          # Extension configuration linking to the determinePrice extension point
│   ├── functions/
│   │   ├── s4SalesPriceOutbound.json             # Outbound function metadata
│   │   └── <uuid>-spoutbound.zip                 # Outbound JS handler (CCS → S/4 request transformation)
│   └── data_mappers/
│       ├── outbound.json                          # CCS determinePrice request → S/4 RetrieveSalesPrice request
│       └── inbound.json                          # S/4 RetrieveSalesPrice response → CCS ExtensionResponse
├── MAPPING.md                                    # Full field mapping table and gap analysis
└── README.md
```

## Field Mapping

See [MAPPING.md](./MAPPING.md) for the full field mapping table covering both directions (CCS→S/4 and S/4→CCS) and a gap analysis.

### Key mapping decisions

- **Outbound**: Uses a JavaScript function handler to build the S/4 request, handling array fan-out for products and computed fields (pricing date).
- **Inbound**: Uses declarative EXPRESSION mapping to map `value[].Product` → `products[].id` and `value[].NetAmount` → `products[].price.value`.
- **`RequestedQuantity`**: Intentionally omitted — CCS `ProductResource` has no quantity field. S/4 defaults to qty 1. Scale/tiered pricing is a known limitation (see Gap G2 in MAPPING.md).
- **`_SystemMessages`**: Not forwarded to CCS due to EXPRESSION mapper constraints (see Gap G7 in MAPPING.md).

## Contributing

If you'd like to contribute code, fixes, or improvements, please create a pull request. Due to legal reasons, contributors must accept a DCO. When you create your first pull request to this project, you are automatically asked to accept the DCO. SAP uses [the standard DCO text of the Linux Foundation](https://developercertificate.org/).

## Code of Conduct

Members, contributors, and leaders pledge to make participation in our community a harassment-free experience. By participating in this project, you agree to always abide by its [Code of Conduct](https://github.com/SAP/.github/blob/main/CODE_OF_CONDUCT.md).

## License

Copyright (c) 2026 SAP SE or an SAP affiliate company. All rights reserved. This project is licensed under the Apache Software License, version 2.0 except as noted otherwise in the [LICENSE](../../LICENSE) file.
