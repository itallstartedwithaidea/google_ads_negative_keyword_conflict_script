# Google Ads Negative Keyword Conflict Resolver

This Google Ads Script helps identify and remove conflicting negative keywords across your account, including conflicts at the ad group, campaign, and shared negative keyword list levels. The goal is to prevent scenarios where negative keywords block ads from appearing for active positive keywords.

---

## Features

- **Conflict Detection:** Identifies conflicts at three levels:
  - Ad Group-level
  - Campaign-level
  - Shared Negative Keyword Lists

- **Advanced Matching:** Enhanced to handle sophisticated conflict detection, including partial matches and different keyword match types (Broad, Phrase, Exact).

- **Dry Run Mode:** Safely preview changes without modifying your account.

- **Email Reporting:** Sends detailed conflict reports via email.

---

## Installation

Follow these steps to install and run the script:

1. **Open Google Ads Account:**
   - Sign in at [ads.google.com](https://ads.google.com).

2. **Navigate to Scripts:**
   - Tools & Settings → Bulk Actions → Scripts.

3. **Create a New Script:**
   - Click the '+' icon to add a new script.
   - Delete the default code.

4. **Paste the Script:**
   - Paste the provided `Negative Keyword Conflict Resolver` script into the editor.

5. **Configure the Script:**
   - Modify the `CONFIG` section to fit your needs:
     - Set `dryRun` to `true` to preview changes.
     - Enter your email address in `emailAddress` for reporting.

6. **Authorize and Run:**
   - Click "Authorize" to grant the script necessary permissions.
   - Click "Run" to execute.

7. **Schedule the Script (Optional):**
   - Set up a recurring schedule (e.g., weekly) for automatic conflict resolution.

---

## Configuration Options

Adjust the following options within the script to customize its behavior:

```javascript
const CONFIG = {
  dryRun: true, // Preview without actual changes
  detectPartialConflicts: true, // Advanced detection
  detailedLogging: true, // Verbose logging
  sendEmail: true, // Enable email reports
  emailAddress: 'youremail@email.com', // Recipient email
  dateRange: '', // Specify analysis period or leave empty for all-time
  maxKeywordsToProcess: 1000000 // Limit keywords processed to prevent timeout
};
```

---

## Reporting

Upon completion, the script generates a comprehensive report detailing:
- Number of conflicts detected and resolved
- Impact analysis for improved ad visibility and efficiency
- Summary of avoided false positives

Reports can be sent via email and logged directly within your Google Ads interface.

---

## Troubleshooting

- **Authorization Issues:** Ensure you have adequate permissions.
- **Timeout Errors:** Reduce the `maxKeywordsToProcess` setting.
- **Email Delivery:** Verify the `sendEmail` setting and recipient address.

---

## Support

For further assistance, reach out to your Google Ads account representative or script developer or john williams john@itallstartedwithaidea.com
