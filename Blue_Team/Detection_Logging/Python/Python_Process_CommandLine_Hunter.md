# Write a Python script that will list running Windows processes or read a text file of command lines then extract PID, name and command line & flag suspicious patterns.
---

NOTE: 1
Take this row:
```
['23148', 'python.exe', 'python.exe main.py']
```
row[0] = '23148' (pid)
row[1] = 'python.exe' (name)
row[2] = 'python.exe main.py' (command line)
Lists count from 0, so the third item is index 2.

And why do we need index 2? Because the command line is what has the sketchy words. ```flags_for``` checks a command line, and that is ```row[2]```. ```row[1]``` is just powershell.exe. The dangerous part is the argument/operator after is like example: -enc downloadstring.
Now that lives in ```row[2]```.


---

Code 
```
import csv 
import io 
import subprocess 
import sys 

# What this program does in essence: IT looks at what is running and scream if the command looks a bit sketchy.
# Csv is just a spreadsheet saved as a textfile.
# sys reads the filename you type after the script; csv splits PowerShell’s table into columns; io turns that PowerShell text into something csv can read.


# Our first function is called flags_for which takes one command, checks it against a list of sketchy words, and returns whichever of those words showed up.

def flags_for(command_line):
    text = (command_line or "").lower()
    suspicious_words = [ # Yes some of them can cause false alerm like iex, exec, hidden!
        "-enc", "downloadstring", "bypass", "iex", "invoke-expression", 
        "noprofile", "windowstyle", "hidden", "encodedcommand", "base64", 
        "b64decode", "eval", "exec", "system.net.webclient"
    ]
    sus_word_list = []
    for sus_word in suspicious_words:
        if sus_word in text:
            sus_word_list.append(sus_word)
    return sus_word_list


def scan_file(filename):
    with open(filename) as file: # If you do with open it will close the file for you automatically. 
        for each_line in file:
            hits  = flags_for(each_line)
            if hits:
                print(f"Command: {each_line.strip()} | hits: {hits}")
 
def scan_processes(): # This will pull live data so it wll not get any parameter passed in it. We will use cmdlet = command-let
    # Get Common Information Model Instance. CIM is just Windows database of system information. Processes, disks, services and etc. Win32_Process is the table for running processes.
    command =  "Get-CimInstance Win32_Process | Select-Object ProcessId, Name, CommandLine|ConvertTo-Csv -NoTypeInformation"
    result = subprocess.run(
        ["powershell.exe", "-Command", command],
        capture_output=True, # grab the output instead of dumping to screen
        text=True, # give it back as a string, not raw bytes
        check=True # raise an error if PowerShell fails
    )
    # Now listen csv.reader wants a file, but we have string so io.StringIO wraps the string to look like a file. Convert the raw string output into an in-memory file stream
    make_it_csv_file = io.StringIO(result.stdout)

    # Read the stream using csv.reader | stream is just data you read piece by piece, like a file you read line by line.
    csv_reader =  csv.reader(make_it_csv_file)

    # Loop through the rows or convert them into a list
    csv_rows = list(csv_reader)

    # Print the parsed rows to verify
    # Also Please Please Read NOTE:1 because it exaplains why we will use row[2].
    for row in csv_rows:
        command_line = row[2]
        hits = flags_for(command_line)
        if hits:
            print(f"PID {row[0]} | {row[1]} | hits: {hits} |  {command_line.strip()}")


scan_processes()
```
