---
title: MacOS.System.Users
hidden: true
tags: [Client Artifact]
---

This artifact collects information about the local users on the
system. The information is stored in plist files.


```yaml
name: MacOS.System.Users
description: |
  This artifact collects information about the local users on the
  system. The information is stored in plist files.

parameters:
  - name: UserPlistGlob
    default: /private/var/db/dslocal/nodes/Default/users/*.plist
  - name: OnlyShowRealUsers
    type: bool
    default: Y

sources:
  - query: |
      LET user_plist = SELECT FullPath FROM glob(globs=UserPlistGlob)
      LET UserDetails =
              SELECT get(member="name.0", default="") AS Name,
                     get(member="realname.0", default="") AS RealName,
                     get(member="shell.0", default="") AS UserShell,
                     get(member="home.0", default="") AS HomeDir,
                     plist(file=get(member="LinkedIdentity.0", default=""),
                           accessor='data') as AppleId,
                     plist(file=get(member="accountPolicyData.0", default=""),
                           accessor='data') AS AccountPolicyData
              FROM plist(file=FullPath)

      SELECT Name, RealName, UserShell, HomeDir,
               get(item=AppleId, field="appleid.apple.com") AS AppleId,
               timestamp(epoch=AccountPolicyData.creationTime) AS CreationTime,
               AccountPolicyData.failedLoginCount AS FailedLoginCount,
               timestamp(epoch=AccountPolicyData.failedLoginTimestamp) AS FailedLoginTimestamp,
               timestamp(epoch=AccountPolicyData.passwordLastSetTime) AS PasswordLastSetTime
      FROM foreach(row=user_plist, query=UserDetails)
      WHERE NOT OnlyShowRealUsers OR NOT UserShell =~ 'false'

```
