# Google Drive - Programatic files deleting using JavaScript & Node.js.
### Delete large number of files and folders in Google Drive using a Service Account and an impersonate Email as user account. Tested deleting over two thousand files in Google Drive without exceeding the Google Work Space quota.  

***ATTENTION. NOT for production ready!. It is important to add code to fix possible errors and misbehaves!***

La descripción del código puede traducirla fácilmente usando la herramienta: https://translate.google.com

```JavaScript
const {google} = require('googleapis');
```
Using a json file provided by Google Workspace Service Account
```JavaScript
process.env.GOOGLE_APPLICATION_CREDENTIALS = './298b752exAw23.json'
const credentials = require('./298b752exAw23.json');
const scopes = ['https://www.googleapis.com/auth/drive', 'https://www.googleapis.com/auth/drive.readonly', 'https://www.googleapis.com/auth/drive.file'];
```
The user in your domain to perform the total files delete process.
```JavaScript
const impersonateEmail = "user@domain.com";
```
W A R N I N G ! The files deleted by this code won't be sent to the Trash, hence are unrecoverable!

Authenticating using the keys in the json file and the ***impersonateEmail*** above assigned.
```JavaScript
const divisa = new google.auth.JWT(
		credentials.client_email, null,
		credentials.private_key, scopes, impersonateEmail);
var totalFiles = 0; totalFolders = 0;   // Optional
```
IMPORTANT! To save Google APIs quota we will code to delete first the folders in the user's Google Drive. As we know, when a folder is deleted all the contains files AND folders inside will be deleted too.

We need to use an Asynchronous function: ***timerToDelete*** to get control over the time, in order to do NOT exceed the API quota limits.

***foldersIDsPool*** is the array to warehouse all the received data.
```JavaScript
async function clearUserFolders(divisa, pageToken = '', foldersIDsPool = []) {
```
***conectorList*** and ***conectorDel*** are the SAME authenticated request to Google Drive. We did used the two to reduce the API call OAuth quota when spawning each assignment.

Google APIs library needs this property must be named ***auth***
```JavaScript
  const conectorList = google.drive({ version: 'v3', auth: divisa });
	const conectorDel = google.drive({ version: 'v3', auth: divisa });
```
***Assignment 1/4.*** Get files(folders) list from Google Drive. Remember that we are looking for first deleting ONLY folders, hence use the ***q*** selector to get ONLY folders IDs.
```JavaScript
	const {data} = await conectorList.files.list({
	q: "mimeType = 'application/vnd.google-apps.folder'",
    "fields": "nextPageToken, files(id)",  // Asking for id's & nextPageToken only.
	pageToken
	});
```
In the very first request ***pageToken*** will be empty, that's Okay. It will get a value if the Array ***foldersIDsPool*** gains more than 100 IDs. Note: 100 is the API default value to deliver.
```JavaScript
	foldersIDsPool.push(data.files);
	const apiResIDs = data.files;  // Storing the body result which is in Json array format. *apiResIDs*-->Iterable.
```

This Function will be called later. Works to spawn a 500 milliseconds rate between each deleted file. By setting such time we have deleted without reach the quota limit almost 2000 files in a single running. Average time per running: 3 minutes. Why? It happens that each user uses many folders to organize their files, so by deleting only folders at first, we save quite amount of time and quota!
```JavaScript
   function timerToDelete(delaying) {
		return new Promise(resolve => {
			setTimeout(() => { resolve('') }, delaying);
		})
	}
```
***Assignment 2/4.*** The core function to delete ONLY folders using the native JS statement for-of. ***apiResIDs*** it's the Json Array Iterable Object that contains the List result in a key-value pair.
```JavaScript
	async function dumpThisIds() {   // Assignment 2/4.
		for (let idToDump of apiResIDs) {
				conectorDel.files.delete({  //  Request to delete one by one.
						"fileId": idToDump.id  // take up then send each .id string value.
					}, (err, response) => {
						if (err) throw err;
				});
			await timerToDelete(500);  // To avoid the 'quota limit exceeded' The magic goes here!
				console.log(idToDump.id);  //  Optional.
		}
	}
	totalFolders += Object.keys(apiResIDs).length; // Optional to check the amount of files.
```

//  The *await* dumpThisIds();  is very important to ensure that the next 100 files are requested only when a prior 100 chunk has been deleted. *nextPageToken* request is also deferred in order not to reach the quota!
	await dumpThisIds();   // Calling the Assignment 2/4. 

// *nextPageToken* received in the result body? If there is another page of ID's to be looked at - repeat again.
	if (data.nextPageToken) {
		await clearUserFolders(divisa, data.nextPageToken, foldersIDsPool);
	} else {
	    console.log("#   #   #   #   #   #   # TOTAL FOLDERS: ", totalFolders);  // Optional
		
// Once all folders was deleted, we need to wait for Google servers to propagues and updates the user's Google Drive state. Without this timeout chances are to get errors for the next step.
		async function timerToRefreshAPI(updating) {
			return new Promise(resolve => {
				setTimeout(() => { resolve('') }, updating);
		})}
		await timerToRefreshAPI(30000);  // 30 seconds it's Okay. You can test less...
	}
  return divisa;  //  Hand the credentials when calling the promise: *.then(clearUserFiles)* as you can see just below...
}

clearUserFolders(divisa).then(clearUserFiles).catch(Error); // Asynchronous events.


/*  *  *  *  *  *  *  *  *  *  *  *  *  *  *
Here the number 3 and 4 assignments performing the same logic but fetching for those true files that NOT were inside any folder that is, in the upper layer of Google Drive.
*  *  *  *  *  *  *  *  *  *  *  *  *  *  */
async function clearUserFiles(divisa, pageToken = '', filesIDsPool = []) {

// Google APIs library needs this variable or property must be named *auth*
	const conectorList = google.drive({ version: 'v3', auth: divisa });
	const conectorDel = google.drive({ version: 'v3', auth: divisa });

// Get really true files list from Google Drive. The query for folders has been removed.
	const {data} = await conectorList.files.list({  //  Assignment 3/4
	"fields": "nextPageToken, files(id)",
	pageToken
  });
    filesIDsPool.push(data.files);
	const apiResIDs = data.files;
			
	function timerToDelete(delaying) {
		return new Promise(resolve => {
			setTimeout(() => { resolve('') }, delaying);
		})
	}
	
	async function dumpThisIds() {		//  Assignment  4/4
		for (let idToDump of apiResIDs) {
				conectorDel.files.delete({ 
						"fileId": idToDump.id
					}, (err, response) => {
						if (err) throw err;
				});
			await timerToDelete(500);
				console.log(idToDump.id);  // Optional
		}
	}
	totalFiles += Object.keys(apiResIDs).length;  // Optional to check the amount of files.
	await dumpThisIds();
// If there is another page of ID's to be looked at - repeat again
	if (data.nextPageToken) {
		await clearUserFiles(divisa, data.nextPageToken, filesIDsPool);
	} else {
	    console.log("@  @  @  @  @  @  @  @ TOTAL FILES: ", totalFiles);  // Optional
	}
// Don't need to wait any more from Google servers. All done!	
}

//  P A R A L L E L    M E T H O D    C O M I N G   S O O N  ! ! !
//  _______________________  0  _________________________   //






### Cheeeeers for all!

