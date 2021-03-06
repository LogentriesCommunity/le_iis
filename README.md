# Agentless IIS and Nxlog to Logentries.

## Install Nxlog

[Download](http://nxlog.org/products/nxlog-community-edition/download) the latest version of nxlog. 

## Insert configuration 

Open the Nxlog configuration file located at: `C:\Program Files (x86)\nxlog\conf\nxlog.conf`

and paste in the following and adjusting to your specific account:

```
## See the nxlog reference manual about the configuration options.
## It should be installed locally and is also available
## online at http://nxlog.org/nxlog-docs/en/nxlog-reference-manual.html
 
## Please set the ROOT to the folder your nxlog was installed into,
## otherwise it will not start.
#define ROOT C:\\Program Files\\nxlog
#define ROOT_STRING C:\\Program Files\\nxlog
define ROOT C:\\Program Files (x86)\\nxlog
define ROOT_STRING C:\\Program Files (x86)\\nxlog
define CERTDIR %ROOT%\\cert
 
Moduledir %ROOT%\\modules
CacheDir %ROOT%\\data
Pidfile %ROOT%\\data\\nxlog.pid
SpoolDir %ROOT%\\data
LogFile %ROOT%\\data\\nxlog.log
 
# Include fileop while debugging, also enable in the output module below
<Extension fileop>
 Module xm_fileop
</Extension>
 
<Extension json>
 Module xm_json
</Extension>
 
<Extension syslog>
 Module xm_syslog
</Extension>
 
<Input internal>
 Module im_internal
 Exec $Message = to_json(); 
</Input>
 
# Windows Event Log
<Input eventlog>
# Uncomment im_msvistalog for Windows Vista/2008 and later
 Module im_msvistalog
 
#Uncomment im_mseventlog for Windows XP/2000/2003
#Module im_mseventlog
 
 Exec $Message = to_json();
</Input>

<Output out>
 Module om_tcp
 Host data.logentries.com
 Port 80
 
 Exec to_syslog_ietf(); $raw_event = replace($raw_event, 'NXLOG@14506', 'LOG-TOKEN-HERE@41506 tag="logentries"] [', 1);
 
#Use the following line for debugging (uncomment the fileop extension above as well)
#Exec file_write("C:\\Program Files (x86)\\nxlog\\data\\nxlog_output.log", $raw_event);
</Output>
 
<Route 1>
 Path internal, eventlog => out
</Route>

# Create the parse rule for IIS logs. You can copy these from the header of the IIS log file.
<Extension w3c>
    Module xm_csv
    Fields $date, $time, $s-ip, $csmethod, $csuristem, $csuriquery, $sport, $csusername, $cip, $csUserAgent, $csReferer, $scstatus, $scsubstatus, $scwin32status, $timetaken
    FieldTypes string, string, string, string, string, string, integer, string, string, string, string, integer, integer, integer, integer
    Delimiter ' '
    QuoteChar '"'
    EscapeControl FALSE
    UndefValue -
</Extension>
 
# Convert the IIS logs to JSON and use the original event time
<Input IIS_Site1>
    Module    im_file
    #IIS File path should be placed in here, can use * for wildcard.
    File    "C:\\inetpub\\logs\\LogFiles\\W3SVC1\\u_ex*"
    SavePos  TRUE
 
     Exec if $raw_event =~ /^#/ drop();   \
       else                               \
       {                                  \
            w3c->parse_csv();             \
            $SourceName = "IIS";          \
            $Message = to_json();         \
       }
</Input>
 
<Route IIS>
    Path IIS_Site1 => out
</Route>
```

Making sure to add the Logentries log token in where it says "LOG-TOKEN-HERE" in the to_syslog_ietf() located on line 53.

Also you may need to uncomment where to set the ROOT to the folder your nxlog was installed into otherwise it will not start, this is located on lines 5-11.

On Line 77 you may need to change the IIS File path is located depending on your setup, can use * for wildcard. The default place for access logs is `%SystemDrive%\inetpub\logs\LogFiles`

## Restart the Nxlog service

Open the services tool in the start menu. Search for nxlog in the services and then restart it

## View a page on your IIS server

View a webpage on your IIS server to generate a new log entry. This will only log new events that have occurred after the restart.
