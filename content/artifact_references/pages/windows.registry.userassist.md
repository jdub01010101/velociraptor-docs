---
title: Windows.Registry.UserAssist
hidden: true
tags: [Client Artifact]
---

Windows systems maintain a set of keys in the registry database
(UserAssist keys) to keep track of programs that executed. The
number of executions and last execution date and time are available
in these keys.

The information within the binary UserAssist values contains only
statistical data on the applications launched by the user via
Windows Explorer. Programs launched via the command­line (cmd.exe)
do not appear in these registry keys.

From a forensics perspective, being able to decode this information
can be very useful.

Limitations: Additional data not parsed by Velociraptor is the FocusTime
and FocusCount however these are not reliable.
Also please note that some methods of viewing an executable will update
the associated UserAssist key, and some methods of accessing an executable
will not update the execution counter or time. Therefore there may be
some executions that have a 0 time and 0 runcount.


```yaml
name: Windows.Registry.UserAssist
description: |
  Windows systems maintain a set of keys in the registry database
  (UserAssist keys) to keep track of programs that executed. The
  number of executions and last execution date and time are available
  in these keys.

  The information within the binary UserAssist values contains only
  statistical data on the applications launched by the user via
  Windows Explorer. Programs launched via the command­line (cmd.exe)
  do not appear in these registry keys.

  From a forensics perspective, being able to decode this information
  can be very useful.

  Limitations: Additional data not parsed by Velociraptor is the FocusTime
  and FocusCount however these are not reliable.
  Also please note that some methods of viewing an executable will update
  the associated UserAssist key, and some methods of accessing an executable
  will not update the execution counter or time. Therefore there may be
  some executions that have a 0 time and 0 runcount.

reference:
  - https://www.aldeid.com/wiki/Windows-userassist-keys

precondition: SELECT OS From info() where OS = 'windows'

parameters:
  - name: UserFilter
    default: ""
    description: If specified we filter by this username.
    type: regex

  - name: ExecutionTimeAfter
    default: ""
    type: timestamp
    description: If specified only show executions after this time.

  - name: UserAssistKey
    default: Software\Microsoft\Windows\CurrentVersion\Explorer\UserAssist\*\Count\*

export:
  LET userAssistProfile = '''
      [
        ["Header", 0, [
          ["NumberOfExecutions", 4, "uint32"],
          ["LastExecution", 60, "uint64"]
        ]]
      ]
    '''

sources:
  - query: |
      LET TMP = SELECT OSPath.Path AS _KeyPath,
          parse_string_with_regex(
                string=OSPath.Path,
                regex="^.+Count\\\\\"?(?P<Name>.+?)\"?$") AS Name,
            OSPath,
            parse_binary(
               filename=Data.value,
               accessor="data",
               profile=userAssistProfile,
               struct="Header"
             ) As ParsedUserAssist,
             Username AS User
      FROM Artifact.Windows.Registry.NTUser(KeyGlob=UserAssistKey)

      LET UserAssist = SELECT _KeyPath,
          if(condition=Name.Name,
             then=rot13(string=Name.Name),
             else=OSPath.Path) AS Name,
          User,
          timestamp(winfiletime=ParsedUserAssist.LastExecution) As LastExecution,
          timestamp(winfiletime=ParsedUserAssist.LastExecution).Unix AS LastExecutionTS,
          ParsedUserAssist.NumberOfExecutions AS NumberOfExecutions
        FROM TMP
        ORDER BY LastExecution
      LET A1 = SELECT * FROM if(
          condition=UserFilter,
          then={
            SELECT * FROM UserAssist WHERE User =~ UserFilter
          },
          else={ SELECT * FROM UserAssist})

      SELECT * FROM if(
          condition=ExecutionTimeAfter,
          then={
            SELECT * FROM A1 WHERE LastExecutionTS > ExecutionTimeAfter
          },
          else={ SELECT * FROM A1})

```
