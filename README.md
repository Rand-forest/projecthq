# projecthq
Gemini Canvas Project Dashboard

📊 ProjectHQ: Google Sheets Project Management Dashboard

ProjectHQ is a lightweight, serverless Project Management Dashboard that runs entirely in your web browser and uses Google Sheets as its database.
It features dynamic forms, automatic project progress calculation, native task filtering, and an interactive Gantt chart. Because it relies on Google Apps Script, it seamlessly bypasses corporate firewalls while maintaining your organization's strict Google Workspace security.
✨ Features
Zero Database Hosting: Runs 100% off a standard Google Sheet.
Full CRUD: Create, Read, Update, and Delete projects and tasks directly from the dashboard.
Interactive Gantt Charts: Automatically generates timelines based on task Start/End dates.
Auto-Calculated Progress: Project progress automatically updates based on the completion status of its underlying tasks.
Dynamic Forms: Add a new column to your Google Sheet, and the dashboard will automatically create a form field for it—no code updates required!
Native Tooltips: Hover over tasks to read multi-line notes and comments.
🛠️ Step 1: Set up the Google Sheet (The Database)
Go to Google Drive and create a new Google Sheet.
Rename the first tab at the bottom to exactly: Projects
In Row 1 of the Projects tab, add the following column headers (exact spelling matters for some features):
Project Name
Status
Project Manager
Due Date
Progress (%)
Click the + button at the bottom left to add a second tab, and name it exactly: Tasks
In Row 1 of the Tasks tab, add the following column headers:
Project Name
Task Name
Status
Assignee
Start Date
End Date
Progress (%)
Notes
(Optional but Recommended): Highlight your Status columns, click Data > Data Validation, and create Drop-down rules (e.g., Planning, Execution, Testing, Completed). The dashboard will automatically read these rules and populate its own dropdown menus!
🔌 Step 2: Set up Google Apps Script (The API Bridge)
This script allows the HTML dashboard to securely talk to your Google Sheet.
In your Google Sheet, click Extensions > Apps Script from the top menu.
Delete any default code in the editor, and paste in the following script:
function doGet(e) {
  var lock = LockService.getScriptLock();
  lock.tryLock(10000);

  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var targetSheetName = e.parameter.sheetName;
    var sheet = targetSheetName ? ss.getSheetByName(targetSheetName) : ss.getSheets()[0];
    
    if (!sheet) throw new Error("Could not find a tab named '" + targetSheetName + "'.");

    var action = e.parameter.action || 'read';

    if (action === 'create') {
      var rowData = JSON.parse(e.parameter.rowData);
      sheet.appendRow(rowData);
    } else if (action === 'update') {
      var row = parseInt(e.parameter.row);
      var rowData = JSON.parse(e.parameter.rowData);
      sheet.getRange(row, 1, 1, rowData.length).setValues([rowData]);
    } else if (action === 'delete') {
      var row = parseInt(e.parameter.row);
      sheet.deleteRow(row);
    }

    SpreadsheetApp.flush(); 

    var data = sheet.getDataRange().getDisplayValues();
    var rows = [];
    var headers = [];
    var statusOptions = []; 

    if (data.length > 0) {
      headers = data[0].map(function(h) { return h.trim(); });
      
      var statusColIndex = headers.findIndex(function(h) { 
        return h.toLowerCase() === 'status' || h.toLowerCase() === 'state'; 
      });
      
      if (statusColIndex !== -1 && sheet.getLastRow() >= 2) {
        var rule = sheet.getRange(2, statusColIndex + 1).getDataValidation();
        if (rule != null) {
          var criteriaType = rule.getCriteriaType();
          if (criteriaType == SpreadsheetApp.DataValidationCriteria.VALUE_IN_LIST) {
            statusOptions = rule.getCriteriaValues()[0];
          } else if (criteriaType == SpreadsheetApp.DataValidationCriteria.VALUE_IN_RANGE) {
            var rangeValues = rule.getCriteriaValues()[0].getValues();
            statusOptions = rangeValues.reduce(function(a, b) { return a.concat(b); }, []).filter(String);
          }
        }
      }

      for (var i = 1; i < data.length; i++) {
        var rowObj = { _row: i + 1 };
        for (var j = 0; j < headers.length; j++) {
          rowObj[headers[j]] = data[i][j];
        }
        rows.push(rowObj);
      }
    }

    var result = { status: 'success', headers: headers, data: rows, statusOptions: statusOptions };
    var jsonString = JSON.stringify(result);

    if (e && e.parameter && e.parameter.callback) {
      return ContentService.createTextOutput(e.parameter.callback + '(' + jsonString + ');').setMimeType(ContentService.MimeType.JAVASCRIPT);
    }
    return ContentService.createTextOutput(jsonString).setMimeType(ContentService.MimeType.JSON);

  } catch (err) {
    var errResult = { status: 'error', message: err.toString() };
    var errString = JSON.stringify(errResult);
    if (e && e.parameter && e.parameter.callback) {
      return ContentService.createTextOutput(e.parameter.callback + '(' + errString + ');').setMimeType(ContentService.MimeType.JAVASCRIPT);
    }
    return ContentService.createTextOutput(errString).setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}

Click the Save icon (floppy disk).
Click the blue Deploy button at the top right > New deployment.
Click the Gear icon next to "Select type" and choose Web app.
Set the settings as follows:
Execute as: Me (your email)
Who has access: Anyone within your organization (or Anyone if using a personal Gmail account).
Click Deploy. (You may be prompted to grant authorization permissions. Click Review Permissions > choose your account > Advanced > Go to script).
Copy the "Web app URL" provided. It will look like: https://script.google.com/macros/s/.../exec.
(Note: If you ever change the script code in the future, you MUST deploy it as a "New Version" via Manage Deployments, or the changes won't take effect).
💻 Step 3: Set up the Dashboard (The Frontend)
Open a text editor on your computer (like Notepad on Windows or TextEdit on Mac).
Paste the provided HTML code for the dashboard into the file.
Scroll down to the bottom of the code (around line 355) and look for the ⚙️ SETUP section.
Paste the Web App URL you copied in Step 2 between the quotation marks:

let activeScriptUrl = "[https://script.google.com/a/macros/.../exec](https://script.google.com/a/macros/.../exec)"; 

Save the file to your computer as ProjectDashboard.html (Make sure the file extension is .html, not .txt).
Double-click the saved file. It will open in your default web browser and instantly load your data!
🚀 How to Share with Your Team
Because this dashboard requires no database servers, you have several easy ways to share it with your team:
Option A: Google Sites (Recommended for Corporate Users)
Go to sites.google.com and create a blank site.
Click Embed in the right-hand menu, switch to the Embed Code tab.
Paste the entire content of ProjectDashboard.html into the box.
Publish the site and share the link with your team. They will access the tool securely using their existing Google workspace login.
Option B: Intranet / SharePoint Simply upload the ProjectDashboard.html file to your company's internal web server, SharePoint, or shared drive, and provide the URL to your team.
🧠 Advanced Usage Tips
Adding Custom Columns: Want to track "Budget" or "Priority"? Just type a new column header in your Google Sheet. Refresh the dashboard, and a new text input will automatically appear in your Edit/Add forms!
Gantt Chart Formatting: The Gantt chart relies on the columns Start Date and End Date. If you rename these, ensure the word "Date", "Start", or "End" is in the header so the dashboard knows to format them correctly.
Auto-Progress: Don't manually type project progress! Whenever you mark a task as "Completed" in the task view, the overarching Project's progress bar will automatically calculate based on the ratio of completed tasks vs total tasks.

