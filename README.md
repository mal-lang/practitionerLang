# `practitionerLang`
_A MAL language using terminology familiar to cyber security practitioners._


The repo houses the `practitionerLang` MAL language[^1], which is a domain-specific language (DSL) for modeling cyber security scenarios. The language is designed to be an intuitive variant of `coreLang`[^3], intended to be to use concepts understandable to cyber security practitioners. The language was developed based on the `coreLang` MAL language, which was modified to for an attacker-focused perspective. The language is intended to be used with the `mal-toolbox` as well as the `mal-simulator` to model and simulate cyber security scenarios.

Links to related repositories:

- [`coreLang`](https://github.com/mal-lang/coreLang)
- [`mal-toolbox`](https://github.com/mal-lang/mal-toolbox) (Tools to compile and use MAL languages)
- [`mal-simulator`](https://github.com/mal-lang/mal-simulator) (Simulator for MAL languages)
- [`tree-sitter-mal`](https://github.com/mal-lang/tree-sitter-mal) (MAL Grammar definition)

[^1]: [Link to first MAL article](https://doi.org/10.1145/3230833.3232799)
[^2]: [Link to article which formally defines the MAL](https://doi.org/10.1016/j.cose.2023.103284)
[^3]: [Link to article about the coreLang](https://doi.org/10.1016/j.cose.2024.104057)

## Compilation

The language can be compiled with the python package `mal-toolbox`. Install the package and compile with the following command:

```bash
pip install mal-toolbox
python -m maltoolbox compile practitionerLangSpec/main.mal practitionerLang-1.0.0.mar
```

## Differences from `coreLang`

This is what gets removed from `coreLang` and the thinking behind it.

### Defense Steps

In `coreLang`, defense steps can be categorized into two types: _active defenses_ and _passive defenses_. _Passive defenses_ are handled like properties or attributes which get set at the start of a simulation. _Active defenses_, on the other hand, model actions that a defender can take during a simulation. In `practitionerLang`, we assume that there is no active defender agent, and therefore all defense steps related to _active defenses_ are removed.  

_Active defenses_ includes, for example:
* `notPresent` (Indicates e.g. shutting off a machine/application or deleting a credential.)
* `XdefenseBypassed` (Defense steps that are bypassable.)
* Specific defenses such as `eavesdropDefense` or `payloadInspection`.

### Concepts

The concept of reverse reach in `coreLang` is related to modelling of reverse reachability by the attacker in a network. Attack steps modelling reverse reachability implement logical checks that data which the attacker is able to reach, it can also send back to the attacker's host. In `practitionerLang`, we assume that the attacker always has reverse reachability in the network which the attack targets. This falls more in line with the user experience of a pentester or red teamer, who can send data back to their own host if they are able to read it.

`coreLang` uses the concept of inspected and uninspected connections to model the presence of network defenses. Inspected network connections make it more diffcult for the attacker to send data relating to the attack, such as exfiltrated data and payloads. In `practitionerLang`, we assume that all connections are uninspected since network defenses are not clearly observed in pentesting or a red team engagement.


### Asset-Specific changes


#### `Hardware`
* Physical access: We don't consider the attacker having physical access to a target system. Physical access as an attack vector is still modeled in a limited fashion via users connecting removable media. However, this models the user having physical access, not the attacker.
* Supply chain attack: We remove the concept of supply chain attacks as we consider them beyond the capabilities of the attackers we wish to model.

#### `Application`
* Supply chain attacks: We remove the concept of supply chain attacks as we consider them beyond the capabilities of the attackers we wish to model.

#### `PhysicalZone`

Removed in its entirety.

We simplify the concept of physical access by saying users can have access to hardware, instead of also considering hardware residing in a physical zone the user has access to.

#### `IDPS`

Removed in its entirety.

We don't model attackers with the capability to utilize attack vectors on Intrusion Detection and Prevention Systems (IDPS) themselves. 

#### DataResources

The following attack steps are removed since they relate to the concept of reverse reachability:
* `applicationRespondConnect`
* `reverseReach`

#### `Data`

Associations that are removed:

* Data in data: We simplify the asset `Data` by saying that no data can contain data.
* Replicas: We remove the concept of replicas in `Data` since it is not relevant to the attacker-focused perspective of `practitionerLang`.

#### Networking

Assocations that are removed:

* NetworkExposure: Marked as "DEPRECATED" in the user info, and can be modeled equivalently using a ConnectionRule.
* DiodeInNetworkConnection: Removed since it is not relevant to the attacker-focused perspective of `practitionerLang`.

#### `Network`

_Active defense steps_ that are removed:

* `networkAccessControl`
* `eavesdropDefense`
* `adversaryInTheMiddleDefense`

#### `ConnectionRule`

_Active defense steps_ that are removed:

* `restricted`
* `payloadInspection`

#### `Vulnerability`

The attack steps `exploitTrivially` and `exploitWithEffort` (and related defense steps or properties) are removed. They are instead simplified to a single attack step `exploit` where the effort required to exploit a vulnerability is instead modelled by setting the TTC of the exploit step. This is a setting that can be done on the instance model level, and is not a property of the asset class.

#### `IAM`

Removed:

* Group and Privilege assets: Privleges are not intended to be used normally. Groups provide litle additional modelling benefit for an extra, similar asset as `Identity`. We choose to simplify the language by removing these assets.
* `credentialsReuse` and associated logic and attack steps is removed. This is instead modelled via a new `Credentials <-> Credentials` association.

#### `User`

Removed _passive defense steps_ (properties):

* `noPasswordReuse` (Can be modeled by an association between credentials.)
* `noRemovableMediaUsage` 
* `securityAwareness`

The purpose of these _passive defense steps_ (properties) can instead be modelled by setting high/infinite TTCs on the relevant attack steps.

### Attack Steps Renamed from `coreLang` to `practitionerLang`

The asset `Application` has these attack steps renamed:

* `specificAccess` -> `userAccess`
* `fullAccess` -> `rootAccess`
* `networkConnectFromResponse` -> `reverseNetworkConnection`
* `userAccess[Read|Modify|Delete|Authenticate]` -> `[read|modify|delete|authenticate]AsUser`

The asset `User` has these attack steps renamed:
* `credentialTheft` -> `stealCredentials`

For IAM related assets, these attack steps are renamed:
* `credentialTheft` -> `stealCredential`§
