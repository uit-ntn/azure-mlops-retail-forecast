# Microsoft Azure Logo Setup

## Required File
Please place the **Microsoft_Azure.png** file in this directory:
```
azure-mlops-retail-forecast/static/Microsoft_Azure.png
```

## Logo Specifications
- **File name**: `Microsoft_Azure.png` (exact name required)
- **Format**: PNG with transparent background preferred
- **Recommended size**: 200px width or similar for optimal display
- **Usage**: This logo will be displayed in the sidebar navigation

## Current Configuration
The logo is configured in `layouts/partials/logo.html` with:
- Width: 60% of container
- Max height: 50px
- Auto height scaling
- Alt text: "Microsoft Azure"

## Note
Once you place the `Microsoft_Azure.png` file in the `static/` directory, the Hugo site will automatically serve it at `/Microsoft_Azure.png` and the logo will appear in the navigation. 