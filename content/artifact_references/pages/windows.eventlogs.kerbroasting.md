---
title: Windows.EventLogs.Kerbroasting
hidden: true
tags: [Client Artifact]
---

This Artifact will return all successful Kerberos TGS Ticket events for
Service Accounts (SPN attribute) implemented with weak encryption. These
tickets are vulnerable to brute force attack and this event is an indicator
of a Kerbroasting attack.

Typical attacker methodology is to firstly request accounts in the domain
with SPN attributes, then request an insecure TGS ticket for brute forcing.
This attack is particularly effective as any domain credentials can be used
to implement the attack and service accounts often have elevated privileges.
Kerbroasting can be used for privilege escalation or persistence by adding a
SPN attribute to an unexpected account.

Log Source: Windows Security Event Log (Domain Controllers).
Event ID: 4769
Status: 0x0 (Audit Success)
Ticket Encryption: 0x17 (RC4)
Service Name: NOT krbtgt or NOT a system account (account name ends in $)
TargetUserName: NOT a system account (*$@*)

Monitor and alert on unusual events with these conditions from an unexpected
IP.
Note: There are potential false positives so whitelist normal source IPs and
manage risk of insecure ticket generation.


```yaml
name: Windows.EventLogs.Kerbroasting
author: Matt Green - @mgreen27

description: |
  This Artifact will return all successful Kerberos TGS Ticket events for
  Service Accounts (SPN attribute) implemented with weak encryption. These
  tickets are vulnerable to brute force attack and this event is an indicator
  of a Kerbroasting attack.

  Typical attacker methodology is to firstly request accounts in the domain
  with SPN attributes, then request an insecure TGS ticket for brute forcing.
  This attack is particularly effective as any domain credentials can be used
  to implement the attack and service accounts often have elevated privileges.
  Kerbroasting can be used for privilege escalation or persistence by adding a
  SPN attribute to an unexpected account.

  Log Source: Windows Security Event Log (Domain Controllers).
  Event ID: 4769
  Status: 0x0 (Audit Success)
  Ticket Encryption: 0x17 (RC4)
  Service Name: NOT krbtgt or NOT a system account (account name ends in $)
  TargetUserName: NOT a system account (*$@*)

  Monitor and alert on unusual events with these conditions from an unexpected
  IP.
  Note: There are potential false positives so whitelist normal source IPs and
  manage risk of insecure ticket generation.

reference:
  - https://attack.mitre.org/techniques/T1208/
  - https://www.trustedsec.com/blog/art_of_kerberoast/

parameters:
  - name: EvtxGlob
    default: '%SystemRoot%\System32\winevt\logs\Security.evtx'
  - name: SearchVSS
    description: "Add VSS into query."
    type: bool

sources:
  - query: |
      -- expand provided glob into a list of paths on the file system (fs)
      LET fspaths <= SELECT FullPath
        FROM glob(globs=expand(path=EvtxGlob))

      -- function returning list of VSS paths corresponding to path
      LET vsspaths(path) = SELECT FullPath
        FROM Artifact.Windows.Search.VSS(SearchFilesGlob=path)

      -- function returning IOC hits
      LET evtxsearch(PathList) = SELECT * FROM foreach(
            row=PathList,
            query={
                SELECT
                    timestamp(epoch=int(int=System.TimeCreated.SystemTime)) AS EventTime,
                    System.EventID.Value as EventID,
                    System.Computer as Computer,
                    EventData.ServiceName as ServiceName,
                    EventData.ServiceSid as ServiceSid,
                    EventData.TargetUserName as TargetUserName,
                    format(format="0x%x", args=EventData.Status) as Status,
                    EventData.TargetDomainName as TargetDomainName,
                    format(format="0x%x", args=EventData.TicketEncryptionType) as TicketEncryptionType,
                    format(format="0x%x", args=EventData.TicketOptions) as TicketOptions,
                    EventData.TransmittedServices as TransmittedServices,
                    EventData.IpAddress as IpAddress,
                    EventData.IpPort as IpPort,
                    FullPath
                FROM parse_evtx(filename=FullPath)
                WHERE
                    System.EventID.Value = 4769
                    AND EventData.TicketEncryptionType = 23
                    AND EventData.Status = 0
                    AND NOT EventData.ServiceName =~ "krbtgt|\\$$"
                    AND NOT EventData.TargetUserName =~ "\\$@"
          })


      -- include VSS in calculation and deduplicate with GROUP BY by file
      LET include_vss = SELECT * FROM foreach(row=fspaths,
            query={
                SELECT *
                FROM evtxsearch(PathList={
                        SELECT FullPath FROM vsspaths(path=FullPath)
                    })
                GROUP BY EventRecordID,Channel
              })

      -- exclude VSS in EvtxHunt`
      LET exclude_vss = SELECT *
        FROM evtxsearch(PathList={SELECT FullPath FROM fspaths})

      -- return rows
      SELECT * FROM if(condition=SearchVSS,
        then=include_vss,
        else=exclude_vss)

```
