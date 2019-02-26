# expensinator
A motley collections of ScanBot iOS app, MacOS folders, Automator workflows and Google Sheets that come together to help tame my receipts and expenses issues

How does this all work?

Good question! On my quest to tame my monstrous receipt problem, things got out of hand.  Would it have been quicker to have just done them all manually?  Yup.  Wouldn't have been so much fun though.

My aim here was to allow me to scan a receipt at the point of acquisition, store it safely in the cloud, harvest the key meta-data required to log it later in Kimble, so I can throw the paper receipt away!  Note, I don't want to bombard my boss with expense reports for each single receipt, so I also wanted a weekly batch job to run that would harvest all the data from the Google sheet, combine all the individual PDFs into one file, and then upload it all into Kimble.  I figure one expense submission a week is about ok.  

Components needed to make this work.  You may as well set these up as you go through them now!

1. Scanbot mobile app.  It is cheap (one-off payment unlike Expsivfy), and full featured.  It allows you to scan and OCR a receipt, and use custom one-click tags and templates to name your files.  So for example, when I scan a receipt today, it automatically names it "Receipt-20190222-" and then I can click a button to add "Subsistence-" and another to add "Drink-" and it remembers frequently used options so will offer "1.70" which is the price of a coffee on my client site.  Quick and pretty painless.  It the saves this to the cloud, in my case iCloud. When I emailed them to ask if they were going to start creating custom labels based on the OCR text, they even said they are working on it, although they confessed it is hard.  Scanning hundreds of receipts I can totally see why.

2. An Apple mac (laptop or desktop but not iOS or tvOS - although that would be cool), with a cloud service sync facility like iCloud.  Box, OneDrive and Dropbox all do this.  This means that when I scan a receipt on my phone, it automagically turns up on my laptop in a Finder folder.

3. Google sheet.  The idea here is to create a spreadsheet with all the key data in, that we can add to as we accrue receipts throughout the week.  We can then download it all to a csv once a week for processing into Kimble.  Strictly, I could have just done all this with a local csv.  I didn't for two reasons: 1) I wanted to have the data cloud based* and 2) it seemed like more fun.  * A csv saved locally to a cloud synced folder would have done this too.  As it turns out, I don't think this way was any harder really, but it does introduce a dependancy on having access to Google sheets.  You'll need your own spreadsheet and to publish it and then use that URL to update my scripts to get this to work.  You can't use my spreadsheet, thank you.  Potential security issue here is that anyone with that sheet link can access it right now, at least until I figure out how to do this using api keys.  Column headings:'doc_type','date','exp_type','exp_desc','cost','text'.  Strictly, I don't really need expense type here as they are all receipts.  I just thought it might come in handy for some other project at a later date.  Same also for text - I was considering dumping the OCR text in the sheet here.  That is a bit of a pain though as OCR can generate some freaky characters, so you'd need to do a good job on escaping out all the text.  And right now I am posting all the info up via curl, so for now, I have left it there but unused.
You need to create your google sheet with these exact headings, then check out this article here on how to make that sheet accessible via curl.  I leave all my data on the tab named "Sheet1".  I plan on creating other sheets to archive submitted expense data off to.  But the main handling sheet will always be "Sheet1".

http://mashe.hawksey.info/2014/07/google-sheets-as-a-database-insert-with-apps-script-using-postget-methods-with-ajax-example/


Curl allows us to add (or delete) a row to the sheet, simply by posting info to a URL, putting our data into the query string.


3. The following Finder folders, setup on your iCloud sync share.  I put mine as subdirectories of the Scanbot iCloud folder.  Note, if you change the names of these, or the paths, you will need to update those locations in the scripts later!

	3.1 "Curl Results" - used to log successful/failed data upload attempts to Google sheets.  "Receipt notifications.workflow" is attached to this folder as a folder action.  When any new result file is added here, this automator script runs, figures out the result, and pushes out a system notification.  Feel free to omit if you don't like this.
	
	3.2 "Ready to move" - after a receipt has been processed and key data logged in the Google sheet, it gets moved here, waiting to be be picked up later by a weekly job executed by the calendar.  No folder actions attached to this, but there is a calendar action that runs once weekly that will look for this folder's contents.
	
	3.3 "Receipts" - all newly scanned receipts get dropped in here.  The script "Receipt Handler Workflow.workflow" is attached to this folder, and is triggered whenever the system detects a new file being added.
	
	3.4 "Receipts OCR" - the "Receipt Handler Workflow.workflow" script (only triggered on adding files to the Receipts folder) extracts the OCR text from the PDF, breaking it out into its own .txt file and dumping it here for posterity.  Not really used for anything - I thought it might be fun to include it for some purpose later.  Feel free to omit this (and the steps from the workflows that create them).  The OCR text remains embedded in the PDFs either way.
	
	3.5 "Upload to Kimble" - the script "Create Kimble Expense.workflow" is triggered whenever a file is added here.  Two files are added here [NOTE: this may need changing to only putting one file in, and one in another separate folder].  Two files are added here by the weekly (calendar attached/triggered script) "Weekly Receipt Batch Upload to Kimble.app(Calendar alarm)".  The first file is a downloaded csv from the google sheet.  The second is a pdf that merges all the individual pdf receipts into one file.  This makes attaching it in Kimble a bit easier.


4. A bunch of scripts, well, technically Automator workflows. Why Automator?  A few reasons.  Firstly, it is probably the easiest route to adding a script so that it can be triggered by a calendar reminder or as a Finder Action so that it gets triggered when something gets added to a specific folder.  Next, there are a lot of pre-built actions you can use, like the one that pulls the OCR text out of the PDFs and saves it to a new file.  But best of all, you can easily use them, chaining multiple steps together to trigger Javascript or Applescript, passing the output of one to the input of the next.  The editor for these is awful admittedly, but using versatility, plus better handling of security (if you add scripts to folders directly in Applescript, as soon as you add your script, you can no longer edit it, so have to keep moving it between a working location and the desired location, which is a PITA frankly.

I've already mentioned the laptop-hosted scripts/workflows, but here they are again, in process-order.

	4.1 "Receipt Handler Workflow.workflow (Folder Action)" is attached to folder "Receipts" and fires whenever a new file is detected in the folder by Finder.  It creates an OCR txt file, extracts the key metadata from the filename and CURLs it to the Google sheet, logs the success/failure of the CURL attempt, and then moves the original PDF to a folder called "Ready to move".  This is the end of this process; files now sit around waiting for a weekly scheduled job to pick them up.
	4.2 "Weekly Receipt Batch Upload to Kimble.app (Calendar Alarm)" is a script attached to a calendar item in Apple Calendar app.  Essentially this is triggered by a diary entry, which is nice because you can set it to repeat on all kinds of weird schedules should you so choose.  My meeting is set to recur every Friday afternoon, and then run this script.  This script just downloads the Google sheet as a .csv file, and then renames the file, and moves it to the folder "Upload to Kimble"
	4.3 "Create Kimble Expense.workflow (Folder Action)".  This script is triggered when the .csv file in the preceding step is moved into the folder "Upload to Kimble". First off it gets all the receipts ready to move and combines them into one pdf, then it prompts you to open a Kimble window in Safari, and log in.  It'll hold at this point until you acknowledge, or abort if you cancel. It will then open up a new expense claim form in Kimble, and start to populate it with all the entries, and upload the single combined PDF created earlier.  Once it is done it will save, but NOT submit the form.  You could script it to do this, but that would be a bit reckless, because ultimately, these kinds of scripts do sometimes fail, and you should also check for duplicates - which this script does not.  The last thing this script does is to Curl to the Google sheet a new row which is null with the exception of the doc_type which is set to "EndOfClaim".  

[TO-DO - add a Google sheets script to the sheet to watch out for the "EndOfClaim" row being added, and when that happens it should rename the current sheet from "Sheet1" to "ArchiveYYYYMMDD", and then create a new sheet called "Sheet1".



KNOWN ISSUES:
1. Finder "on add" events are a bit flaky.  If you only add one file at a time, then it works really nicely.  If on the hand you add 50 files all together, it has a habit of not catching them all.  I've not coded for this issue (yet) but it can be avoided by a) Not adding 50 receipts all in one go, b) if you do, keep an eye on the results.  Mostly this happens if you scan a backlog of receipts whilst your laptop is switched off - when it comes back online it will sync and suddenly find those 50 new rceipts.  If you simply must do a batch, you can avoid the issue by leaving your Mac on and running.
2. Only handles "Subsistence" type claims: dropdown hardcoded so does not use value from spreadsheet.
3. I am not entirely sure the PDFs are uploading correctly, or if it is just some rubbish going on generally in Kimble.  Kimble expense preview doesn't seem to work like it once did.
4. You will have to allow the scripts to have automation control over your computer.  If you don't do this, MacOS will prompt you to go into System preferences >> Security & Privacy >> Privacy and select the left navigation options for "Automation" and also "Accessibility".  This changes from time-to-time is MacOS, but you get the idea.  It's a security thing to either allow scripts to control your computer, or not.  For this reason, the first few times you run the scripts, make sure you just use somne test data - if the system has to ask you for automation/accessibility control, the scripts will usually fail.


