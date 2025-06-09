# Tibia Screenshot Sorter.workflow

A macOS Automator **Quick Action** that organises Tibia client screenshots by parsing metadata embedded in the filename. It extracts the **capture date**, **timestamp**, **character name**, and **event type**, then moves the file into a chronologically structured directory tree. All operations are logged per execution.

---

## Filename Format

Each screenshot must follow this strict naming pattern:

```
YYYY-MM-DD_<timestamp>_<CharacterName>_<EventType>.png
```

### Example:
```
2025-06-07_170210376_Night'Flyn_Hotkey.png
```

- `2025-06-07` → date of screenshot (YYYY-MM-DD)
- `170210376` → numeric timestamp (for uniqueness)
- `Night'Flyn` → character name
- `Hotkey` → event type (e.g. DeathPvE, LevelUp, ValuableLoot)

---

## Directory Mapping Logic

Given the input file:
```
2025-06-07_170210376_Night'Flyn_Hotkey.png
```

The output path will be:
```
<ParentFolder>/
    └── Night'Flyn/
        └── Hotkey/
            └── 2025/
                └── 06/
                    └── 07/
                        └── 2025-06-07_170210376_Night'Flyn_Hotkey.png
```

Additionally, a metadata log file will be created:
```
2025-06-07_HHMMSS_Metadata_Log.txt
```

---

## Installation

1. Move `Tibia Screenshot Sorter.workflow` into:
   ```
   ~/Library/Services/
   ```

2. It will appear in Finder under:
   ```
   Right-click → Quick Actions → Tibia Screenshot Sorter
   ```

---

## Usage

1. Select one or more screenshot files in Finder.
2. Right-click → Quick Actions → **Tibia Screenshot Sorter**.
3. The workflow will:
   - Parse and validate filenames.
   - Create the required folder structure.
   - Move each file accordingly.
   - Write a structured log file in the original directory.
   - Notify on success or failure via macOS Notification Centre.

---

## Log File

Each execution creates a structured `.txt` log detailing:
- Filename
- Date components
- Character name
- Event type
- Destination path
- Errors (if any)

Log format:
```
2025-06-07_134500_Metadata_Log.txt
```

---

## AppleScript Source

```applescript
-- Define the main handler
on run {input, parameters}
	-- Initialize metadata log
	set metadataLog to ""
	
	-- Generate a timestamp for the log file (YYYY-MM-DD_HHMMSS)
	set logTimestamp to getCurrentTimestamp()
	
	-- Process each file in the input
	repeat with aFile in input
		try
			-- Convert the file to a POSIX path
			set posixFilePath to POSIX path of aFile
			
			-- Get the parent directory using shell script
			set posixParentFolder to do shell script "dirname " & quoted form of posixFilePath
			
			-- Convert POSIX parent path back to AppleScript format
			set parentFolder to POSIX file posixParentFolder as text
			
			-- Get the filename without extension
			tell application "System Events"
				set fileName to name of aFile
				set fileExtension to name extension of aFile
				set baseName to text 1 thru ((length of fileName) - (length of fileExtension) - 1) of fileName
			end tell
			
			-- Log the baseName for debugging
			log "Processing file: " & baseName
			
			-- Validate filename format
			if baseName does not start with "20" then
				set metadataLog to metadataLog & "Skipped: " & fileName & " (Invalid filename format)" & return & return
				exit repeat
			end if
			
			-- Extract metadata
			set {fileDate, fileCharacter, fileEvent} to extractMetadata(baseName)
			
			-- Separate date components
			set dateYear to text 1 thru 4 of fileDate
			set dateMonth to text 6 thru 7 of fileDate
			set dateDay to text 9 thru 10 of fileDate
			
			-- Ensure the required folder structure exists within the parent directory
			set characterFolder to parentFolder & fileCharacter
			set eventFolder to characterFolder & ":" & fileEvent
			set yearFolder to eventFolder & ":" & dateYear
			set monthFolder to yearFolder & ":" & dateMonth
			set dayFolder to monthFolder & ":" & dateDay -- NEW: Add day subfolder
			
			-- Create the necessary folders
			createFolderIfNotExists(characterFolder)
			createFolderIfNotExists(eventFolder)
			createFolderIfNotExists(yearFolder)
			createFolderIfNotExists(monthFolder)
			createFolderIfNotExists(dayFolder) -- NEW: Create day folder
			
			-- Define the destination path for the file
			set destinationPath to dayFolder & ":" & fileName
			
			-- Move the file to the correct folder
			tell application "Finder"
				move aFile to folder dayFolder with replacing
			end tell
			
			-- Log extracted metadata
			set metadataLog to metadataLog & "File: " & fileName & return & ¬
				"Date Year: " & dateYear & return & ¬
				"Date Month: " & dateMonth & return & ¬
				"Date Day: " & dateDay & return & ¬
				"Character: " & fileCharacter & return & ¬
				"Event: " & fileEvent & return & ¬
				"Moved to: " & destinationPath & return & return
			
		on error errMsg
			-- Log errors
			set metadataLog to metadataLog & "Error: " & fileName & " - " & errMsg & return & return
		end try
	end repeat
	
	-- Write metadata to a timestamped log file in the same folder as the first input file
	if (count of input) > 0 then
		writeToTextFile(metadataLog, parentFolder, logTimestamp)
	end if
	
	return input
end run

-- Ensure that the given folder path exists, creating it if necessary
on createFolderIfNotExists(folderPath)
	-- Convert AppleScript path to POSIX path for shell compatibility
	set posixPath to POSIX path of folderPath
	do shell script "mkdir -p " & quoted form of posixPath
end createFolderIfNotExists

-- Extract metadata from the filename
on extractMetadata(baseName)
	-- Extract date (YYYY-MM-DD)
	set fileDate to text 1 thru 10 of baseName
	
	-- Extract all underscores
	set lastUnderscore to my lastIndexOf("_", baseName)
	if lastUnderscore = -1 then error "Invalid filename format: No underscore found for event."
	
	-- Detect if the filename ends with a number
	set lastSegment to text (lastUnderscore + 1) thru -1 of baseName
	set isLastSegmentNumber to my isNumeric(lastSegment)
	
	-- Determine the correct event position
	if isLastSegmentNumber then
		-- If the filename ends with a number, use the second-to-last underscore for the event
		set eventUnderscore to my lastIndexOf("_", text 1 thru (lastUnderscore - 1) of baseName)
		set fileEvent to text (eventUnderscore + 1) thru (lastUnderscore - 1) of baseName
	else
		-- Otherwise, use the last underscore
		set fileEvent to text (lastUnderscore + 1) thru -1 of baseName
		set eventUnderscore to lastUnderscore
	end if
	
	-- Extract character name (between the second underscore and event underscore)
	set firstUnderscore to my indexOf("_", baseName, 1)
	if firstUnderscore = -1 then error "Invalid filename format: No underscore found for character name."
	
	set secondUnderscore to my indexOf("_", baseName, firstUnderscore + 1)
	if secondUnderscore = -1 then error "Invalid filename format: No second underscore found for character name."
	
	set fileCharacter to text (secondUnderscore + 1) thru (eventUnderscore - 1) of baseName
	
	return {fileDate, fileCharacter, fileEvent}
end extractMetadata

-- Write metadata to a timestamped text file in the same folder as the files
on writeToTextFile(metadataLog, destinationFolder, timestamp)
	-- Define the file path with the timestamp
	set filePath to destinationFolder & timestamp & "_Metadata_Log.txt"
	
	-- Write the metadata to the file
	try
		set fileRef to open for access file filePath with write permission
		write metadataLog to fileRef starting at eof
		close access fileRef
		display notification "Metadata saved to " & filePath with title "Metadata Extractor"
	on error errMsg
		display notification "Error writing to file: " & errMsg with title "Metadata Extractor"
	end try
end writeToTextFile

-- Generate a timestamp in the format YYYY-MM-DD_HHMMSS
on getCurrentTimestamp()
	set currentDate to current date
	set yearText to year of currentDate as string
	set monthText to text -2 thru -1 of ("0" & (month of currentDate as integer)) -- Ensure two digits
	set dayText to text -2 thru -1 of ("0" & (day of currentDate))
	set hoursText to text -2 thru -1 of ("0" & (hours of currentDate))
	set minutesText to text -2 thru -1 of ("0" & (minutes of currentDate))
	set secondsText to text -2 thru -1 of ("0" & (seconds of currentDate))
	
	return yearText & "-" & monthText & "-" & dayText & "_" & hoursText & minutesText & secondsText
end getCurrentTimestamp

-- Helper function to find the index of a character in a string
on indexOf(searchString, inputString, startIndex)
	set inputLength to length of inputString
	repeat with i from startIndex to inputLength
		if text i thru (i + (length of searchString) - 1) of inputString is searchString then
			return i
		end if
	end repeat
	return -1
end indexOf

-- Helper function to find the last index of a character in a string
on lastIndexOf(searchString, inputString)
	set inputLength to length of inputString
	repeat with i from inputLength to 1 by -1
		if text i thru (i + (length of searchString) - 1) of inputString is searchString then
			return i
		end if
	end repeat
	return -1
end lastIndexOf

-- Helper function to check if a string is numeric
on isNumeric(inputString)
	try
		inputString as number
		return true
	on error
		return false
	end try
end isNumeric
```

---

## Constraints

- Only files with valid `YYYY-MM-DD_<timestamp>_<char>_<event>.png` names are processed
- Files are **moved**, not copied
- Existing files in the destination path are overwritten

---

## Licence

© 2025  
Free for personal use. No warranty. Use at your own risk.
