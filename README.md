# WooFlow

A lightweight WordPress plugin that adds an **Export** button to each row of the WooCommerce Orders table. Clicking it appends that order's data directly to a Google Sheet — using the official Google Sheets API via a service account, with **no SheetDB, Zapier, or Make.com** in the middle.

---

## Credits

**Developed by:**
- Mohammod Hamed Hasan

---

## Features

- ✅ One-click export per order row (works with both legacy and HPOS order tables)
- ✅ Writes directly to Google Sheets using a service account (free, no third-party API limits)
- ✅ Exports: Serial Number, Courier ID, Order ID, Date, Customer Name, Phone, Items, Item Quantity, Total Quantity, Amount (BDT), Delivery Charge, ADC, Address, Order Notes, and Source
- ✅ Permanent "Success" status indicator once an order is exported
- ✅ Built-in Debug Mode to troubleshoot failed exports without digging through server logs
- ✅ Pure PHP — no Composer dependencies, no external libraries
- ✅ AJAX-powered export without page refresh

---

## Requirements

- WordPress with WooCommerce active
- A Google account (free) to create a service account
- PHP `openssl` extension enabled (standard on virtually all WordPress hosts)

---

## Installation

1. Download `wooflow.php` from this repository.
2. Upload it to `wp-content/plugins/wooflow/` on your WordPress site (via FTP, hosting file manager, or WP Admin → Plugins → Add New → Upload Plugin).
3. Activate **WooFlow** from **WP Admin → Plugins**.

---

## Setup Guide

You need a **Google Cloud service account** — a free, machine-only credential that's allowed to write to a specific sheet you share with it.

### Step 1: Create a Google Cloud Project

1. Navigate to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown (top-left corner) → **New Project**
3. Name your project (e.g., `woo-flow-export`) → Click **Create**
4. Select your newly created project from the dropdown menu

### Step 2: Enable Google Sheets API

1. Use the top search bar → search for `Google Sheets API`
2. Click on the result → Click **Enable**

### Step 3: Create a Service Account

1. Search for `Service Accounts` (or navigate to **IAM & Admin → Service Accounts** in the sidebar)
2. Click **+ Create Service Account**
3. Name it (e.g., `sheet-writer`) → Click **Create and Continue**
4. Skip assigning roles and user access steps → Click **Done**

### Step 4: Generate JSON Key

1. Click on the service account's email address in the list
2. Go to the **Keys** tab → Click **Add Key** → **Create new key**
3. Choose **JSON** format → Click **Create**
4. A `.json` file will download automatically — **keep this file secure and private** as it contains sensitive credentials

### Step 5: Share Your Google Sheet

1. Open the downloaded JSON file and copy the `client_email` value
2. Open your target Google Sheet → Click **Share** (top-right corner)
3. Paste the service account email, set permission to **Editor** → Click **Send**

### Step 6: Locate Your Sheet ID

Find the Sheet ID in your Google Sheet URL:

```
https://docs.google.com/spreadsheets/d/SHEET_ID_HERE/edit
```

Example: `1A2b3C4d5E6f7G8h9I0j` (the long string between `/d/` and `/edit`)

### Step 7: Configure the Plugin

Navigate to **WP Admin → Settings → WooFlow** and fill in:

| Field | Description |
|-------|-------------|
| **Google Sheet ID** | The Sheet ID from Step 6 |
| **Sheet/Tab Name** | The tab name at the bottom of your sheet (e.g., `Sheet1`) |
| **Service Account JSON Key** | Paste the entire contents of your downloaded `.json` file |
| **Enable Debug Mode** | Optional: Check to show API errors on screen |

Click **Save Changes** to store your configuration.

### Step 8: Set Up Your Sheet Header Row

Row 1 of your Google Sheet must contain these columns in exactly this order (A through O):

| Column | Header Text | Description |
|--------|-------------|-------------|
| A | SL | Auto-generated serial number via `=ROW()-1` |
| B | Courier ID | Pathao or Steadfast consignment ID |
| C | Order ID | WooCommerce order number |
| D | Date | Order creation date (YYYY-MM-DD) |
| E | Customer Name | Billing first and last name |
| F | Phone | Billing phone number |
| G | Items | Product names with quantities |
| H | Item Quantity | **Left empty** - filled by Google Apps Script |
| I | Total Quantity | **Left empty** - filled by Google Apps Script |
| J | Amount (BDT) | Order total amount |
| K | Delivery Charge | Shipping total |
| L | ADC | **Left empty** - for manual entry or other use |
| M | Address | Billing address (street, city) |
| N | Order Notes | Customer/order notes |
| O | Source | Always "WooCommerce" |

> **Important:** Columns `Item Quantity` (H), `Total Quantity` (I), and `ADC` (L) are left empty by this plugin. They can be filled automatically by your Google Apps Script if needed.

---

## Usage Instructions

1. Go to **WooCommerce → Orders** in your WordPress admin
2. Each order row now displays an **Export** button (light green)
3. Click the **Export** button for any order
4. The AJAX request sends data to Google Sheets without refreshing the page
5. On success, the button turns **dark green with "Success"** text and is permanently disabled
6. On failure, an error alert appears explaining the issue

### Button States

| State | Appearance | Meaning |
|-------|-----------|---------|
| **Default** | Light green with "Export" | Ready to export |
| **Processing** | Grayed out with "Sending..." | Export in progress |
| **Success** | Dark green with "Success" | Order successfully exported (permanent) |

---

## Troubleshooting Guide

Enable **Debug Mode** in the plugin settings page. On your next export attempt, instead of redirecting back to the orders list, the plugin will display:

- The exact row of data it attempted to send
- Google's complete HTTP response (success details or specific error messages)

### Common Issues and Solutions

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| "No service account JSON key saved" | JSON wasn't saved correctly | Re-paste the entire JSON content in settings and save again |
| HTTP 403 / PERMISSION_DENIED | Sheet not shared with service account | Share the sheet with the `client_email` as **Editor** |
| HTTP 400 / Unable to parse range | Wrong Sheet ID or tab name | Double-check Sheet ID and tab name in settings |
| Courier ID column is blank | Courier plugin uses different meta key | See [Customization](#customization) section below |
| Network timeout | Slow internet connection | Check your server's internet connectivity to Google APIs |
| Export button remains "Export" after success | Database meta not saved | Check if `_woosheet_is_exported` meta is being saved to order |

---

## Customization

### Changing Courier ID Meta Keys

The plugin checks these order meta keys in priority order:

```php
// Pathao
$pathao_id = $order->get_meta('ptc_consignment_id');
if (empty($pathao_id)) {
    $pathao_id = get_post_meta($order_id, 'ptc_consignment_id', true);
}

// Steadfast
$steadfast_id = $order->get_meta('steadfast_consignment_id');
if (empty($steadfast_id)) {
    $steadfast_id = get_post_meta($order_id, 'steadfast_consignment_id', true);
}
if (empty($steadfast_id)) {
    $steadfast_id = $order->get_meta('_steadfast_consignment_id');
}
if (empty($steadfast_id)) {
    $steadfast_id = get_post_meta($order_id, '_steadfast_consignment_id', true);
}
```

If your courier plugin stores the ID under a different meta key:

1. Open `wooflow.php` in a code editor
2. Locate the `woosheet_process_export()` function (note: function name remains from original)
3. Update the meta key names to match your courier plugin's storage
4. Find the correct meta key by:
   - Using a plugin like "Custom Fields" debug view in WP Admin
   - Checking your courier plugin's source code or documentation
   - Using `var_dump($order->get_meta_keys())` temporarily

### Adding or Removing Export Fields

Edit the `$row_values` array inside `woosheet_process_export()`:

```php
$row_values = [
    0  => '=ROW()-1',              // A: SL
    1  => (string) $courier_id,    // B: Courier ID
    2  => (string) $order_id,      // C: Order ID
    // Add or remove fields here - keep array indices sequential
];
```

**Important:** 
- Keep the column order in `$row_values` matching your sheet's column order
- If you change the number of columns, update the range in `woosheet_append_row_to_sheet()` (line ~124: `!A:O` to match your column count)
- Array indices must start at 0 and increment sequentially

---

## Security Considerations

- **JSON Key Storage:** The service account JSON key is stored in the WordPress options table with `autoload` disabled, but it is **not encrypted at rest**. Anyone with database access or PHP execution on your server can read it. This is standard for this type of integration.

- **Permission Control:** The export button uses WordPress nonces and a capability check (`edit_shop_orders`), so only logged-in users with order-editing permission can trigger exports.

- **Token Security:** Access tokens obtained from Google are cached in a transient for under an hour and are never exposed to the browser.

- **AJAX Security:** All AJAX requests are validated with nonces and capability checks to prevent CSRF attacks.

- **Best Practice:** Consider using WordPress's `wp-config.php` constants for sensitive values if security is a primary concern.

---

## Frequently Asked Questions

### Q: Can I export multiple orders at once?
A: Currently, WooFlow supports one-click export per individual order. Bulk export functionality may be added in future versions.

### Q: What happens if the export fails?
A: With Debug Mode enabled, you'll see the exact error. Without debug mode, a user-friendly error message appears via JavaScript alert, and the failure is logged for investigation.

### Q: Why does the "Success" button stay even after refreshing?
A: The plugin saves a meta field `_woosheet_is_exported` to the order in the database. Once an order is successfully exported, the button permanently shows "Success".

### Q: Can I manually reset the "Success" status?
A: Yes, you can remove the meta field from the order. Navigate to the order in WP Admin, look for custom fields, and delete `_woosheet_is_exported`. Or use a plugin like "Custom Fields" for easier management.

### Q: Does this work with subscription orders?
A: Yes, WooFlow works with any order type in WooCommerce, including subscription orders (if using WooCommerce Subscriptions).

### Q: Can I use multiple sheets for different stores?
A: Yes, each WordPress installation has its own settings. You can use separate service accounts and sheets for different sites.

### Q: Will this slow down my orders page?
A: The Export button adds minimal overhead. The actual export process only runs when the button is clicked, not during page load. The AJAX implementation ensures no page refresh is needed.

### Q: What are columns H (Item Quantity), I (Total Quantity), and L (ADC) for?
A: These columns are intentionally left empty by the plugin. They are designed to be filled automatically by a Google Apps Script that can process the raw data. For example, an Apps Script can parse column G (Items) to extract individual item quantities and calculate totals.

---

## Changelog

### Version 1.0.2
- Added permanent "Success" button state with database persistence
- Implemented AJAX-based export to prevent page refresh
- Added support for both Pathao (`ptc_consignment_id`) and Steadfast courier IDs
- Improved error handling with user-friendly alerts
- Added 15-column support (A:O) with specific fields
- Added `SL` column with `=ROW()-1` formula

### Version 1.0.1
- Fixed HPOS compatibility
- Added support for multiple courier meta key checks

### Version 1.0.0
- Initial release
- One-click export to Google Sheets
- Debug mode for troubleshooting
- WordPress nonce security implementation

---

## Technical Details

### Column Mapping

| Array Index | Column | Data Source |
|-------------|--------|-------------|
| 0 | A: SL | `=ROW()-1` formula |
| 1 | B: Courier ID | Pathao or Steadfast consignment ID |
| 2 | C: Order ID | `$order->get_id()` |
| 3 | D: Date | `$order->get_date_created()->date('Y-m-d')` |
| 4 | E: Customer Name | `$order->get_billing_first_name() . ' ' . $order->get_billing_last_name()` |
| 5 | F: Phone | `$order->get_billing_phone()` |
| 6 | G: Items | Concatenated product names with quantities |
| 7 | H: Item Quantity | **Left empty** (for Apps Script) |
| 8 | I: Total Quantity | **Left empty** (for Apps Script) |
| 9 | J: Amount (BDT) | `$order->get_total()` |
| 10 | K: Delivery Charge | `$order->get_shipping_total()` |
| 11 | L: ADC | **Left empty** (for manual entry) |
| 12 | M: Address | Concatenated billing address fields |
| 13 | N: Order Notes | Concatenated order notes |
| 14 | O: Source | "WooCommerce" |

---

## Support

For issues, feature requests, or contributions:
- Create an issue in the [GitHub repository](https://github.com/yourusername/wooflow)
- Check the [troubleshooting guide](#troubleshooting-guide) above for common issues
- Ensure your WordPress, WooCommerce, and PHP versions are up to date

---

## License

MIT License — Use, modify, and redistribute freely for personal or commercial projects.

---

Built for the WooCommerce community — simplifying order exports without expensive third-party services.
