# node-nssm-iis
These bat files will help setting up nodeJS servers with NSSM and IIS. 
Don't forget to
- Install IIS redirect module
- Install NSSM somewhere
- If you don't use Microsoft dnscmd, but external DNS, then just remove the call to dns-create.bat in go.bat
- Create a rootfolder for your nodeJS apps, e.g. d:\nodeApps; each of your apps should be in its own folder, with the foldername set to the domain, e.g. d:\nodeApps\subdomain.domain.com\ or d:\nodeApps\domain.com\
- Put the *.bat files bellow to the root folder, e.g. d:\nodeApps\go.bat, d:\nodeApps\nssm.bat, ... and set them up
- Call go.bat from cmd to install your app: go.bat domain.com install 3891
- Call go.bat from cmd to remove your app: go.bat domain.com remove

#go.bat
```bat
echo e.g. domain.com install/remove 3891 (port)

@echo off
set name=%1
set mode=%2
set url=%1
set port=%3
set path=%~dp0

cls
echo "%path%nssm.bat"
if %mode%==install (
	echo Installing application %name%
	call "%path%dns-create.bat" %name% %mode% %url%
) 
if %mode%==remove (
	echo Removing application %name%
)
if %mode%==stop (
	echo Stopping application %name%
	call "%path%nssm.bat" %name% remove %port%
	@echo on
	EXIT /B 0
)
call "%path%nssm.bat" %name% %mode% %port%
call "%path%serverfarm.bat" %name% %mode% %port%
call "%path%urlrewrite.bat" %name% %mode% %url%
@echo on
EXIT /B 0
```

#dns-create.bat - DNS via dnscmd.exe
Set ipv4, ipv6, hostUrl and hostName according to your needs.
```bat
@echo off
set domain=%1
set dnscmd=C:\Windows\system32\dnscmd.exe
set ipv4=11.11.11.11
set ipv6=1a11:111:11:1111:1aaa:111a:0:1
set hostUrl=machine.domain.com
set hostName=xyz

:: SPF
:: v=spf1 a mx ptr ip4:%ipv4% ip6:%ipv6% ~all
:: Use WMIC to retrieve date and time
FOR /F "skip=1 tokens=1-6" %%G IN ('%systemroot%\system32\wbem\WMIC Path Win32_LocalTime Get Day^,Hour^,Minute^,Month^,Second^,Year /Format:table') DO (
   IF "%%~L"=="" goto s_done
      Set _yyyy=%%L
      Set _mm=00%%J
      Set _dd=00%%G
      Set _hour=00%%H
      SET _minute=00%%I
      SET _second=00%%K
)
:s_done

:: Pad digits with leading zeros
      Set _mm=%_mm:~-2%
      Set _dd=%_dd:~-2%
      Set _hour=%_hour:~-2%
      Set _minute=%_minute:~-2%
      Set _second=%_second:~-2%

Set logtimestamp=%_yyyy%%_mm%%_dd%
goto make_dump

:make_dump

:: dnscmd <ServerName> /ZoneAdd <ZoneName> {/Primary|/DsPrimary|/Secondary|/Stub|/DsStub} [/file <FileName>] [/load] [/a <AdminEmail>] [/DP <FQDN>]

call "%dnscmd%" %hostName% /ZoneAdd %domain% /Primary /a hostmaster@%domain%
:: call "%dnscmd%" %hostName% /RecordAdd %domain% @ NS %hostUrl%
echo %logtimestamp%
call "%dnscmd%" %hostName% /RecordAdd %domain% @ SOA %hostUrl% hostmaster@%domain% %logtimestamp%01 10800 1800 604800 10800

echo Blank
call "%dnscmd%" %hostName% /RecordAdd %domain% @ A %ipv4%
echo WWW
call "%dnscmd%" %hostName% /RecordAdd %domain% www A %ipv4%
echo SMTP
call "%dnscmd%" %hostName% /RecordAdd %domain% smtp A %ipv4%
echo IMAP
call "%dnscmd%" %hostName% /RecordAdd %domain% imap A %ipv4%
echo POP
call "%dnscmd%" %hostName% /RecordAdd %domain% pop A %ipv4%
echo POP3
call "%dnscmd%" %hostName% /RecordAdd %domain% pop3 A %ipv4%
echo MAIL
call "%dnscmd%" %hostName% /RecordAdd %domain% mail A %ipv4%
call "%dnscmd%" %hostName% /RecordAdd %domain% @ /Aging /OpenAcl MX 10 mail.%domain%
@echo on
```

#nssm.bat - NSSM to keep nodeJS app running
Set base (the folder where your apps are running), nssm (where nssm is installed), appEntry.

```bat
:: Argument 1: Name of service
:: Argument 2: Mode, install or remove

@echo off
set name=%1
set mode=%2
set port=%3

set service=nssm.%name%
set base=d:\nodeApps
set nssm=d:\apps\nssm
set appEntry=server.js

if %mode%==install (
	echo Installing nssm %service% %base%\%name% to port %port%
    %nssm% install %service% node
	%nssm% set %service% AppParameters "%base%\%name%\%appEntry%"
	%nssm% set %service% AppDirectory "%base%\%name%"
	%nssm% set %service% AppStdout "%base%\%name%\nssm.log"
	%nssm% set %service% AppStderr "%base%\%name%\nssm.log"
	%nssm% set %service% AppEnvironmentExtra PORT=%port% NODE_ENV=production
	%nssm% start %service%
) 
if %mode%==remove (
	echo Removing nssm %service% %base%\%name%
	%nssm% stop %service%
    %nssm% remove %service% confirm
)
@echo on
```

#serverfarm.bat - IIS Serverfarms
```bat
:: Argument 1: Name of service
:: Argument 2: Mode, install or remove
:: Argument 3: Port

@echo off
set name=%1
set mode=%2
set port=%3
set appcmd=%windir%\System32\inetsrv\appcmd.exe

if %mode%==install (
	echo Installing serverfarm %name% at %port%
	%appcmd% set config -section:webFarms /+"[name='nssm.%name%',enabled='true']" /commit:apphost
	%appcmd% set config -section:webFarms /+"[name='nssm.%name%'].[address='localhost',enabled='true']" /commit:apphost
	%appcmd% set config -section:webfarms -[name='nssm.%name%'].[address='localhost'].applicationRequestRouting.httpPort:%port% /commit:apphost
) 
if %mode%==remove (
	echo Removing serverfarm %name%
	%appcmd% set config -section:webFarms /-"[name='nssm.%name%']" /commit:apphost
)
@echo on
```

#url-rewrite.bat - IIS url-rewrite
```bat
:: Argument 1: Name of service
:: Argument 2: Mode, install or remove
:: Argument 3: Url

@echo off
set name=%1
set mode=%2
set url=%3
set appcmd=%windir%\System32\inetsrv\appcmd.exe

if %mode%==install (
	echo Installing url-rewrite %name% at %url%
	%appcmd% set config -section:system.webServer/rewrite/globalrules /+"[name='nssm.%name%',patternSyntax='Wildcard',stopProcessing='true']" /commit:apphost
	%appcmd% set config -section:system.webServer/rewrite/globalrules -[name='nssm.%name%'].match.url:"*" /commit:apphost
	%appcmd% set config -section:system.webServer/rewrite/globalrules -[name='nssm.%name%'].action.type:"Rewrite" /commit:apphost
	%appcmd% set config -section:system.webServer/rewrite/globalrules -[name='nssm.%name%'].action.url:"http://nssm.%name%/{R:0}" /commit:apphost
	%appcmd% set config -section:system.webServer/rewrite/globalrules /+"[name='nssm.%name%'].conditions.[input='{HTTP_HOST}',pattern='%url%']" /commit:apphost
) 
if %mode%==remove (
	echo Removing url-rewrite %name%
	%appcmd% set config -section:system.webServer/rewrite/globalrules /-"[name='nssm.%name%']" /commit:apphost
)
@echo on
```

