/**
 * Google Ads Negative Keyword Conflict Resolver
 * 
 * This script identifies and removes negative keywords that conflict with
 * positive keywords in your Google Ads account, preventing situations where
 * your ads wouldn't show due to contradictory keyword settings.
 * 
 * The script handles conflicts at three levels:
 * 1. Ad group level negative keywords
 * 2. Campaign level negative keywords
 * 3. Shared negative keyword lists
 * 
 * Enhanced to handle more sophisticated conflict detection including partial matches
 * and different match types.
 */

// Configuration
const CONFIG = {
  // Set to true to see what would be removed without actually removing anything
  dryRun: false,
  
  // Advanced conflict detection settings
  detectPartialConflicts: true,  // Check if negative keywords are contained within positive keywords
  
  // Logging settings
  detailedLogging: true,  // Set to true for more verbose logging
  
  // Email reporting settings
  sendEmail: true,
  emailAddress: 'youremail@email.com',
  
  // Date range for analysis (leave empty for all-time)
  dateRange: '',  // Remove this to avoid the DURING syntax issue
  
  // Performance safeguards
  maxKeywordsToProcess: 1000000 // Set a limit to prevent timeouts
};


/**
 * Main function that orchestrates the conflict detection and removal process.
 */
function main() {
  try {
    Logger.log('Starting Negative Keyword Conflict Resolution Script...');
    
    // Validate our conflict detection algorithm first
    if (!validateConflictDetection()) {
      Logger.log('WARNING: Conflict detection validation failed. Some conflicts may be incorrectly identified.');
      
      // If in dry run mode, exit to prevent incorrect actions
      if (CONFIG.dryRun) {
        Logger.log('Exiting due to validation failure in dry run mode.');
        return;
      }
      
      // Otherwise, continue with a warning
      Logger.log('Continuing despite validation failure. Use caution when reviewing results.');
    }
    
    // Track metrics for reporting
    const metrics = {
      adGroupConflictsFound: 0,
      campaignConflictsFound: 0,
      sharedListConflictsFound: 0,
      adGroupConflictsRemoved: 0,
      campaignConflictsRemoved: 0,
      sharedListConflictsRemoved: 0,
      falsePositivesAvoided: 0
    };
    
    // 1) Build a lookup of all active positive keywords.
    Logger.log('Building positive keywords map...');
    const positiveKeywordsMap = buildPositiveKeywordsMap();
    
    // 2) Check conflicts at the ad group level.
    Logger.log('Checking ad group-level conflicts...');
    metrics.adGroupConflictsRemoved = removeAdGroupNegativeConflicts(positiveKeywordsMap, metrics);
    
    // 3) Check conflicts at the campaign level.
    Logger.log('Checking campaign-level conflicts...');
    metrics.campaignConflictsRemoved = removeCampaignNegativeConflicts(positiveKeywordsMap, metrics);
    
    // 4) Check conflicts at the shared list level.
    Logger.log('Checking shared list conflicts...');
    metrics.sharedListConflictsRemoved = removeSharedListNegativeConflicts(positiveKeywordsMap, metrics);
    
    // 5) Generate report
    const report = generateReport(metrics);
    Logger.log(report);
    
    // 6) Email report if configured
    if (CONFIG.sendEmail) {
      MailApp.sendEmail({
        to: CONFIG.emailAddress,
        subject: 'Google Ads Negative Keyword Conflict Report',
        body: report
      });
    }
    
    Logger.log('Script completed successfully.');
  } catch (error) {
    Logger.log('Error executing script: ' + error);
    if (CONFIG.sendEmail) {
      MailApp.sendEmail({
        to: CONFIG.emailAddress,
        subject: 'ERROR: Google Ads Negative Keyword Conflict Script',
        body: 'The script encountered an error: ' + error
      });
    }
  }
}


/**
 * Gathers all enabled (active) positive keywords and organizes them by
 * campaign ID and ad group ID. Returns a nested map.
 * 
 * @return {Object} A map organized by campaign ID and ad group ID
 */
function buildPositiveKeywordsMap() {
  const map = {
    byId: {},         // Organized by campaign/ad group ID
    allKeywords: new Set(),  // Set of all keyword texts for quick lookup
    keywordDetails: []  // Array of all keyword details for advanced conflict checking
  };
  
  // Apply date range condition if specified and only include keywords in ENABLED ad groups and campaigns
  let condition = "Status = ENABLED AND AdGroupStatus = ENABLED AND CampaignStatus = ENABLED";
  if (CONFIG.dateRange) {
    condition += " AND metrics.impressions > 0 DURING " + CONFIG.dateRange;
  }
  
  // Use the correct pagination approach for Google Ads Scripts
  const keywordIterator = AdsApp.keywords()
    .withCondition(condition)
    .get();
  
  let count = 0;
  
  Logger.log("Processing keywords...");
  
  while (keywordIterator.hasNext()) {
    if (count >= CONFIG.maxKeywordsToProcess) {
      Logger.log(`Reached maximum keyword processing limit (${CONFIG.maxKeywordsToProcess}). Consider running in smaller batches.`);
      break;
    }
    
    const keyword = keywordIterator.next();
    const campaignId = keyword.getCampaign().getId();
    const adGroupId = keyword.getAdGroup().getId();
    const campaignName = keyword.getCampaign().getName();
    const adGroupName = keyword.getAdGroup().getName();
    const text = keyword.getText().toLowerCase();
    const matchType = keyword.getMatchType();
    
    // Add to the ID-based map
    if (!map.byId[campaignId]) {
      map.byId[campaignId] = {
        name: campaignName,
        adGroups: {}
      };
    }
    if (!map.byId[campaignId].adGroups[adGroupId]) {
      map.byId[campaignId].adGroups[adGroupId] = {
        name: adGroupName,
        keywords: {}
      };
    }
    
    // Store with match type
    map.byId[campaignId].adGroups[adGroupId].keywords[text] = matchType;
    
    // Add to the full set for quick lookup
    map.allKeywords.add(text);
    
    // Add detailed info for advanced conflict checking
    map.keywordDetails.push({
      text: text,
      matchType: matchType,
      campaignId: campaignId,
      campaignName: campaignName,
      adGroupId: adGroupId,
      adGroupName: adGroupName
    });
    
    count++;
    
    // Log progress periodically
    if (count % 5000 === 0) {
      Logger.log(`Processed ${count} keywords...`);
    }
  }
  
  Logger.log(`Found ${count} active positive keywords across ${Object.keys(map.byId).length} campaigns`);
  return map;
}


/**
 * Checks if a negative keyword conflicts with a positive keyword based on text and match types.
 * Enhanced with more accurate conflict detection based on how Google Ads actually processes negatives.
 * 
 * @param {string} negativeText The negative keyword text
 * @param {string} negativeMatchType The negative keyword match type
 * @param {string} positiveText The positive keyword text
 * @param {string} positiveMatchType The positive keyword match type
 * @return {boolean} Whether a conflict exists
 */
function hasKeywordConflict(negativeText, negativeMatchType, positiveText, positiveMatchType) {
  // Normalize inputs
  negativeText = negativeText.toLowerCase().trim();
  positiveText = positiveText.toLowerCase().trim();
  
  // For BROAD match negatives:
  if (negativeMatchType === "BROAD") {
    // Split negative into individual words
    const negTerms = negativeText.replace(/^\+/, '').split(/\s+/);
    
    // For single-word broad negatives, check if it appears as a complete word in the positive
    if (negTerms.length === 1) {
      const negTerm = negTerms[0];
      // Use word boundary regex to ensure we're matching whole words, not parts of words
      const wordBoundaryRegex = new RegExp(`\\b${negTerm}\\b`, 'i');
      return wordBoundaryRegex.test(positiveText);
    }
    
    // For multi-word broad negatives, all words must appear in the positive (in any order)
    const posTerms = positiveText.split(/\s+/);
    return negTerms.every(term => {
      // Check if this term appears as a complete word in the positive keyword
      return posTerms.some(posTerm => posTerm === term);
    });
  }
  
  // For PHRASE match negatives:
  if (negativeMatchType === "PHRASE") {
    // Short phrase negatives (1-2 characters) often cause false positives
    // They need to appear as standalone words/phrases, not as part of larger words
    if (negativeText.length <= 2) {
      // Check if the short negative appears as a complete word/phrase
      const wordBoundaryRegex = new RegExp(`\\b${negativeText}\\b`, 'i');
      return wordBoundaryRegex.test(positiveText);
    }
    
    // For longer phrase negatives, check if they appear as a continuous phrase
    // Split both into words to ensure we're matching complete words
    const negWords = negativeText.split(/\s+/);
    const posWords = positiveText.split(/\s+/);
    
    // Look for the sequence of negative words within positive words
    for (let i = 0; i <= posWords.length - negWords.length; i++) {
      let match = true;
      for (let j = 0; j < negWords.length; j++) {
        if (posWords[i + j] !== negWords[j]) {
          match = false;
          break;
        }
      }
      if (match) return true;
    }
    
    return false;
  }
  
  // For EXACT match negatives:
  if (negativeMatchType === "EXACT") {
    // For exact match, the negative must match the entire positive keyword exactly
    return negativeText === positiveText;
  }
  
  return false;
}


/**
 * Removes ad group-level negative keywords that conflict with
 * the positive keywords in the same ad group.
 * 
 * @param {Object} positiveKeywordsMap The map of positive keywords
 * @param {Object} metrics The metrics object for tracking
 * @return {number} Number of conflicts removed
 */
function removeAdGroupNegativeConflicts(positiveKeywordsMap, metrics) {
  let conflictsRemoved = 0;
  
  // We need to iterate through ad groups to access their negative keywords
  // Only include ENABLED ad groups in ENABLED campaigns
  const adGroupIterator = AdsApp.adGroups()
    .withCondition("Status = ENABLED")
    .withCondition("CampaignStatus = ENABLED")
    .get();
  
  while (adGroupIterator.hasNext()) {
    const adGroup = adGroupIterator.next();
    const adGroupId = adGroup.getId();
    const campaignId = adGroup.getCampaign().getId();
    const adGroupName = adGroup.getName();
    const campaignName = adGroup.getCampaign().getName();
    
    // Get negative keywords for this ad group
    const negativeKeywordIterator = adGroup.negativeKeywords().get();
    
    while (negativeKeywordIterator.hasNext()) {
      const negativeKeyword = negativeKeywordIterator.next();
      const negText = negativeKeyword.getText().toLowerCase();
      const negMatchType = negativeKeyword.getMatchType();
      
      // Check for conflicts within this ad group
      let conflictFound = false;
      let conflictReason = '';
      
      // Check if this would have been flagged by the old algorithm
      let wouldBeOldConflict = false;
      
      if (positiveKeywordsMap.byId[campaignId] && 
          positiveKeywordsMap.byId[campaignId].adGroups[adGroupId]) {
        
        const adGroupKeywords = positiveKeywordsMap.byId[campaignId].adGroups[adGroupId].keywords;
        
        // Check each positive keyword in this ad group for conflicts
        for (const posText in adGroupKeywords) {
          const posMatchType = adGroupKeywords[posText];
          
          // Check if this would be flagged by the old algorithm
          if (negMatchType === "PHRASE" && posText.includes(negText)) {
            wouldBeOldConflict = true;
          } else if (negMatchType === "BROAD") {
            const negTerms = negText.replace(/^\+/, '').split(/\s+/);
            const posTerms = posText.split(/\s+/);
            if (negTerms.every(term => posTerms.includes(term.replace(/^\+/, '')))) {
              wouldBeOldConflict = true;
            }
          }
          
          // Check with our improved algorithm
          if (hasKeywordConflict(negText, negMatchType, posText, posMatchType)) {
            conflictFound = true;
            conflictReason = `Conflicts with positive keyword '${posText}' (${posMatchType})`;
            break;
          }
        }
      }
      
      // If it would have been flagged by old algorithm but not by new one, count as false positive avoided
      if (wouldBeOldConflict && !conflictFound) {
        metrics.falsePositivesAvoided++;
        if (CONFIG.detailedLogging) {
          Logger.log(`False positive avoided: '${negText}' (${negMatchType}) in Ad Group '${adGroupName}' would not actually block any keywords`);
        }
      }
      
      if (conflictFound) {
        metrics.adGroupConflictsFound++;
        
        const logMessage = `Ad Group Conflict: '${negText}' (${negMatchType}) in Ad Group '${adGroupName}' (${adGroupId}) - ${conflictReason}`;
        Logger.log(logMessage);
        
        if (!CONFIG.dryRun) {
          negativeKeyword.remove();
          conflictsRemoved++;
          Logger.log(`  ✓ Removed`);
        } else {
          Logger.log(`  ⚠ Would remove (dry run)`);
        }
      }
    }
  }
  
  return conflictsRemoved;
}


/**
 * Removes campaign-level negative keywords that conflict with
 * any positive keyword in the same campaign.
 * 
 * @param {Object} positiveKeywordsMap The map of positive keywords
 * @param {Object} metrics The metrics object for tracking
 * @return {number} Number of conflicts removed
 */
function removeCampaignNegativeConflicts(positiveKeywordsMap, metrics) {
  let conflictsRemoved = 0;
  
  // We need to iterate through campaigns to access their negative keywords
  // Only include ENABLED campaigns
  const campaignIterator = AdsApp.campaigns()
    .withCondition("Status = ENABLED")
    .get();
  
  while (campaignIterator.hasNext()) {
    const campaign = campaignIterator.next();
    const campaignId = campaign.getId();
    const campaignName = campaign.getName();
    
    // Get negative keywords for this campaign
    const negativeKeywordIterator = campaign.negativeKeywords().get();
    
    while (negativeKeywordIterator.hasNext()) {
      const negativeKeyword = negativeKeywordIterator.next();
      const negText = negativeKeyword.getText().toLowerCase();
      const negMatchType = negativeKeyword.getMatchType();
      
      // Check for conflicts with any positive keyword in this campaign
      let conflictFound = false;
      let conflictDetails = '';
      
      if (positiveKeywordsMap.byId[campaignId]) {
        // Check each ad group in the campaign
        for (const adGroupId in positiveKeywordsMap.byId[campaignId].adGroups) {
          const adGroupData = positiveKeywordsMap.byId[campaignId].adGroups[adGroupId];
          
          // Check each positive keyword in this ad group
          for (const posText in adGroupData.keywords) {
            const posMatchType = adGroupData.keywords[posText];
            
            if (hasKeywordConflict(negText, negMatchType, posText, posMatchType)) {
              conflictFound = true;
              conflictDetails = `Conflicts with '${posText}' (${posMatchType}) in Ad Group '${adGroupData.name}'`;
              break;
            }
          }
          
          if (conflictFound) break;
        }
      }
      
      if (conflictFound) {
        metrics.campaignConflictsFound++;
        
        const logMessage = `Campaign Conflict: '${negText}' (${negMatchType}) in Campaign '${campaignName}' (${campaignId}) - ${conflictDetails}`;
        Logger.log(logMessage);
        
        if (!CONFIG.dryRun) {
          negativeKeyword.remove();
          conflictsRemoved++;
          Logger.log(`  ✓ Removed`);
        } else {
          Logger.log(`  ⚠ Would remove (dry run)`);
        }
      }
    }
  }
  
  return conflictsRemoved;
}


/**
 * Removes negative keywords from shared negative keyword lists that conflict
 * with any positive keyword in the account.
 * 
 * @param {Object} positiveKeywordsMap The map of positive keywords
 * @param {Object} metrics The metrics object for tracking
 * @return {number} Number of conflicts removed
 */
function removeSharedListNegativeConflicts(positiveKeywordsMap, metrics) {
  let conflictsRemoved = 0;
  
  // Use the correct function name for shared negative keyword lists
  const sharedNegLists = AdsApp.negativeKeywordLists().get();
  
  if (!sharedNegLists.hasNext()) {
    Logger.log('No shared negative keyword lists found.');
    return conflictsRemoved;
  }
  
  while (sharedNegLists.hasNext()) {
    const negList = sharedNegLists.next();
    const listName = negList.getName();
    
    // Get campaigns that are actually using this shared negative keyword list
    const campaignsUsingList = negList.campaigns().get();
    const campaignsWithList = new Set();
    
    // Build a set of campaign IDs that are using this list
    while (campaignsUsingList.hasNext()) {
      const campaign = campaignsUsingList.next();
      campaignsWithList.add(campaign.getId());
    }
    
    if (campaignsWithList.size === 0) {
      Logger.log(`Shared list "${listName}" is not assigned to any campaigns. Skipping.`);
      continue;
    }
    
    Logger.log(`Checking shared list: ${listName} (assigned to ${campaignsWithList.size} campaigns)`);
    
    // Get negative keywords for this list
    const negKeywordIterator = negList.negativeKeywords().get();
    
    while (negKeywordIterator.hasNext()) {
      const negKeyword = negKeywordIterator.next();
      const negText = negKeyword.getText().toLowerCase();
      const negMatchType = negKeyword.getMatchType();
      
      // Check against positive keywords, but only in campaigns that use this list
      let conflictFound = false;
      let conflictDetails = '';
      let wouldBeOldConflict = false;
      
      // First do a quick check against the Set of all keyword texts
      const potentialDirectMatch = positiveKeywordsMap.allKeywords.has(negText);
      
      // Check each positive keyword for conflicts, but only in campaigns using this list
      for (const kwDetails of positiveKeywordsMap.keywordDetails) {
        // Skip if this keyword's campaign is not using the shared list
        if (!campaignsWithList.has(kwDetails.campaignId)) {
          continue;
        }
        
        // Check if this would be flagged by the old algorithm
        if (negMatchType === "PHRASE" && kwDetails.text.includes(negText)) {
          wouldBeOldConflict = true;
        } else if (negMatchType === "BROAD") {
          const negTerms = negText.replace(/^\+/, '').split(/\s+/);
          const posTerms = kwDetails.text.split(/\s+/);
          if (negTerms.every(term => posTerms.includes(term.replace(/^\+/, '')))) {
            wouldBeOldConflict = true;
          }
        }
        
        // Check with our improved algorithm
        if (hasKeywordConflict(negText, negMatchType, kwDetails.text, kwDetails.matchType)) {
          conflictFound = true;
          conflictDetails = `Conflicts with '${kwDetails.text}' (${kwDetails.matchType}) in Campaign '${kwDetails.campaignName}', Ad Group '${kwDetails.adGroupName}'`;
          break;
        }
      }
      
      // If it would have been flagged by old algorithm but not by new one, count as false positive avoided
      if (wouldBeOldConflict && !conflictFound) {
        metrics.falsePositivesAvoided++;
        if (CONFIG.detailedLogging) {
          Logger.log(`False positive avoided: '${negText}' (${negMatchType}) in shared list '${listName}' would not actually block any keywords`);
        }
      }
      
      if (conflictFound) {
        metrics.sharedListConflictsFound++;
        
        const logMessage = `Shared List Conflict: '${negText}' (${negMatchType}) in List '${listName}' - ${conflictDetails}`;
        Logger.log(logMessage);
        
        if (!CONFIG.dryRun) {
          negKeyword.remove();
          conflictsRemoved++;
          Logger.log(`  ✓ Removed`);
        } else {
          Logger.log(`  ⚠ Would remove (dry run)`);
        }
      }
    }
  }
  
  return conflictsRemoved;
}


/**
 * Generates a summary report of the conflicts found and actions taken.
 * 
 * @param {Object} metrics The metrics object with conflict counts
 * @return {string} A formatted report string
 */
function generateReport(metrics) {
  const totalConflictsFound = 
    metrics.adGroupConflictsFound + 
    metrics.campaignConflictsFound + 
    metrics.sharedListConflictsFound;
  
  const totalConflictsRemoved = 
    metrics.adGroupConflictsRemoved + 
    metrics.campaignConflictsRemoved + 
    metrics.sharedListConflictsRemoved;
  
  // Determine if we found any conflicts
  const noConflictsFound = totalConflictsFound === 0;
  
  // Create a more visually appealing HTML email
  let htmlReport = `
    <html>
    <head>
      <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; max-width: 800px; margin: 0 auto; }
        .header { background-color: #4285f4; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; }
        .summary { background-color: #f8f9fa; padding: 15px; border-radius: 5px; margin-bottom: 20px; }
        .details { margin-bottom: 20px; }
        table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
        th, td { padding: 10px; text-align: left; border-bottom: 1px solid #ddd; }
        th { background-color: #f2f2f2; }
        .success { color: #0f9d58; font-weight: bold; }
        .warning { color: #f4b400; }
        .footer { font-size: 12px; color: #666; border-top: 1px solid #eee; padding-top: 10px; }
        .no-conflicts { background-color: #e6f4ea; padding: 15px; border-radius: 5px; margin-bottom: 20px; border-left: 4px solid #0f9d58; }
        .false-positives { background-color: #e8f5e9; padding: 15px; border-radius: 5px; margin-bottom: 20px; border-left: 4px solid #4caf50; }
      </style>
    </head>
    <body>
      <div class="header">
        <h1>Google Ads Negative Keyword Conflict Report</h1>
        <p>${new Date().toLocaleString()}</p>
      </div>
      <div class="content">
        ${noConflictsFound ? `
        <div class="no-conflicts">
          <h2><span class="success">✓ Great job!</span></h2>
          <p>No negative keyword conflicts were found in your account. This means your negative keywords are properly configured and not blocking any of your positive keywords.</p>
          <p>Your account is optimized for maximum visibility for the keywords you're actively bidding on.</p>
        </div>
        ` : `
        <div class="summary">
          <h2>Summary</h2>
          <p><strong>Date Range:</strong> ${CONFIG.dateRange || 'All Time'}</p>
          <p><strong>Mode:</strong> ${CONFIG.dryRun ? 'DRY RUN (no changes made)' : 'ACTIVE (conflicts removed)'}</p>
          <p><strong>Total Conflicts Found:</strong> ${totalConflictsFound}</p>
          ${!CONFIG.dryRun ? `<p><strong>Total Conflicts Removed:</strong> ${totalConflictsRemoved}</p>` : ''}
        </div>
        
        <div class="details">
          <h2>Details</h2>
          <table>
            <tr>
              <th>Level</th>
              <th>Conflicts Found</th>
              ${!CONFIG.dryRun ? '<th>Conflicts Removed</th>' : ''}
            </tr>
            <tr>
              <td>Ad Group Level</td>
              <td>${metrics.adGroupConflictsFound}</td>
              ${!CONFIG.dryRun ? `<td>${metrics.adGroupConflictsRemoved}</td>` : ''}
            </tr>
            <tr>
              <td>Campaign Level</td>
              <td>${metrics.campaignConflictsFound}</td>
              ${!CONFIG.dryRun ? `<td>${metrics.campaignConflictsRemoved}</td>` : ''}
            </tr>
            <tr>
              <td>Shared Lists</td>
              <td>${metrics.sharedListConflictsFound}</td>
              ${!CONFIG.dryRun ? `<td>${metrics.sharedListConflictsRemoved}</td>` : ''}
            </tr>
          </table>
        </div>
        
        <div class="impact">
          <h2>Impact Analysis</h2>
          <p>Removing these negative keyword conflicts will help ensure your ads show for the keywords you're actively bidding on, potentially increasing your impression share and overall campaign performance.</p>
          
          <p>By resolving these conflicts, you've eliminated situations where your ads were being blocked by your own negative keywords, which could lead to:</p>
          <ul>
            <li><strong>Increased visibility</strong> for your targeted keywords</li>
            <li><strong>Better ad spend efficiency</strong> by ensuring budget goes to keywords you want to target</li>
            <li><strong>Improved campaign consistency</strong> across your account</li>
          </ul>
        </div>
        `}
        
        ${metrics.falsePositivesAvoided > 0 ? `
        <div class="false-positives">
          <h2>False Positives Avoided: ${metrics.falsePositivesAvoided}</h2>
          <p>These are potential conflicts that would have been incorrectly flagged by simpler detection methods.</p>
          <p>Examples include phrase match negatives like "mini" not actually blocking "mining" or "a c" not blocking "ammonia compressor".</p>
        </div>
        ` : ''}
        
        <div class="footer">
          <p>This report was generated automatically by the Google Ads Negative Keyword Conflict Resolver script.</p>
          <p>For questions or support, please contact your account manager.</p>
        </div>
      </div>
    </body>
    </html>
  `;
  
  // Also create a plain text version for the logs and for email clients that don't support HTML
  let textReport = '\n' + '='.repeat(50) + '\n';
  textReport += 'NEGATIVE KEYWORD CONFLICT RESOLUTION REPORT\n';
  textReport += '='.repeat(50) + '\n\n';
  
  if (noConflictsFound) {
    textReport += 'GREAT JOB! NO CONFLICTS FOUND\n';
    textReport += 'Your account is optimized with no negative keyword conflicts.\n';
    textReport += 'This means your negative keywords are properly configured and not blocking any of your positive keywords.\n\n';
  } else {
    textReport += 'SUMMARY:\n';
    textReport += '- Date Range: ' + (CONFIG.dateRange || 'All Time') + '\n';
    textReport += '- Mode: ' + (CONFIG.dryRun ? 'DRY RUN (no changes made)' : 'ACTIVE (conflicts removed)') + '\n';
    textReport += '- Total Conflicts Found: ' + totalConflictsFound + '\n';
    
    if (!CONFIG.dryRun) {
      textReport += '- Total Conflicts Removed: ' + totalConflictsRemoved + '\n';
    }
    
    textReport += '\nDETAILS:\n';
    textReport += '- Ad Group Level: ' + metrics.adGroupConflictsFound + ' conflicts found';
    if (!CONFIG.dryRun) {
      textReport += ', ' + metrics.adGroupConflictsRemoved + ' removed';
    }
    textReport += '\n';
    
    textReport += '- Campaign Level: ' + metrics.campaignConflictsFound + ' conflicts found';
    if (!CONFIG.dryRun) {
      textReport += ', ' + metrics.campaignConflictsRemoved + ' removed';
    }
    textReport += '\n';
    
    textReport += '- Shared Lists: ' + metrics.sharedListConflictsFound + ' conflicts found';
    if (!CONFIG.dryRun) {
      textReport += ', ' + metrics.sharedListConflictsRemoved + ' removed';
    }
    textReport += '\n';
    
    textReport += '\nIMPACT:\n';
    textReport += '- Removing these conflicts will increase visibility for your targeted keywords\n';
    textReport += '- Better ad spend efficiency by ensuring budget goes to keywords you want to target\n';
    textReport += '- Improved campaign consistency across your account\n';
  }
  
  // Add false positives avoided to the report
  if (metrics.falsePositivesAvoided > 0) {
    textReport += `\nFALSE POSITIVES AVOIDED: ${metrics.falsePositivesAvoided}\n`;
    textReport += 'These are potential conflicts that would have been incorrectly flagged by simpler detection methods.\n';
    textReport += 'Examples include phrase match negatives like "mini" not actually blocking "mining" or "a c" not blocking "ammonia compressor".\n';
  }
  
  textReport += '\n' + '-'.repeat(50) + '\n';
  textReport += 'Script executed on: ' + new Date().toLocaleString() + '\n';
  textReport += '='.repeat(50) + '\n';
  
  // For email, use the HTML version - only send once
  if (CONFIG.sendEmail) {
    MailApp.sendEmail({
      to: CONFIG.emailAddress,
      subject: noConflictsFound ? 'Google Ads: No Negative Keyword Conflicts Found ✓' : 'Google Ads Negative Keyword Conflict Report',
      htmlBody: htmlReport,
      body: textReport // Fallback plain text version
    });
    
    Logger.log("Email report sent to: " + CONFIG.emailAddress);
  }
  
  // For logs, return the text version
  return textReport;
}

/**
 * Validates the conflict detection algorithm against known examples.
 * This helps ensure our conflict detection is accurate.
 */
function validateConflictDetection() {
  const testCases = [
    // Format: [negative text, negative match type, positive text, expected result, description]
    ["a 1", "PHRASE", "wsda 1.25", false, "False conflict: 'a 1' in 'wsda 1.25'"],
    ["r5 ra", "PHRASE", "busch r5 ra 0025", true, "Real conflict: 'r5 ra' in 'busch r5 ra 0025'"],
    ["ra 0255", "PHRASE", "busch r5 ra 0255 d", true, "Real conflict: 'ra 0255' in 'busch r5 ra 0255 d'"],
    ["a c", "PHRASE", "liquid ring ammonia compressor", false, "False conflict: 'a c' in 'ammonia compressor'"],
    ["medical", "BROAD", "medical liquid ring pump", true, "Real conflict: 'medical' in 'medical liquid ring pump'"],
    ["ed", "PHRASE", "edwards gsx750", false, "False conflict: 'ed' in 'edwards'"],
    ["oil", "PHRASE", "nash vectra oil seal package", true, "Real conflict: 'oil' in 'nash vectra oil seal package'"],
    ["mini", "PHRASE", "copper mining liquid ring pump", false, "False conflict: 'mini' in 'mining'"],
    ["kd", "PHRASE", "kinney kdp", false, "False conflict: 'kd' in 'kdp'"],
    ["small", "PHRASE", "small capacity liquid ring pump", true, "Real conflict: 'small' in 'small capacity liquid ring pump'"],
    ["chamber", "PHRASE", "vacuum chamber liquid ring", true, "Real conflict: 'chamber' in 'vacuum chamber liquid ring'"],
    ["ac", "PHRASE", "nash sc 6 vacuum pump", false, "False conflict: 'ac' in 'nash sc 6 vacuum pump'"],
    ["busch", "PHRASE", "busch dolphin la", true, "Real conflict: 'busch' in 'busch dolphin la'"],
    ["rotary vane", "PHRASE", "oil circulated rotary vane pump", true, "Real conflict: 'rotary vane' in 'oil circulated rotary vane pump'"],
    ["milking", "PHRASE", "milking system liquid ring", true, "Real conflict: 'milking' in 'milking system liquid ring'"],
    ["rietschle", "PHRASE", "elmo rietschle s", true, "Real conflict: 'rietschle' in 'elmo rietschle s'"]
  ];
  
  Logger.log("Validating conflict detection algorithm...");
  let passedTests = 0;
  let failedTests = 0;
  
  for (const test of testCases) {
    const [negText, negMatchType, posText, expectedResult, description] = test;
    const actualResult = hasKeywordConflict(negText, negMatchType, posText, "BROAD"); // Assuming positive is broad
    
    if (actualResult === expectedResult) {
      passedTests++;
      if (CONFIG.detailedLogging) {
        Logger.log(`✓ PASS: ${description}`);
      }
    } else {
      failedTests++;
      Logger.log(`✗ FAIL: ${description} - Expected ${expectedResult} but got ${actualResult}`);
    }
  }
  
  Logger.log(`Validation complete: ${passedTests} passed, ${failedTests} failed`);
  return failedTests === 0;
} 
