| Symbol | Meaning        |
|--------|----------------|
| `E`    | Exists         |
| `!E`   | Not exists     |
| `<-`   | Require        |
| `->`   | Override       |
| `+>`   | Inherit        |


## IAM

### Credentials

```
                            read ------------------┐                      ┌- identities.attemptAssume
                                                   |                      ├- origCreds.guessCredentialsFromHash
weakCredentials----------┐  write -----------------┤                      ├- credentials.use (Use other creds like in MFA)                                                            
guessCredentialsFromHash ┴> guessCredentials ------┼--> attemptUse -> use ├- attemptPropagateOneCredentialCompromised -> propagateOneCredentialCompromised -> (1) 
                            useLeakedCredentials --┤                      ├- encryptedData.accessDecryptedData
  attemptCredentialTheft -> credentialTheft--------┤                      ├- signedData.accessSpoofedData
 attemptCredentialsReuse -> credentialsReuse ------┘                      └- hashes.attemptUse

(1) credentials*.identities.users.oneCredentialCompromised (Some kind of propagation of credential reuse)
```
`weakCredentials` is caused by having no Identity associated.

`guessCredentialsFromHash` can be done if there is another Credentials object representing hashed creadentials associated. Other wise, it is directly gueassable. 

```
deny -> identities.lockoutFromCredentials
```
### Identity

If there are no User(s) associated with the identity, it causes the associated credentials with the identity to be weak.

**Lockout**

```     (IAMObject)
        deny ------------------┐  (IAMObject)       (IAMObject)          ┌- execPrivApps.denyFromLockout
        lockoutFromCredentials ┼> attemptLockout -> successfulLockout -> ├- readPrivData.denyFromLockout
                               |                                         └- subprivileges.lockout
           (Group)             |
lockout -> lockoutFromMembers -┘
```

**User Activity**

```
                                                                                   ┌- execPrivApps.attemptUnsafeUserActivity
attemptUnsafeUserActivity -> successfulUnsafeUserActivity -> unsafeUserActivity -> ├- highPrivApps.attemptUnsafeUserActivity
                                                                                   ├- lowPrivApps.attemptUnsafeUserActivity
                                                                                   └- subprivileges.attemptUnsafeUserActivity
```

**Assume**

```          (Propagate to parent identity)
          ┌- parentId.attemptAssume
assume -> |
          └- subprivileges.attemptUnsafeUserActivity
             (Propagate to privleges)
```
### Privlege

Privileges are basically identities without being able to do lockout.

### Group

Groups are basically identities without being able to do user activity.

## DataResources

### Data

**Read/Write/Delete**

All `read`,`write`,`delete` and `deny` lead to `information.read/write/delete/deny`
```
authorizedAccessFromApplication
│
├─> authorizedReadFromApplication -> authorizedRead -> attemptRead -> successfulRead -> read
│
├─> authorizedWriteFromApplication -> authorizedWrite -> attemptWrite -> successfulWrite -> write
│
└─> authorizedDeleteFromApplication -> authorizedDelete -> attemptDelete -> successfulDelete -> delete -> attemptDeny -> successfulDeny -> deny

authorizedReadFromIAM -> authorizedRead -> attemptRead -> successfulRead -> read
authorizedWriteFromIAM -> authorizedWrite -> attemptWrite -> successfulWrite -> write
authorizedDeleteFromIAM -> authorizedDelete -> attemptDelete -> successfulDelete -> delete
```

**Respond Connect**

Reverse send something to the attacker? Like reverse shell.

```
authorizedApplicationRespondConnectFromApplication -> authorizedApplicationRespondConnect -> attemptApplicationRespondConnect -> applicationRespondConnect -> applicationRespondConnect
authorizedApplicationRespondConnectFromIAM -> authorizedApplicationRespondConnect -> attemptApplicationRespondConnect -> applicationRespondConnect -> applicationRespondConnect
```

**Encryption**

You need the encryptCreds Credentials to do this:

```
accessUnencryptedData -> accessDecryptedData -> respondConnect,read,write,delete
```
This is used to evade WAF and and other scanners by encrypting the data:
```
bypassPayloadInspectionFromEncryptedData -> [active connection assets].payloadInspectionBypassed
```
**Signing**

You need the sigingCreds Credentials to do this:
```
accessUnsignedData -> accessSpoofedData -> respondConnect,write
```

**Reverse Reach**

This is for seeing if the data can be sent back to the attacker
```
attemptReverseReach -> reverseReach -> extract
```

**Information**

The actual `read`,`write`,`delete`,`extract` and `deny`.

```
write -> delete -> deny
```

May contain another data instance. But not in coreAttackerLang.

## Vulnerability

```
attemptAbuse -> abuse -> attemptExploit -> exploit -> impact
```

### Software Vulnerability

**Abuse**

```
                                           ┌- softwareProduct.softApplications.softwareProductVulnerabilityUserInteractionAchieved
attemptAbuse -> inherentUserInteraction ---┤
                                           └- userInteractionAchieved ┐ (# userInteractionRequired)
                    (# networkAccessRequired) networkAccessAchieved --┤
                    (# localAccessRequired)   localAccessAchieved ----┼-> abuse
                   (# physicalAccessRequired) physicalAccessAchieved -┤
                    (# lowPrivilegeRequired)  lowPrivilegesAchieved --┤
                   (# highPrivilegeRequired)  highPrivilegesAchieved -┘
```

**Exploit**

```
                  (# highComplexityExploitRequired)
                ┌ exploitTrivially --┐
attemptExploit -┤                    ├-exploit
                └ exploitWithEffort -┘
```

**Impact**

```
          ┌- read -> application.read, softwareProduct.readApplications       (# confidentialityImpactLimitations)
impact -> ├- modify -> application.modify, softwareProduct.modifyApplications (# integrityImpactLimitations)
          └- deny -> application.deny, softwareProduct.denyApplications       (# availabilityImpactLimitations)
```

## Compute Resources

### Application

**Conncect & Authenticate**

High priv variants?

```
networkConnectInspected ┬> networkConnect -> networkAccess -> fullAccess
                        └> specificAccessNetworkConnect -> specificAccessFromNetworkConnetion -> specificAccess
```

This ▼ can be reached through `reverseReach`. `networkConnectFromResponse` seems to be reverse conntect. For example reverse shell?

```
                                                                                             ┌> attemptUseVulnerability
attemptNetworkConnectFromResponse -> networkConnectFromResponse -> networkConnectUninspected ┼> allVulnerabilities().networkAccessAchieved
                                                                                             ├> networkConnect -> networkAccess -> fullAccess
                                                                                             └> specificAccessNetworkConnect -> specificAccessFromNetworkConnetion
                                                                                                                                    |
                                                                                                                                    ▼               
                                                                                                                                    specificAccess
                                                                                             
```

```
             ┌>allVulnerabilities().highPrivilegesAchieved
             |        
             |               ┌> localAccess -> fullAccess
authenticate ┼> localConnect ┼> specificAccessFromLocalConnection -> specificAccess
             |               ├> allVulnerabilities().localAccessAchieved
             |               └> attemptUseVulnerability
             └> networkAccess -> fullAccess
```

Low priv variants?

```

                           ┌> specificAccessFromLocalConnection -> specificAccess
specificAccessAuthenticate ┼> allVulnerabilities().lowPrivilegesAchieved
                           └> specificAccessFromNetworkConnection -> specificAccess
```

**Vulnerability**
```
attemptUseVulnerability ┬> successfulUseVulnerability -> useVulnerability -> allVulnerabilities().attemptAbuse
                        └> protectorIDPSs.bypassEffectiveness
```


**Full Access**
```
            ┌> executionPrivIAMs.attemptAssume,
            ├> attemptApplicationRespondConnectThroughData,
            ├> hostApp.localConnect
            ├> managedRoutingFw.attemptModify,
            ├> hostHardware.attemptSpreadWormThroughRemovableMedia,
            ├> hostHardware.attemptUseVulnerabilityFromSoftwareFullAccess
            ├> accessNetworkAndConnections -┬> outgoingApplicationConnections().attemptConnectToApplicationsUninspected
            |                               ├> outgoingApplicationConnections().attemptConnectToApplicationsInspected
            |                               ├> outgoingApplicationConnections().attemptAccessNetworksUninspected
            |                               ├> outgoingApplicationConnections().attemptAccessNetworksInspected
            |                               ├> networks.accessUninspected
            |                               └> networks.accessInspected
            |
fullAccess -┼> specificAccess (Continues on next heading)
            |
            |                                         ┌> containedData.attemptRead
            ├> attemptRead -> successfulRead -> read -┼> appExecutedApps.attemptRead
            |                                         ├> sentData.attemptRead
            |                                         └> receivedData.attemptRead
            |
            |                                               ┌> fullAccess
            ├> attemptModify -> successfulModify -> modify -┼> containedData.attemptWrite
            |                                               ├> appExecutedApps.attemptModify
            |                                               └> sentData.attemptWrite
            |
            |                                         ┌> containedData.attemptDeny
            └> attemptDeny -> successfulDeny -> deny -┼> appExecutedApps.attemptDeny
                                                      └> sentData.attemptDeny
```



**Specific Access**

```
                                                ┌> sentData.authorizedReadFromApplication
                ┌> appExecutedApps.localConnect ├> receivedData.authorizedReadFromApplication
                ├> specificAccessRead ----------┴> containedData.authorizedReadFromApplication
                ├> specificAccessModify --------┬> containedData.authorizedWriteFromApplication
                |                               └> sentData.authorizedWriteFromApplication
                ├> specificAccessDelete --------┬> containedData.authorizedDeleteFromApplication
                |                               └> sentData.authorizedDeleteFromApplication
specificAccess -┼> bypassContainerization -------> hostApp.attemptUseVulnerability
                ├> attemptUseVulnerability ┬> successfulUseVulnerability -> useVulnerability -┬> allVulnerabilities().attemptAbuse
                |                          └> protectorIDPSs.bypassEffectiveness              ├> readFromSoftProdVulnerability <---------------------┐
                |                                                                             ├> modifyFromSoftProdVulnerability <-------------------┤
                |                                                                             ├> denyFromSoftProdVulnerability <---------------------┤
                |                                                                             └> attemptSoftwareProductAbuse -> softwareProductAbuse ┘
                ├> attemptAuthorizedApplicationRespondConnectThroughData -> successfulAuthorizedApplicationRespondConnectThroughData -> authorizedApplicationRespondConnectThroughData -> (1)
                └> accessNetworkAndConnections ┬> networks.accessUninspected
                                               ├> networks.accessInspected
                                               ├> outgoingApplicationConnections().attemptConnectToApplicationsUninspected
                                               ├> outgoingApplicationConnections().attemptConnectToApplicationsInspected
                                               ├> outgoingApplicationConnections().attemptAccessNetworksUninspected
                                               └> outgoingApplicationConnections().attemptAccessNetworksInspected

(1) receivedData.authorizedApplicationRespondConnectFromApplication

```

**Phising Stuff?**
```
                                                                                                                  ┌> allVulnerabilities().userInteractionAchieved
                                                                                                                  ├> networkConnectUninspected
                           ┌> successfulUnsafeUserActivity -> successfulUnsafeUserActivity -> unsafeUserActivity -┼> localConnect
attemptUnsafeUserActivity -|                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      |
                           |                                                                                      └--------------------------------------------┐
                           ├> attackerUnsafeUserActivityCapabilityWithReverseReach ---┬> attackerUnsafeUserActivityCapability -> successfulUnsafeUserActivity -┘
                           ├> attackerUnsafeUserActivityCapabilityWithoutReverseReach ┘
                           └> protectorIDPSs.bypassEffectiveness

```

**Supply Chain**
```
                                                                                                                    ┌- containedData.attemptRead
attemptReadFromSoftProdVulnerability -> readFromSoftProdVulnerability -----> attemptRead -> successfulRead -> read -┼- appExecutedApps.attemptRead
                                                                                                                    ├- sentData.attemptRead
                                                                                                                    └- receivedData.attemptRead

                                                                                                                          ┌- fullAccess
attemptModifyFromSoftProdVulnerability -> modifyFromSoftProdVulnerability -> attemptModify -> successfulModify -> modify -┼- containedData.attemptWrite
                                                                                                                          ├- appExecutedApps.attemptModify
                                                                                                                          └- sentData.attemptWrite

                                                                                                                    ┌- containedData.attemptDeny
attemptDenyFromSoftProdVulnerability -> denyFromSoftProdVulnerability -----> attemptDeny -> successfulDeny -> deny -┼- appExecutedApps.attemptDeny
                                                                                                                    └- sentData.attemptDeny
```

