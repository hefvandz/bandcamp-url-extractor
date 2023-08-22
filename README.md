# bandcamp-url-extractor

This is a Google Apps Script project that extracts URLs from Gmail threads and saves them to a Google Sheets spreadsheet. Super specific to my own workflow but has saved me hours in manually opening emails and links. Pairs well with a bulk URL opener tool.

## Features

- Extracts Bandcamp URLs from new release emails and new purchase emails from users you follow
- Checks for duplicates and saves unique extracted URLs to a Google Sheets spreadsheet
- Mark email threads as processed to prevent re-processing

## Usage

1. Set up your Gmail label and customize the variables in the script.
2. Run the `extractUrlsAndSaveToSheetWithLiveProgress` function from the Apps Script editor.
3. The script will process threads, extract URLs, and save them to the designated spreadsheet in batches of 500 threads per run.

## Configuration

**Gmail Label Setup**

Before running the script, you need to set up a Gmail label to organize the threads you want to process. 

Search in Gmail and create filters for each of these:

      `from:(noreply@bandcamp.com), subject:(new releases)`
     
      `from:(noreply@bandcamp.com), subject:(New release from)`
      
      `from:(noreply@bandcamp.com), subject:(bought new music)`

   - In the filter creation window, check the box next to "Apply the label."
   - Select the label you want to use to label music coming in from Bandcamp.
   - You can also choose to apply the filter to existing messages that match the criteria.

Now your Gmail label is set up to organize the threads you want to process. The Google Apps Script will target this label to extract URLs and save them to a Google Sheets spreadsheet.

Remember to replace `labelName` in the script code with the exact label name you used, ensuring it matches the label you've created.

**Create Apps Script Project:**
   - Open your Google Sheets document.
   - Click on `Extensions > Apps Script` to open the Apps Script editor.
   - Delete any existing code and paste in the code provided in this repository.

**Add Gmail Service to Apps Script Project:**
   - In the Apps Script editor, go to `Services +`.
   - Enable the Gmail API.

**Enable Gmail API:**
   - Create a project on [Google Cloud Console](https://console.cloud.google.com/).
   - Enable the Gmail API for your project, go to `APIs & Services > Enabled APIs & Services`
   - Add your email as a test user under the "OAuth 2.0 Client IDs" section.
  
**Link Cloud Project to Apps Script Project:**
   - In the Apps Script editor, go to `Project Settings > Google Cloud Platform (GCP) Project`.
   - Enter your Project ID.

Before running the script, you need to customize the following variables:

- `labelName`: The name of the Gmail label containing threads to process.
- `processedLabelName`: The label name to mark processed threads.
- `maxExecutionTime`: Maximum execution time for the script.
- `spreadsheetId`: ID of the Google Sheets spreadsheet to save URLs.

## Other Notes
  - As you are running as a test user, the auth will expire every week or so.

## Code

<pre>
var labelName = "NEWMUSICLABELNAME"; // Replace this with the name of the top-level label you want to process.
var batchSize = 25; // Set the number of threads to process in each batch.
var totalThreadsToProcess = 500; // Set the total number of threads to process.
var processedLabelName = "PROCESSEDLABELNAME"; // Replace this with the name of the label to mark processed threads.
var maxExecutionTime = 5 * 60 * 1000; // 5 minutes in milliseconds
var spreadsheetId = "SHEETNAME"; // Replace this with your new spreadsheet ID.

function extractUrlsAndSaveToSheetWithLiveProgress() {
  Logger.log("Execution started");

  // Get the top-level label.
  var label = GmailApp.getUserLabelByName(labelName);

  var totalProcessedThreads = 0; // Variable to keep track of the total processed threads.
  var startTime = new Date().getTime();

  while (totalProcessedThreads < totalThreadsToProcess) {
    var threads = label.getThreads(0, batchSize);

    if (threads.length === 0) {
      // No more threads to process. Exit the loop.
      break;
    }

    for (var i = 0; i < threads.length; i++) {
      var thread = threads[i];

      // Skip threads that have the "Processed" label.
      if (threadHasProcessedLabel(thread)) {
        continue;
      }

      Logger.log("Processing thread " + (totalProcessedThreads + 1) + " of " + totalThreadsToProcess);

      var messages = thread.getMessages();

      for (var j = 0; j < messages.length; j++) {
        var message = messages[j];
        var body = message.getPlainBody();
        var urls = body.match(/https?:\/\/[^\s]+(?:track|album)[^\s]+/gi);

        if (urls) {
          urls.forEach(function (url) {
            // Modify this line to address the edge case
            var isTrackOrAlbumUrl = /bandcamp[^"]*\/(?:track|album)/gi.test(url);

            if (isTrackOrAlbumUrl) {
              // Remove everything after the first instance of "?from=fanpub".
              var fromFanpubIndex = url.indexOf("?from=fanpub");
              if (fromFanpubIndex !== -1) {
                url = url.substring(0, fromFanpubIndex);
              }

              // Remove leading and trailing white spaces.
              url = url.trim();

              // Check if the URL is already in the sheet.
              if (!isUrlAlreadyInSheet(url, spreadsheetId)) {
                appendToSheet(url, spreadsheetId);
                Logger.log("Link added to the sheet: " + url); // Log URL addition to the sheet.
              }
            }
          });
        }
      }

      // Apply the "Processed" label to the thread to mark it as processed.
      markThreadAsProcessed(thread);

      // Increase the count of processed threads.
      totalProcessedThreads++;

      if (totalProcessedThreads >= totalThreadsToProcess) {
        break;
      }
    }

    // Introduce a delay to avoid exceeding Google Sheets quotas.
    Utilities.sleep(1000);

    var currentTime = new Date().getTime();
    // Check if the execution time is close to the maximum limit, and if so, break the loop.
    if (currentTime - startTime > maxExecutionTime) {
      break;
    }
  }

  Logger.log("Execution completed. Number of threads processed: " + totalProcessedThreads);
}

function threadHasProcessedLabel(thread) {
  var processedLabel = GmailApp.getUserLabelByName(processedLabelName);
  var labels = thread.getLabels();
  for (var i = 0; i < labels.length; i++) {
    if (labels[i].getName() === processedLabel.getName()) {
      return true;
    }
  }
  return false;
}

function isUrlAlreadyInSheet(url, spreadsheetId) {
  // Check if the URL is already present in the sheet.
  var sheet = SpreadsheetApp.openById(spreadsheetId).getSheetByName("Sheet1");
  var data = sheet.getDataRange().getValues();
  for (var i = 1; i < data.length; i++) {
    if (data[i][0] === url) {
      return true;
    }
  }
  return false;
}

function appendToSheet(url, spreadsheetId) {
  // Append the URL to the sheet.
  var sheet = SpreadsheetApp.openById(spreadsheetId).getSheetByName("Sheet1");
  sheet.appendRow([url]);
}

function markThreadAsProcessed(thread) {
  // Function to apply the "Processed" label to the thread.
  var processedLabel = GmailApp.getUserLabelByName(processedLabelName);
  thread.addLabel(processedLabel);

  // Remove the "new music" label to prevent re-processing in future runs.
  var label = GmailApp.getUserLabelByName(labelName);
  thread.removeLabel(label);
}

</pre>
  

