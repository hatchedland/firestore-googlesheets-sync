# firestore-googlesheets-sync
Cloud function to keep firestore and google sheet synced

This document details how the **`dataToSheetsFromFirestore`** Firebase Cloud Function works, as well as how it writes data to Google Sheets via the helper function **`writeToGoogleSheet`**.

---

## Overview

1. **Firestore Trigger**  
   - Triggered whenever a document in the `collection_name` collection is created or updated.

2. **Data Preparation**  
   - Extracts fields you want in the sheet.
   - Bundles the data in a standardized format for Sheets.

3. **Google Sheets Update**  
   - Uses a service account for **JWT** authentication. (not required if using firebase functions)
   - Checks if the target sheet exists; if not, creates it.
   - Appends new rows if a record with `id` (Firestore doc ID) does not exist.
   - Updates existing rows if the `id` already exists in the sheet.

---

## Main Function: `dataToSheetsFromFirestore`

```js
export const dataToSheetsFromFirestore = onDocumentWritten(
  "<collection_name>/{id}",
  async (event) => {
    // 1) Grab the Document ID from Firestore
    const docId = event.params.id;
    // 2) Get the changed data from Firestore
    const newData = event.data.after.data();

    // If the document is deleted, stop execution
    if (!newData) {
      console.log(`Document ${docId} was deleted. Ignoring update.`);
      return;
    }


      // Prepare data for Google Sheets
      const formattedData = [
        {
          id: docId,
          ...
        },
      ];

      console.log(formattedData, "data to be sent to google sheet");

      // Send to helper function that writes to Google Sheets
      await writeToGoogleSheet(formattedData, SPREADSHEET_ID, SHEET_NAME);
    }
  }
);



// Helper function writeToGoogleSheet


export const writeToGoogleSheet = async (
  data: any[], 
  spreadsheetId: string, 
  sheetName: string
) => {
  if (!data || data.length === 0) {
    console.warn("No data to write.");
    return;
  }

  try {
    // 1) Service account credentials
    const sheetsCreds = {
      type: <service_acc_key>,
      project_id: <project_id_key>,
      private_key_id: <private_key_id>
      private_key: <private_key>,
      client_email: <client_email>,
      client_id: <client_id>,
      token_uri: <token_uri>,
    };

    // 2) Authorize with JWT
    const jwtClient = new google.auth.JWT(
      sheetsCreds.client_email,
      null,
      sheetsCreds.private_key,
      ["https://www.googleapis.com/auth/spreadsheets"]
    );
    await jwtClient.authorize();

    // 3) Initialize Sheets API
    const sheets = google.sheets({ version: "v4", auth: jwtClient });

    // 4) Check if the target sheet exists
    const spreadsheet = await sheets.spreadsheets.get({ spreadsheetId });
    const sheetExists = spreadsheet.data.sheets?.some(
      sheet => sheet.properties?.title === sheetName
    );

    // 5) If it doesn't exist, create it
    if (!sheetExists) {
      await sheets.spreadsheets.batchUpdate({
        spreadsheetId,
        requestBody: {
          requests: [
            {
              addSheet: {
                properties: {
                  title: sheetName,
                  gridProperties: { rowCount: 100, columnCount: 10 },
                },
              },
            },
          ],
        },
      });
    }

    // 6) Define consistent column order
    const fixedColumns = [
      "id",
      ...
    ];

    // 7) Retrieve existing rows in the sheet (A:H)
    const getRows = await sheets.spreadsheets.values.get({
      spreadsheetId,
      range: `${sheetName}!A:H`,
    });
    const rows = getRows.data.values || [];

    // 8) Loop through each item in 'data'
    for (const item of data) {
      // Match values to the fixed column order
      const rowData = fixedColumns.map(field => item[field] || "");
      const docId = rowData[0]; // 'id' is the first column

      console.log(`Processing ID: ${docId}`);

      // 9) Check if docId already exists in the sheet
      const rowIndex = rows.findIndex(row => row[0] === docId);

      if (rowIndex === -1) {
        // 10) Append a new row
        await sheets.spreadsheets.values.append({
          spreadsheetId,
          range: `${sheetName}!A1`,
          valueInputOption: "RAW",
          insertDataOption: "INSERT_ROWS",
          requestBody: { values: [rowData] },
        });
        console.log(`New record ${docId} added.`);
      } else {
        // 11) Update the existing row
        const updateRange = `${sheetName}!A${rowIndex + 1}:H${rowIndex + 1}`;
        await sheets.spreadsheets.values.update({
          spreadsheetId,
          range: updateRange,
          valueInputOption: "RAW",
          requestBody: { values: [rowData] },
        });
        console.log(`Record ${docId} updated.`);
      }
    }

    console.log("Sheet update completed.");
  } catch (error) {
    console.error("Error writing to Google Sheet:", error);
  }
};

